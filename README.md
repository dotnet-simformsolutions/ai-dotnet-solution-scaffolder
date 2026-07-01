# Scaffolder for .NET 10 Web API

A scaffolding engine for generating .NET 10 Web API solutions with various architectural patterns.

## Overview

This tool acts as a senior .NET architect to scaffold complete Web API projects based on user requirements. It supports multiple database providers, authentication methods, and architectural patterns.

## Supported Architectures

- Clean Architecture
- Repository + Unit of Work Pattern
- Onion Architecture
- Vertical Slice Architecture

## Features

- **Database Providers**: SQL Server or PostgreSQL
- **Authentication**: ASP.NET Core Identity + JWT, Microsoft Entra ID, Microsoft Authentication, Google OAuth2
- **Cloud Providers**: Azure or AWS with optional storage (Blob Storage/S3) and secrets management (Key Vault/Secrets Manager)
- **Third-Party Integrations**: Typed HttpClient with Polly resilience for external services
- **Multi-Tenant Support**: Optional tenant isolation
- **Docker Support**: Dockerfile and docker-compose generation
- **Testing**: Unit and Integration test projects (XUnit, AutoFixture, Moq, FluentAssertions)

## Usage

In Claude Code, just ask Claude to scaffold a new .NET Web API solution — the `scaffold-dotnet-webapi` skill auto-triggers and asks for all necessary inputs before generating the project structure.

## Prompt

Scaffold a new .NET Web API solution

## Instructions

Detailed implementation guides for each architecture are bundled inside the scaffolding skill's `references/` folder:

- [Clean Architecture](.claude/skills/scaffold-dotnet-webapi/references/clean-architecture.md)
- [Onion Architecture](.claude/skills/scaffold-dotnet-webapi/references/onion-architecture.md)
- [Repository + Unit of Work](.claude/skills/scaffold-dotnet-webapi/references/repository-uow-architecture.md)
- [Vertical Slice Architecture](.claude/skills/scaffold-dotnet-webapi/references/vertical-slice-architecture.md)

C# performance guidance is defined as a path-scoped Claude Code rule at [`.claude/rules/csharp-performance.md`](.claude/rules/csharp-performance.md), automatically applied whenever Claude works with `*.cs` files.

## Claude Structure

This repo is a scaffolding tool, not an application — it has no buildable code of its own. Claude-facing configuration lives under `.claude/`:

- `skills/scaffold-dotnet-webapi/` — the main scaffolding workflow (this repo's core purpose)
- `skills/*` — additional analysis skills (e.g. `csharp-perf-analysis`, `efcore-query-analysis`, `tech-debt-scoring`)
- `rules/` — path-scoped guidance auto-applied to matching files (e.g. `csharp-performance.md` for `*.cs`)
- `agents/` — reusable subagents copied into generated solutions
- `commands/` — reusable slash commands

Every solution *generated* by this tool gets its own `CLAUDE.md` — that's a separate file created fresh each run, not part of this repo.

## Requirements

- .NET 10 SDK
- Target: Web API projects only