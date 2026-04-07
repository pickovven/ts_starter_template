# TypeScript Monorepo Architecture for Web + CLI Applications

This document describes a clean, scalable architecture for building an
application that supports both a **web interface** and a **CLI tool**,
using **TypeScript** with a **monorepo** structure.

------------------------------------------------------------------------

## Overview

The core idea is to **separate business logic** from all user
interfaces.

The CLI tool and the web application will both use the same shared methods in the backend. They will both access these methods and functions via structured data and API calls.

This structure should enforce and ensure: 
- The CLI and the Web Application both act as GUIs
- THe application **does not** duplicate any logic
- The application functions are segmented and interact with eachother in a consistent way so that new functionality can be added incrementally and existing functionality can be easily modified
- Easy testing

------------------------------------------------------------------------

## Architecture Pattern

### **Hexagonal / Clean Architecture**

The application is divided into: - **Core Layer** -- business logic,
invariants, domain models
- **Application Layer** 
  - methods and service functions that handle structured data
  - application features can be written in different languages depending on needs 
- **Interface Adapters**: 
  - CLI
  - Web frontend
  - HTTP API 
  - Database adapters 

Each adapter calls into the **shared core**, but the core depends on
nothing external.

------------------------------------------------------------------------

## Monorepo Overview

The starter project contains:
- `core` — shared business logic & types
- `api` — API server
- `frontend`— web application interface
- `cli` — CLI/Terminal interface
- `db` — database
- `infra/` — Terraform examples & notes

## Monorepo Structure

    my-app/
      core/           # All business logic and domain models
        src/
          services/
          models/
          utils/
        package.json

      cli/            # Command line tool
        src/
          commands/
          index.ts
        package.json

      api/            # Web API server
        src/
          routes/
          server.ts
        package.json

      frontend/            # Frontend (optional)
        src/
        package.json
      
      db/

      infra/

------------------------------------------------------------------------

## Core Layer (Shared Logic)
The core application should utilize the most appropriate tech stack for the job. Examplse are listed below. 

Methods and services in the core layer that can be packaged and run locally -- skipping the need for API calls -- should be setup as packages that can be exported and used in the CLI or React frontend.

### Web, HTTP, API and User Input Related Application Functions
Example services that should be handled using Typescript and Node:
- API calls
- User input handling, including validation
- The typescript application should use pnpm as its package manager

For web and HTTP related application methods, the application should use ES Modules (rather than commonJS). 

Example:

``` ts
// packages/core/src/services/userService.ts
export function createUser(data: CreateUserRequest) {
  if (!data.email.includes("@")) {
    throw new Error("Invalid email");
  }

  return {
    id: crypto.randomUUID(),
    ...data
  };
}
```

### Large Internal Data Handling
Examples of this might include DB queries and manipulation

These methods will never be done locally because of the scale and complexity. These should be handled by Python on a server.

------------------------------------------------------------------------

## CLI Layer
- The CLI should be built using Rust. 
- The CLI should be a thin wrapper with absolute minimal business logic. The only exception might be personal or proprietary data that shouldn't be transported up to a cloud environment. 
- On build, the application will be deployed (either locally or in a cloud environment) and an executable binary should be created that will be used for the CLI. 
- The CLI should have a local config setup process to connect with the application (whether locally or in the cloud) and manage authenticaion.
- The executable binaries should be determined based on the infrastructure commands that control the build.
- The README should have instructions on how where to get the distributed binaries and how to install to the user's path

------------------------------------------------------------------------

## API Layer

- build the API layer using Fastify
- The API should use authentication
- The API should automatically generate docs

------------------------------------------------------------------------

## Web Layer (Frontend)

The web layer should use Svelte with the SvelteKit so that mobile development is easy.

------------------------------------------------------------------------

## Database

The databse should use postgresql with Prisma 

------------------------------------------------------------------------

## Infrastructure 

Infrastructure deployment should be managed with Terraform

------------------------------------------------------------------------

## Logging

