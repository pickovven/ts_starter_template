# TypeScript Monorepo Architecture for Web + CLI Applications

This document describes a clean, scalable architecture for building an
application that supports both a **web interface** and a **CLI tool**,
using **TypeScript** with a **monorepo** structure.

------------------------------------------------------------------------

## Overview

The core idea is to **separate business logic** from all user
interfaces.\
Your CLI tool and your web API will both import and use the same shared
library.

This ensures: - No duplicated logic\
- Consistent behavior across interfaces\
- Easy testing\
- Clean long‑term maintainability

------------------------------------------------------------------------

## Architecture Pattern

### **Hexagonal / Clean Architecture**

The application is divided into: - **Core Layer** -- business logic,
invariants, domain models\
- **Application Layer** -- use cases and service functions\
- **Interface Adapters** - CLI (Node.js) - HTTP API (Express, Fastify,
or NestJS) - Web frontend (React/Next.js) - Database adapters (Postgres,
MongoDB, SQLite, etc.)

Each adapter calls into the **shared core**, but the core depends on
nothing external.

------------------------------------------------------------------------

## Monorepo Overview

The starter project contains:
- `packages/core` — shared business logic & types
- `apps/api` — Fastify TypeScript API server
- `apps/cli` — Node.js CLI using Commander
- `infra/` — Terraform examples & notes

## Monorepo Structure

    my-app/
      packages/
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
        
      package.json
      pnpm-workspace.yaml (or turbo.json)

------------------------------------------------------------------------

## Core Layer (Shared Logic)

- the application should utilize ES Modules (rather than commonJS)
The `core` package contains: - Domain models - Validation logic -
Use‑case functions (e.g., `createUser`, `generateReport`) - Reusable
utilities
- any utilities related to configuration should be in the core layer

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

Both CLI and API will import this.

------------------------------------------------------------------------

## CLI Layer

- Use **oclif** to build the CLI tool.

Example command:

``` ts
#!/usr/bin/env node
import { createUser } from "@my-app/core";

console.log("Creating user...");
const user = createUser({ email: "test@example.com", name: "Alice Example" });
console.log("Created:", user);
```

- The CLI should be a thin wrapper with absolute minimal business logic. The only exception might be personal or proprietary data that shouldn't be transported up to a cloud environment. 
- On build, the application should be built into a NPM package distribution. Provide instructions int he readme for how to use this.
- oclif should be configured to run with ES Modules
- The README should have instructions on how where to get the distributed binaries and how to install to the user's path
- Use workspace:* in package.json dependencies
- Make sure shared packages are compiled before running CLI.
- Packaging (oclif pack) requires referencing built JS, not raw TS.

------------------------------------------------------------------------

## API Layer

- build the API layer using Fastify
- The API should use authentication

Again, all logic lives in `core`.

------------------------------------------------------------------------

## Web Layer (Frontend)

If you include a web UI (React/Next.js), it communicates only with the
API server.

This keeps the architecture clean and ensures compatibility with any
future interface.

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

-   **PNPM workspaces**
-   **ESBuild & ES Modules** for fast builds\
-   **TypeScript project references** for compile‑time efficiency
-   **pino** for local logging
-   **Cloudwatch?? Something Else?** for log shipping
-   **OCLIF** for CLI and binary packaging for distribution
-   **Terraform** for deployment & CI/CD

------------------------------------------------------------------------