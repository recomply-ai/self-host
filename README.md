# recomply.ai Self-Hosted Deployment

This document outlines the deployment and operation of the recomply.ai compliance platform on your premises using Docker.

## Deployment Requirements

### Prerequisites
- Docker and Docker Compose installed on your system
- Network access to GitHub Container Registry (ghcr.io)
- The `.ghcr_token` file provided by recomply.ai team. This file contains a GitHub Personal Access Token required to pull our private container images. This token ensures secure access to the latest verified builds of our software.

## Running the system

1. **Authenticate with GitHub Container Registry**:
   ```bash
   cat .ghcr_token | docker login ghcr.io -u kovalevvlad --password-stdin
   ```

2. **Start the entire platform**:
   ```bash
   docker compose up -d
   ```

The system will automatically:
- Pull the latest verified images from our repositories
- Initialize the database with the latest schema
- Start all services in the correct dependency order
- Make the frontend available at http://localhost:3000

## Using the system

Once the service is running (see the previous step):

1) Go to http://localhost:3000
2) Use the following credentials on the login page:
   - **Username**: `admin`
   - **Password**: `secret123`
3) ☕ Enjoy!

## Data Persistence

Application data is persisted in local directories:
- PostgreSQL data: `./.data/postgres/`
- Redis data: `./.data/redis/`

These directories will be created automatically and contain all your compliance data and system state.

## System Architecture (if you're curious)

The recomply.ai platform consists of six key components orchestrated through Docker Compose:

### Core Services

- **Frontend** - Web interface serving the user dashboard and compliance management tools
- **API** - Backend REST API service handling all business logic and client requests  
- **Worker** - Asynchronous task execution process for background operations
- **Migrations** - Database schema management service ensuring data structure consistency

### Infrastructure Services

- **PostgreSQL** - Primary database for application data persistence and compliance records
- **Redis** - In-memory data store used for asynchronous task coordination via ARQ (Async Redis Queue)

All application images are automatically built and published from our code repositories after passing comprehensive tests and security checks. 