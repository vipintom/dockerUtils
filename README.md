# Dockerized Services

This repository contains Docker Compose configurations for various services, making it easy to set up and manage them.

## Table of Contents

- [Introduction](#introduction)
- [Services](#services)
  - [Beszel](#beszel)
  - [Kuma](#kuma)
  - [Netdata](#netdata)
  - [Portainer](#portainer)
  - [Statping](#statping)
  - [Watchtower](#watchtower)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Contributing](#contributing)
- [License](#license)

## Introduction

This project provides a collection of Docker Compose files to quickly deploy and manage various services. Each service is configured in its own directory, and a main `docker-compose.yml` file is provided to manage all the services together.

## Services

### Beszel

Beszel is a real-time communication platform.

- **Image:** `henrygd/beszel:latest`
- **Ports:** `8090:8090`
- **Volumes:** `./beszel_hub_data:/beszel_data`

### Kuma

Kuma is a monitoring and uptime tool.

- **Image:** `louislam/uptime-kuma:1`
- **Ports:** `3001:3001`
- **Volumes:** `./uptime-kuma-data:/app/data`

### Netdata

Netdata is a real-time performance monitoring tool.

- **Image:** `netdata/netdata`
- **Network Mode:** `host`
- **Volumes:**
  - `./netdataconfig:/etc/netdata`
  - `./netdatalib:/var/lib/netdata`
  - `./netdatacache:/var/cache/netdata`
  - and many more system directories for monitoring

### Portainer

Portainer is a management UI for Docker.

- **Image:** `portainer/portainer-ce:latest`
- **Ports:** `9000:9000`
- **Volumes:**
  - `/var/run/docker.sock:/var/run/docker.sock:ro`
  - `./portainer_data:/data`

### Statping

Statping is a status page for your applications.

- **Image:** `adamboutcher/statping-ng`
- **Ports:** `8080:8080`
- **Volumes:** `./statping:/app`
- **Environment:** `DB_CONN=sqlite`

### Watchtower

Watchtower is a tool to automatically update running Docker containers.

- **Image:** `containrrr/watchtower:latest`
- **Volumes:** `/var/run/docker.sock:/var/run/docker.sock:ro`
- **Command:** `--label-enable`
- **Environment:**
  - `WATCHTOWER_CLEANUP=true`
  - `WATCHTOWER_INCLUDE_RESTARTING=true`
  - `WATCHTOWER_POLL_INTERVAL=36000`

## Prerequisites

Before you begin, ensure you have met the following requirements:

- You have installed Docker and Docker Compose.
- You are on a Linux-based system.

## Installation

1. Clone this repository:

   ```bash
   git clone https://github.com/your-username/your-repository.git
   ```

2. Navigate to the project directory:

   ```bash
   cd your-repository
   ```

3. Create a `.env` file from the example:

   ```bash
   cp .env.example .env
   ```

4. Edit the `.env` file and provide the necessary values for `BESZEL_KEY` and `BESZEL_TOKEN`.

## Usage

To start all the services, run the following command from the root directory:

```bash
docker-compose up -d
```

To stop all the services, run:

```bash
docker-compose down
```

You can also manage individual services by navigating to their respective directories and using `docker-compose`.

## Configuration

Each service has its own `docker-compose.yml` file located in its respective directory. You can modify these files to customize the configuration. The main `docker-compose.yml` file in the root directory extends these individual configurations.

## Contributing

Contributions are welcome! Please open an issue or a pull request to suggest changes or improvements.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
