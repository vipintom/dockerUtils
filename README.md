# Docker Infrastructure Stack

A comprehensive Docker-based infrastructure stack for monitoring, management, and web services. This repository provides a complete self-hosted solution with reverse proxy, monitoring, uptime tracking, container management, and system observability.

## 🏗️ Architecture Overview

This infrastructure stack is built using Docker Compose with a modular approach, where each service has its own compose file and configuration. The main `docker-compose.yml` orchestrates all services using the `extends` feature for better maintainability.

### Core Components

- **Reverse Proxy**: Caddy with automatic HTTPS
- **Monitoring**: Prometheus + Grafana + Loki stack
- **Container Management**: Portainer for Docker GUI
- **Uptime Monitoring**: Uptime Kuma for service availability
- **System Monitoring**: Beszel for system metrics and Netdata for real-time monitoring
- **Auto-Updates**: Watchtower for automatic container updates
- **Log Aggregation**: Promtail + Loki for centralized logging

## 🚀 Quick Start

### Prerequisites

- Docker Engine 20.10+
- Docker Compose v2.0+
- Domain name pointed to your server (for SSL certificates)

### Installation

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd docker-infrastructure-stack
   ```

2. **Configure environment variables**

   ```bash
   cp .env.example .env
   # Edit .env with your specific values
   ```

3. **Update domain configuration**

   ```bash
   # Edit caddy/Caddyfile with your domain names
   nano caddy/Caddyfile
   ```

4. **Start the infrastructure**

   ```bash
   docker compose up -d
   ```

5. **Access services**
   - Portainer: `https://portainer.yourdomain.com`
   - Grafana: `https://grafana.yourdomain.com`
   - Uptime Kuma: `https://kuma.yourdomain.com`
   - Beszel: `https://beszel.yourdomain.com`

## 📋 Services Overview

### Web Services & Proxy

#### Caddy (Reverse Proxy)

- **Purpose**: Automatic HTTPS reverse proxy and web server
- **Features**:
  - Automatic SSL certificate management via Let's Encrypt
  - HTTP/2 and HTTP/3 support
  - Metrics endpoint for Prometheus
  - JSON structured logging
- **Access**: Handles all incoming traffic and routes to appropriate services
- **Configuration**: `caddy/Caddyfile`

### Monitoring Stack

#### Prometheus (Metrics Collection)

- **Purpose**: Time-series database for metrics collection
- **Features**:
  - Scrapes metrics from Caddy and other services
  - Alert rule evaluation
  - 30-second scrape interval
- **Storage**: Persistent volume for metrics data
- **Configuration**: `monitoring/prometheus.yml`

#### Grafana (Visualization)

- **Purpose**: Metrics visualization and dashboarding
- **Features**:
  - Custom dashboards for infrastructure monitoring
  - Alert management
  - Multi-datasource support (Prometheus, Loki)
- **Access**: `https://grafana.yourdomain.com`
- **Default Port**: 4004

#### Loki (Log Aggregation)

- **Purpose**: Log aggregation system
- **Features**:
  - Efficient log storage and querying
  - Integration with Grafana
  - 14-day log retention (configurable)
- **Storage**: BoltDB for indexes, filesystem for chunks
- **Configuration**: `monitoring/loki-config.yaml`

#### Promtail (Log Collection)

- **Purpose**: Log shipping agent for Loki
- **Features**:
  - Collects logs from Caddy and Docker containers
  - Service discovery for Docker containers
  - Log parsing and labeling
- **Configuration**: `monitoring/promtail-config.yaml`

### Container Management

#### Portainer (Docker GUI)

- **Purpose**: Web-based Docker management interface
- **Features**:
  - Container, image, and volume management
  - Stack deployment
  - User access control
- **Access**: `https://portainer.yourdomain.com`
- **Storage**: Persistent volume for Portainer data

#### Watchtower (Auto-Updates)

- **Purpose**: Automatic container updates
- **Features**:
  - Label-based container selection
  - Daily update checks
  - Automatic cleanup of old images
- **Configuration**: Updates containers with `com.centurylinklabs.watchtower.enable=true` label

### System Monitoring

#### Beszel (System Metrics)

- **Purpose**: Lightweight system monitoring
- **Components**:
  - **Hub**: Web interface for metrics visualization
  - **Agent**: System metrics collection (CPU, memory, disk, network)
- **Access**: `https://beszel.yourdomain.com`
- **Features**: Real-time system monitoring with minimal resource usage

#### Netdata (Real-time Monitoring)

- **Purpose**: Real-time system performance monitoring
- **Features**:
  - Per-second metrics collection
  - Interactive web dashboard
  - Anomaly detection
  - Docker container monitoring
- **Access**: Direct host network access
- **Storage**: Local cache and configuration persistence

### Uptime Monitoring

#### Uptime Kuma

- **Purpose**: Uptime and status page monitoring
- **Features**:
  - HTTP/HTTPS monitoring
  - TCP port monitoring
  - Status pages
  - Multiple notification channels
- **Access**: `https://kuma.yourdomain.com`
- **Storage**: SQLite database for monitoring data

### Additional Services