- Use structured JSON logs (timestamp, level, service, request_id, trace_id)
- Use pino logger with context support (pino for Node/Fastify, bunyan, etc.)
- Include request/response metadata (path, status, duration)
- Capture and propagate correlation IDs across services and clients

**Log shipping**
- For cloud: CloudWatch, Stackdriver, Datadog, New Relic, or Loki + Grafana
- Use agents or exporters in your infra (e.g., fluentd / promtail)

Example (Fastify + pino):
```ts
const server = Fastify({ logger: { level: process.env.LOG_LEVEL || 'info', prettyPrint: false } });
```

------------------------------------------------------------------------

## Environment Variables

**Principles**
- Keep secrets out of source control.
- Use 12-factor app patterns.
- Validate env vars at startup using Zod or Joi.
- Provide `.env.example` for developers (without secrets).

**Local dev**
- `.env` files (gitignored) and dotenv for loading
- Use a local secrets manager for teams (direnv, dotenv-cli)

**Production**
- Use cloud secrets manager (AWS Secrets Manager, Azure Key Vault)
- Inject secrets via CI/CD into deployment environment or use IAM roles
- Prefer mounting secrets as files or environment variables rather than baking into images

Example: env validation
```ts
import { z } from 'zod';
const env = z.object({
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(['development','production','test']).default('development'),
}).parse(process.env);
```

------------------------------------------------------------------------

## Cloud Deployment & Infrastructure as Code (IaC)

**General notes**
- Use Terraform (or Pulumi) for reproducible infra.
- Keep state remote (S3 + DynamoDB locking for Terraform).
- Use modules for repeatable components (VPC, DB, ECR/ECS, ALB).
- Apply least-privilege IAM roles and separate environments (dev/staging/prod).

**Example deployment targets**
- API: AWS ECS Fargate or AWS App Runner, or a serverless function (AWS Lambda)
- Database: RDS (Postgres), or serverless (Aurora Serverless)
- Static web: Vercel/Netlify or S3 + CloudFront
- Mobile: app stores (iOS/Android) — use Expo EAS or Fastlane for CI/CD

**CI/CD**
- Use GitHub Actions / GitLab CI / CircleCI
- Steps:
  - Run tests
  - Build and publish packages/artifacts
  - Terraform plan (comment on PR) and terraform apply on release
  - Deploy container images to ECR/registry
  - Deploy services (ECS, App Runner, or cloud run)

**Terraform example (schematic)**

```hcl
provider "aws" { region = var.aws_region }

resource "aws_ecr_repository" "api" {
  name = "myapp-api"
}
```

------------------------------------------------------------------------

## Observability & Monitoring

- Metrics: expose Prometheus metrics endpoint (or use CloudWatch metrics)
- Tracing: use OpenTelemetry to propagate traces from frontend -> API -> DB
- Alerts: set SLOs and alerts for error rate and latency

------------------------------------------------------------------------

## Security

- Use HTTPS everywhere and enforce HSTS
- Rotate secrets regularly
- Validate and sanitize inputs at the boundary
- Use OAuth2 / OIDC or JWTs for auth, and validate tokens server-side
- Run dependency scanning and container image scanning in CI

------------------------------------------------------------------------

## Tooling Overview

-   **Node package management**
-   **UV for Python Package**
-   **ESBuild & ES Modules** for fast builds\
-   **TypeScript project references** for compile‑time efficiency
-   **Rust for the CLI** for CLI and binary packaging for distribution
-   **Fastify for the API**
-   **Svelte for the Web UI Framework**
-   **Pino** for local logging
-   **Cloudwatch?? Something Else?** for log shipping
-   **Terraform** for deployment & CI/CD

------------------------------------------------------------------------

## Ongoing Development

API and UI Parity

The CLI is a thin layer that references the UI. As features as added to either the UI or the CLI, those features should be replicated in the other. So if a new API endpoint is added to support a feature in the UI, generally speaking, a command should be add to the CLI to use the new API endpoint
