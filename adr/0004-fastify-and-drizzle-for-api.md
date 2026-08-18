# 0004. Fastify and Drizzle ORM for the API

Date: 2026-08-18
Status: Accepted

## Context

The API is TypeScript on Node.js. It needs schema validated routes with typed handlers, predictable JSON errors, request ids and structured logging, and a query layer that gives full TypeScript inference without hiding SQL.

## Decision

Fastify 5 with `fastify-type-provider-zod` for request and response schemas, pino for logging. Drizzle ORM with `drizzle-kit` for the schema and generated SQL migrations, over the `postgres` driver. Application rules live in service classes; repositories own the SQL; routes only translate HTTP.

## Consequences

- Zod schemas are the single source for validation, serialization and TypeScript types on each route.
- Drizzle keeps queries close to SQL and infers row types from the schema, so `any` does not creep in. Migrations are plain SQL committed to the repository.
- Business logic must not depend on Drizzle specifics so the query layer could be replaced; services take a `DbOrTx` handle and repositories are the only Drizzle consumers.
- Under NodeNext module resolution the zod type provider needs a small ambient declaration for a deep `fastify/types/schema` import; it is documented in the api source.

## Alternatives considered

- Express: no built in schema types, slower, more middleware glue.
- NestJS: heavy decorators and dependency injection for a service that wants to stay boring.
- Prisma: strong developer experience but a separate engine, generated client and less direct SQL; Drizzle fits a team that wants to read the queries.
