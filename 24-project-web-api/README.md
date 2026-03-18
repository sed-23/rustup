# Stage 24: Real-World Project — Web API 🌐

> *Build a production-grade REST API. Database, authentication, middleware, deployment — the full stack.*

## What You'll Build

A complete REST API using Axum (Rust's most popular modern web framework), with a database, authentication, middleware, and deployment configuration.

## What You'll Learn

- Setting up Axum web server
- REST API design (routes, handlers, extractors)
- Database integration with SQLx (PostgreSQL)
- Authentication with JWT
- Middleware (logging, CORS, rate limiting)
- Request validation and serialization with serde
- Deploying your Rust web service

## Prerequisites

- Completed [Stage 23: Project — CLI Tool](../23-project-cli/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Axum Setup & Hello Server](./01-axum-setup.md) | Creating an Axum project, first route, running the server |
| 2 | [Routes, Handlers & Extractors](./02-routes-and-handlers.md) | Path, Query, Json extractors, response types |
| 3 | [Database with SQLx](./03-database-sqlx.md) | PostgreSQL setup, migrations, CRUD operations |
| 4 | [Serialization with serde](./04-serialization.md) | JSON serialization/deserialization, custom formats |
| 5 | [Authentication & JWT](./05-authentication.md) | JWT tokens, middleware-based auth, password hashing |
| 6 | [Middleware & Error Handling](./06-middleware.md) | Tower layers, CORS, logging, rate limiting, error responses |
| 7 | [Deployment](./07-deployment.md) | Docker, environment variables, health checks, production tips |

## By the End of This Stage

You'll have a production-ready REST API. You'll understand the Rust web ecosystem and be ready to build real backend services.

---

**Previous:** [← Stage 23: Project — CLI Tool](../23-project-cli/) | **Next:** [Stage 25: Ecosystem & Patterns →](../25-ecosystem-and-patterns/)
