# Connector Sovity — ODEON

A **Barbara IoT** connector application, developed as part of the needs of the **ODEON** project.

Its purpose is to connect Barbara edge nodes to a federated data space, so that information generated on the plant floor can be published, discovered, negotiated and exchanged with third parties in a sovereign and governed way — without exposing the OT systems directly and without depending on external exchange infrastructure. It turns a Barbara edge node into a complete data space connector based on Eclipse Dataspace Components (EDC), ready to take part in IDS/Gaia-X style ecosystems with policy-based access control and participant identity.

| | |
| --- | --- |
| Developer | Barbara IoT |
| Project | ODEON |
| Version | `16.6.2-b.1` |
| Upstream | Sovity EDC Community Edition `v16.6.2` |
| Validated and published | Barbara Marketplace, 20 July 2026 |

The connector defaults to the participant identity `odeon` / `odeon-control`.

## Overview

The **Sovity Connector** is a production-ready distribution of the Eclipse Dataspace Components (EDC) that lets organizations publish, discover, negotiate, and securely exchange data within data spaces. It provides Management and Protocol APIs, policy-based access control, participant identity handling, and interoperability with common data space profiles (e.g., IDS/Gaia-X–aligned ecosystems). In short: it’s the runtime that connects your systems to a federated data space with governance built in.

This app packages **Sovity EDC Community Edition v16.6.2** as three containers deployed together with Docker Compose, and is configured entirely through environment variables held in a single `.env` file at the root of the repository.

## Services

| Service         | Base image                        | Purpose                                                                                    | Exposed ports             |
| --------------- | --------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------- |
| `postgresql`    | `docker.io/postgres:17.10-alpine3.24` | Bundled PostgreSQL database used by the connector for persistence. Internal only.      | none (internal `5432`)    |
| `sovity-edc`    | `ghcr.io/sovity/edc-ce:16.6.2`    | EDC connector with integrated control and data plane. Serves the Management and DSP APIs.  | 11002 (mgmt), 11003 (DSP) |
| `sovity-edc-ui` | `ghcr.io/sovity/edc-ce-ui:16.6.2` | Next.js management web UI that talks to the connector's Management API.                     | 8080 (UI)                 |

Startup is ordered by health: `sovity-edc` waits until `postgresql` is healthy, and `sovity-edc-ui` waits until `sovity-edc` is healthy. Once running, the management web UI is available at `http://<host-ip>:<SOVITY_UI_API_PORT>` (default port `8080`).

## Getting Started

Requirements: Docker Engine with the Compose plugin.

