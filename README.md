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

Run the scaffolding process by following the instructions in `scaffold.md`. The tool will ask for all necessary inputs before generating the project structure.

## Prompt 

Follow scaffold.md and generate project

## Instructions

Detailed implementation guides for each architecture are available in the `instructions/` folder:

- [Clean Architecture](.github/instructions/clean-architecture.md)
- [Onion Architecture](.github/instructions/onion-architecture.md)
- [Repository + Unit of Work](.github/instructions/repository-uow-architecture.md)
- [Vertical Slice Architecture](.github/instructions/vertical-slice-architecture.md)

## Requirements

- .NET 10 SDK
- Target: Web API projects only