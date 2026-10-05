# Conduit Container

A Docker-based setup to run an Angular frontend and a Django backend together in a unified, containerized environment. This project demonstrates how to package, network, and secure multiple services using Docker and Docker Compose for local development and cloud-ready deployment.

The main goal of this project is to consolidate knowledge in containerization, networking, and basic network security by operating a full-stack application stack in isolated yet connected containers.

---

## Table of Contents

- [Installation / Quickstart](#installation--quickstart)
  - [Prerequisites](#prerequisites)
  - [Setup Steps](#setup-steps)
- [Usage](#usage)
  - [Starting and Stopping Services](#starting-and-stopping-services)
  - [Creating a Django Superuser](#creating-a-django-superuser)
  - [Accessing Services](#accessing-services)
  - [Database Management](#database-management)
  - [Viewing Logs](#viewing-logs)
  - [Running Django Management Commands](#running-django-management-commands)
  - [Rebuilding Containers](#rebuilding-containers)
- [Architecture Overview](#architecture-overview)
- [CI/CD Deployment](#cicd-deployment)
  - [Overview](#overview-1)
  - [Server Prerequisites](#server-prerequisites)
  - [Optional: Clean Up GitHub Repository Tabs](#optional-clean-up-github-repository-tabs)
  - [GitHub Secrets and Variables](#github-secrets-and-variables)
  - [SSH Key Setup](#ssh-key-setup)
  - [Configure the Deployment Branch](#configure-the-deployment-branch)
  - [docker-compose.yml in Production](#docker-composeyml-in-production)
  - [Triggering a Deployment](#triggering-a-deployment)
  - [Rollback Behavior](#rollback-behavior)
- [Contributing](#contributing)
- [License](#license)

---

## Installation / Quickstart

Follow these steps to get the project up and running locally.

### Prerequisites

* Docker Desktop (Docker Engine + Docker Compose)
* Git

If Docker is not installed, follow the official installation guide:

* Docker Desktop (Windows / macOS): [https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)
* Docker Engine (Linux): [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

After installation, verify Docker:
```bash
docker --version
docker compose --version
```

### Setup Steps

1. Clone the project repository:
```bash
   git clone git@github.com:HPetersen2/conduit-container.git
   cd conduit-container
```

2. Initialize and update Git submodules (frontend & backend):

```bash
   git submodule update --init --recursive
```

3. Copy the environment template file:

```bash
   cp .env.template .env
```

4. Make sure Docker Desktop is installed and running.

5. Build and start all services:
```bash
   docker compose up --build -d
```

6. To stop and remove the containers:
```bash
   docker compose down
```

---

## Usage

After starting the stack with Docker Compose, the containers run together within a shared Docker network. The Angular frontend consumes the backend via the service name defined in `docker-compose.yml`. All configuration (ports, hostnames, secrets) is controlled via the `.env` file.

### Starting and Stopping Services

Start the stack in detached mode:
```bash
docker compose up -d
```

Start the stack with build (useful after code changes):
```bash
docker compose up --build -d
```

Stop and remove containers:
```bash
docker compose down
```

Stop containers but keep volumes (preserves database data):
```bash
docker compose down --volumes
```

### Creating a Django Superuser

To access the Django admin interface, you need to create a superuser account:
```bash
docker compose exec backend python manage.py createsuperuser
```

Follow the prompts to set a username, email, and password. After creation, you can log in to the Django admin panel.

### Accessing Services

Once the containers are running, the services are available at:

* **Frontend (Angular)**: `http://localhost:4200`
* **Backend (Django API)**: `http://localhost:8000`
* **Django Admin Panel**: `http://localhost:8000/admin`

Make sure the ports match those defined in your `.env` file.

### Database Management

Run Django migrations to set up or update the database schema:
```bash
docker compose exec backend python manage.py migrate
```

Create new migrations after model changes:
```bash
docker compose exec backend python manage.py makemigrations
```

Access the database shell:
```bash
docker compose exec backend python manage.py dbshell
```

### Viewing Logs

View logs from all services:
```bash
docker compose logs -f
```

View logs from a specific service (e.g., backend):
```bash
docker compose logs -f backend
```

View logs from the frontend:
```bash
docker compose logs -f frontend
```

### Running Django Management Commands

Execute any Django management command inside the backend container:
```bash
docker compose exec backend python manage.py <command>
```

Examples:
```bash
# Check for issues
docker compose exec backend python manage.py check

# Collect static files
docker compose exec backend python manage.py collectstatic --no-input

# Create a new Django app
docker compose exec backend python manage.py startapp myapp

# Load fixture data
docker compose exec backend python manage.py loaddata fixtures/initial_data.json
```

### Rebuilding Containers

If you've made changes to Dockerfiles or dependencies, rebuild the containers:
```bash
docker compose build
```

Rebuild a specific service:
```bash
docker compose build backend
```

Rebuild without cache (clean build):
```bash
docker compose build --no-cache
```

### Updating Submodules

If the Git submodules (frontend or backend) need to be updated, run the following command:
```bash
git pull --recurse-submodules
```

---

## Architecture Overview

* **Angular Frontend**: Built and served from a dedicated frontend container
* **Django Backend**: Runs in its own container with isolated dependencies
* **Docker Network**: Enables secure communication between services
* **Environment Variables**: Centralized configuration via `.env` file

This setup mirrors real-world cloud deployments and encourages best practices in service isolation and network security.

---

## CI/CD Deployment

### Overview

The pipeline is defined in `.github/workflows/deployment.yaml` and runs automatically on every push to the `dev` branch. It consists of two jobs:

1. **Build & Push Images** — builds the Angular frontend and Django backend as Docker images and pushes them to the GitHub Container Registry (ghcr.io).
2. **Deploy Application** — connects to the remote server via SSH, pulls the new images, and restarts the stack using Docker Compose.

### Server Prerequisites

The remote server must have the following installed and running before the first deployment:

* Docker Engine: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
* Docker Compose Plugin (V2): [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

Verify after installation:
```bash
docker --version
docker compose version
```

The project folder referenced by `PATH_TO_PROJECT_FOLDER` must exist on the server before the first deployment:
```bash
mkdir -p /path/to/your/project
```

### Optional: Clean Up GitHub Repository Tabs

By default, GitHub enables several tabs (Issues, Projects, Wiki, etc.) that may not be needed for this project. To keep the repository tidy, unused tabs can be disabled under **Settings → General → Features**.

### GitHub Secrets and Variables

All sensitive configuration is stored as GitHub Actions Secrets and must be set before the first pipeline run. Open the repository on GitHub and navigate to **Settings → Secrets and variables → Actions**.

Add the following **Secrets** (all values from `.env.template` plus the additional deployment secrets listed below):

| Secret | Description |
|---|---|
| `BACKEND_API_URL` | Public URL of the backend API, e.g. `http://YOUR_SERVER_IP:8000/api` |
| `DJANGO_SECRET_KEY` | Django secret key |
| `DJANGO_LOGLEVEL` | Log level, e.g. `INFO` |
| `DEBUG` | `False` in production |
| `DJANGO_ALLOWED_HOSTS` | Comma-separated list of allowed hosts |
| `CORS_ALLOWED_ORIGINS` | Allowed CORS origins |
| `CSRF_TRUSTED_ORIGINS` | Trusted origins for CSRF |
| `CORS_ALLOW_CREDENTIALS` | `True` or `False` |
| `CORS_ORIGIN_WHITELIST` | CORS origin whitelist |
| `DATABASE_ENGINE` | Django database engine, e.g. `django.db.backends.postgresql` |
| `DATABASE_NAME` | Database name |
| `DATABASE_USERNAME` | Database user |
| `DATABASE_PASSWORD` | Database password |
| `DATABASE_HOST` | Database host |
| `DATABASE_PORT` | Database port |
| `DJANGO_SUPERUSER_USERNAME` | Initial superuser username |
| `DJANGO_SUPERUSER_EMAIL` | Initial superuser email |
| `DJANGO_SUPERUSER_PASSWORD` | Initial superuser password |
| `REMOTE_USER` | SSH user on the remote server, e.g. `ubuntu` |
| `REMOTE_HOST` | IP address or hostname of the remote server |
| `SSH_PRIVATE_KEY` | Private SSH key used to authenticate against the server (see [SSH Key Setup](#ssh-key-setup)) |

Add the following **Variable** (under the Variables tab, not Secrets):

| Variable | Description |
|---|---|
| `PATH_TO_PROJECT_FOLDER` | Absolute path to the project folder on the server, e.g. `/home/ubuntu/conduit-container` |

### SSH Key Setup

The pipeline authenticates against the remote server using an SSH key pair. To generate the key pair directly on the server:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions
```

This creates two files:

* `~/.ssh/github_actions` — the **private key**
* `~/.ssh/github_actions.pub` — the **public key**

Add the public key to the server's list of authorized keys so the pipeline can log in:

```bash
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Then copy the contents of the private key:

```bash
cat ~/.ssh/github_actions
```

Paste the entire output (including the `-----BEGIN...` and `-----END...` lines) as the value of the `SSH_PRIVATE_KEY` secret in GitHub.

### Configure the Deployment Branch

The pipeline triggers on pushes to the `dev` branch. This is configured in `.github/workflows/deployment.yaml`:

```yaml
on:
  push:
    branches:
      - dev
```

Change this value if your main working branch has a different name.

### docker-compose.yml in Production

In production the `docker-compose.yml` does not build images locally. Instead it pulls pre-built images from the GitHub Container Registry:

```yaml
image: ghcr.io/${REPOSITORY_OWNER}/conduit-container/frontend:latest
```

The `REPOSITORY_OWNER` variable is written into the `.env` file automatically by the pipeline — no manual action is required.

### Triggering a Deployment

A deployment starts automatically on every push to the `dev` branch:

```bash
git push origin dev
```

To monitor the pipeline, open the repository on GitHub and navigate to the **Actions** tab. Select the latest workflow run to view the logs for each step in real time.

### Rollback Behavior

If the deployment fails after the new containers have been started, the pipeline automatically attempts a rollback. It restores the previously running image tags and brings the stack back up with the last known working state.

The rollback is triggered on any error during the deploy step (`trap rollback ERR`). If the rollback itself fails, the error is surfaced in the Actions log and the stack may be left in a stopped state — in that case, log into the server manually and run:

```bash
cd /path/to/your/project
docker compose up -d
```

---

## Contributing

Contributions are welcome.

1. Fork the repository
2. Create a new feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'Add new feature'`)
4. Push the branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