Review the settings in `.env` (see [Environment Variables](#environment-variables)) — at minimum change `EDC_API_AUTH_KEY` and `NEXT_PUBLIC_MANAGEMENT_API_KEY` away from the `demo-api-key` default before exposing the connector — then start the stack:

```bash
docker compose -f docker-compose_base.yml up -d --build
```

The three containers come up in order and the command returns once they report healthy. Check them and reach the APIs:

```bash
docker compose -f docker-compose_base.yml ps
```

| What | Where |
| ---- | ----- |
| Management web UI | `http://localhost:8080` |
| Management API | `http://localhost:11002/api/management` (send `x-api-key: $EDC_API_AUTH_KEY`) |
| DSP protocol API | `http://localhost:11003/api/v1/dsp` |

To stop the stack while keeping the database, or to remove it together with its data:

```bash
docker compose -f docker-compose_base.yml down
docker compose -f docker-compose_base.yml down -v
```

## Configuration

All operator-configurable settings live in `.env`, which Compose uses for two things at once: it interpolates `SOVITY_MANAGEMENT_API_PORT`, `SOVITY_DATASPACE_API_PORT` and `SOVITY_UI_API_PORT` into the published `ports:` mappings, and it injects the whole file into the `sovity-edc` and `sovity-edc-ui` containers via `env_file`. Changing a value requires recreating the containers (`docker compose -f docker-compose_base.yml up -d`).

Note that `SOVITY_EDC_FQDN_PUBLIC`, `NEXT_PUBLIC_MANAGEMENT_API_URL`, `NEXT_PUBLIC_PROTOCOL_API_URL` and `NEXT_PUBLIC_PRECONFIGURED_COUNTERPARTIES` all default to `localhost`, which only works when the browser runs on the same machine as the connector. Point them at the host's reachable address or hostname before other participants can connect.

## Persistence

The connector's state is persisted in the bundled PostgreSQL, whose data cluster is stored in the named Docker volume `VOLUME_SOVITY_DB_DATA` (mounted at `/var/lib/postgresql/data`). The volume survives container recreation.

The database credentials (`POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` on the `postgresql` service and the matching `SOVITY_JDBC_URL`/`SOVITY_JDBC_USER`/`SOVITY_JDBC_PASSWORD` on `sovity-edc`) are internal wiring: they are embedded directly in `docker-compose_base.yml`, kept consistent by construction, and are **not** exposed as operator-configurable settings. The database is reachable only on the `connectorServices` network (no host port is published).

## Key Features

-   **Integrated Control & Data Plane** - Deploy a single connector that combines both control plane and data plane for simplified setup and management.
-   **Management Web UI** - Includes a modern Next.js-based frontend for managing assets, contracts, and policies without needing to call APIs manually.
-   **Flexible Deployment Profiles** - Supports multiple deployment modes (integrated, standalone control, standalone data plane) to adapt to various architectures.
-   **Mock IAM for Local Testing** - Offers `sovity-mock-iam` dataspace mode to simulate identity and access management flows without needing external IAM systems.
-   **Cross-Origin Support (CORS)** - Configurable CORS headers allow browser-based UIs and external applications to safely call the connector APIs.

## Technical Details

-   **Versioned Release** - Built from the upstream `sovity/edc-ce` and `sovity/edc-ce-ui` `16.6.2` images, packaged with support for x86_64 and ARM64 architectures.
-   **Runtime Configuration** - Controlled via environment variables such as `SOVITY_DEPLOYMENT_KIND`, `SOVITY_DATASPACE_KIND`, and `EDC_API_AUTH_KEY`.
-   **Database Integration** - Persists its state in the bundled PostgreSQL 17 database. The datasource wiring is embedded in `docker-compose_base.yml` and is not operator-configurable. See the Persistence section.
-   **API Endpoints** - Exposes the management API (default port 11002) secured by API key, and the dataspace protocol/DSP API (default port 11003) for connector-to-connector communication.


## Environment Variables

All of the following are set in `.env`. The three port settings are read by Compose to publish the host ports; the rest are passed into the containers.

| Name                                       | Type    | Default                                                    | Description                         | Required |
| ------------------------------------------ | ------- | ---------------------------------------------------------- | ----------------------------------- | -------- |
| `SOVITY_MANAGEMENT_API_PORT`               | number  | 11002                                                      | Sovity Management Api Port          | No       |
| `SOVITY_DATASPACE_API_PORT`                | number  | 11003                                                      | Sovity Dataspace Api Port           | No       |
| `SOVITY_UI_API_PORT`                       | number  | 8080                                                       | Sovity Web UI Port                  | No       |
| `SOVITY_DEPLOYMENT_KIND`                   | string  | `control-plane-with-integrated-data-plane`                 | Sovity type of deployment           | Yes      |
| `SOVITY_DATASPACE_KIND`                    | string  | `sovity-mock-iam`                                          | Sovity dataspace kind to use        | Yes      |
| `SOVITY_MANAGEMENT_API_IAM_KIND`           | string  | `management-iam-api-key`                                   | Sovity Management Api Iam Kind      | Yes      |
| `EDC_API_AUTH_KEY`                         | string  | `demo-api-key`                                             | Sovity Edc Api Auth Key             | Yes      |
| `EDC_PARTICIPANT_ID`                       | string  | `odeon`                                                    | Sovity Edc Participant Id           | Yes      |
| `EDC_COMPONENT_ID`                         | string  | `odeon-control`                                            | Sovity Edc Component Id             | Yes      |
| `SOVITY_EDC_FQDN_INTERNAL`                 | string  | `sovity-edc`                                               | Sovity Edc Fqdn Internal            | Yes      |
| `SOVITY_EDC_FQDN_PUBLIC`                   | string  | `localhost`                                                | Sovity Edc Fqdn Public              | Yes      |
| `NEXT_PUBLIC_MANAGEMENT_API_URL`           | string  | `http://localhost:11002/api/management`                    | Sovity Management Api Url           | Yes      |
| `NEXT_PUBLIC_MANAGEMENT_API_KEY`           | string  | `demo-api-key`                                             | Sovity Management Api Key           | Yes      |
| `NEXT_PUBLIC_PROTOCOL_API_URL`             | string  | `http://localhost:11003/api/v1/dsp`                        | Sovity Protocol Api Url             | Yes      |
| `NEXT_PUBLIC_PRECONFIGURED_COUNTERPARTIES` | string  | `['http://localhost:11003/api/v1/dsp?participantId=odeon']` | Sovity Preconfigured Counterparties | Yes   |
| `EDC_WEB_REST_CORS_ENABLED`                | boolean | true                                                       | Sovity Edc CORS Enabled             | Yes      |
| `EDC_WEB_REST_CORS_ORIGINS`                | string  | `http://localhost:8080`                                    | Sovity Edc CORS Origins             | Yes      |
| `EDC_WEB_REST_CORS_METHODS`                | string  | `GET,POST,PUT,DELETE,PATCH,OPTIONS`                        | Sovity Edc CORS Methods             | Yes      |
| `EDC_WEB_REST_CORS_HEADERS`                | string  | `*,x-api-key,authorization,content-type`                   | Sovity Edc CORS Headers             | Yes      |
| `EDC_WEB_REST_CORS_CREDENTIALS`            | boolean | false                                                      | Sovity Edc CORS Credentials         | Yes      |

> **Note:** The database credentials (`POSTGRES_*` and `SOVITY_JDBC_*`) are internal wiring embedded in `docker-compose_base.yml` and are **not** operator-configurable.

## Changelog

-   v16.6.2-b.1 : Upgrade to Sovity CE 16.6.2 (connector and UI). DSP protocol path changed to `/api/v1/dsp`. **Requires a fresh database** (v16 migration history is incompatible with v15). PostgreSQL pinned to `17.10-alpine3.24`. Upstream changelog: https://github.com/sovity/edc-ce/blob/main/CHANGELOG.md
-   v15.0.1-b.1 : Initial Release (Sovity CE 15.0.1), with a bundled PostgreSQL 17 database for connector persistence.

## Provenance

Developed by **Barbara IoT** as part of the **ODEON** project. Validation completed and the application published to the Barbara Marketplace on 20 July 2026.

The upstream connector and management UI are Sovity EDC Community Edition, © sovity GmbH, used here under its own licence — see the [sovity/edc-ce](https://github.com/sovity/edc-ce) repository. This repository contains the packaging (Compose deployment, image definitions, healthchecks and configuration), not the connector source.