#### CaddyManager (Optional)

- **Purpose**: Web-based Caddy configuration management
- **Components**:
  - MongoDB database
  - Backend API (Node.js)
  - Frontend interface (React)
- **Status**: Currently commented out in Caddyfile
- **Ports**: Backend (3011), Frontend (8080)

## 🔧 Configuration

### Environment Variables

Create a `.env` file based on `.env.example`:

```bash
# Beszel Configuration
BESZEL_HUB_PORT=8090
BESZEL_KEY=your-beszel-key
BESZEL_TOKEN=your-beszel-token

# Grafana Configuration (add these)
GRAFANA_USERNAME=admin
GRAFANA_PASSWORD=your-secure-password
```

### Domain Configuration

Update `caddy/Caddyfile` with your domains:

```caddyfile
# Replace 010790.xyz with your domain
yourdomain.com {
    respond "Welcome to your domain"
}

portainer.yourdomain.com {
    reverse_proxy portainer:9000
}

grafana.yourdomain.com {
    reverse_proxy grafana:4004
}
# ... add other services
```

### SSL Certificates

Caddy automatically handles SSL certificates via Let's Encrypt. Ensure:

- Your domains point to your server's IP
- Ports 80 and 443 are accessible
- Email is configured in Caddyfile for certificate notifications

## 📊 Monitoring & Alerting

### Metrics Collection

The stack collects metrics from:

- **Caddy**: HTTP request metrics, response times, status codes
- **System**: CPU, memory, disk, network (via Beszel and Netdata)
- **Containers**: Docker container metrics (via Promtail)

### Log Aggregation

Logs are collected from:

- **Caddy**: Access logs and error logs
- **Docker Containers**: Container logs via Docker socket
- **System**: Various system logs

### Alerting

Basic alerting is configured for:

- High 5xx error rates on Caddy
- Custom alert rules can be added to `monitoring/alert_rules.yml`

## 🔒 Security Considerations

### Network Security

- Services communicate via internal Docker networks
- Only necessary ports are exposed to the host
- Caddy handles SSL termination

### Data Protection

- Sensitive data stored in Docker volumes
- Environment variables for secrets
- Regular automated backups recommended

### Access Control

- Portainer has built-in authentication
- Grafana requires login credentials
- Consider adding additional authentication layers for production

## 🚀 Performance Optimization

The repository includes a comprehensive `OPTIMIZATION_GUIDE.md` with:

- Storage optimizations (log rotation, retention policies)
- Performance tuning (resource limits, caching)
- Advanced configurations (object storage, service mesh)
- Monitoring improvements

Key optimizations already implemented:

- Efficient log rotation and compression
- Optimized Loki configuration for log storage
- Health checks for all critical services
- Resource-efficient container configurations

## 📁 Directory Structure

```
├── beszel/                 # Beszel system monitoring
│   ├── docker-compose.yml
│   ├── beszel_hub_data/   # Hub data (gitignored)
│   └── beszel_agent_data/ # Agent data (gitignored)
├── caddy/                 # Reverse proxy
│   ├── docker-compose.yml
│   ├── Caddyfile
│   └── logs/             # Caddy logs
├── caddymanager/         # Optional Caddy management
│   ├── docker-compose.yml
│   ├── backend/.env
│   └── frontend/.env
├── kuma/                 # Uptime monitoring
│   ├── docker-compose.yml
│   └── uptime-kuma-data/ # Kuma data (gitignored)
├── monitoring/           # Prometheus stack
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   ├── loki-config.yaml
│   ├── promtail-config.yaml
│   └── alert_rules.yml
├── netdata/              # Real-time monitoring
│   ├── docker-compose.yml
│   └── netdata*/         # Netdata data (gitignored)
├── portainer/            # Container management
│   ├── docker-compose.yml
│   └── portainer_data/   # Portainer data (gitignored)
├── watchtower/           # Auto-updates
│   └── docker-compose.yml
├── docker-compose.yml    # Main orchestration
├── .env.example         # Environment template
└── OPTIMIZATION_GUIDE.md # Performance guide
```

## 🔧 Maintenance

### Regular Tasks

1. **Monitor disk usage**: Check log sizes and data volumes
2. **Review metrics**: Use Grafana dashboards to monitor performance
3. **Update containers**: Watchtower handles this automatically
4. **Backup data**: Regular backups of Docker volumes recommended

### Troubleshooting

#### Service Health Checks

Most services include health checks. Check status with:

```bash
docker compose ps
```

#### Log Analysis

View service logs:

```bash
docker compose logs [service-name]
```

#### Resource Monitoring

- Use Grafana dashboards for historical data
- Use Netdata for real-time monitoring
- Check Beszel for system-level metrics

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📄 License

This project is open source. Please check individual service licenses for their respective terms.

## 🆘 Support

For issues and questions:

1. Check the `OPTIMIZATION_GUIDE.md` for performance issues
2. Review service logs for error details
3. Consult individual service documentation
4. Open an issue in this repository

---

**Note**: This infrastructure stack is designed for self-hosting and requires proper server administration knowledge. Always test changes in a non-production environment first.
