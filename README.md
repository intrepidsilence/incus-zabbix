# Zabbix Template for Incus

A comprehensive Zabbix 7.x template for monitoring Incus container and VM clusters using the REST API and Prometheus metrics.

## Features

- **Cluster Monitoring**: Monitor cluster health, member status, and failover
- **Instance Monitoring**: Track containers and VMs including CPU, memory, disk, and network metrics
- **Storage Pool Monitoring**: Monitor storage pool usage and status
- **Network Monitoring**: Track managed network status
- **Image Management**: Monitor available images
- **Project Tracking**: Discover and monitor Incus projects
- **Warning Alerts**: Track active Incus warnings

## Requirements

- Zabbix Server 7.0 or later
- Incus server with REST API enabled
- TLS client certificate for API authentication

## Installation

### 1. Generate Client Certificates

On your Incus server, generate a client certificate for Zabbix:

```bash
# Generate client certificate
incus remote generate-certificate zabbix

# Or use openssl
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 \
  -sha384 -keyout client.key -nodes -out client.crt -days 3650 \
  -subj "/CN=zabbix-monitoring"

# Add the certificate to Incus trust store
incus config trust add client.crt --name zabbix-monitoring
```

### 2. Install Certificates on Zabbix Server

Copy certificates to the Zabbix server:

```bash
# Copy to Zabbix SSL directories
sudo cp client.crt /usr/share/zabbix/ssl/certs/incus-client.crt
sudo cp client.key /usr/share/zabbix/ssl/keys/incus-client.key

# Set permissions
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/keys/incus-client.key
sudo chmod 644 /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chmod 600 /usr/share/zabbix/ssl/keys/incus-client.key
```

### 3. Import Template

1. In Zabbix web interface, go to **Data collection** > **Templates**
2. Click **Import** and select `template_incus_cluster_http.yaml`
3. Click **Import**

### 4. Create Host

1. Go to **Data collection** > **Hosts**
2. Click **Create host**
3. Set hostname (e.g., "Incus Cluster")
4. Add an **Agent** interface pointing to your Incus server (used for HTTP agent items)
5. Link the **Incus Cluster by HTTP** template
6. Configure host macros:
   - `{$INCUS.API.HOST}`: Your Incus server hostname (e.g., `incus1.example.com`)
   - `{$INCUS.API.PORT}`: API port (default: `8443`)

## Configuration Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.API.HOST}` | `localhost` | Incus server hostname |
| `{$INCUS.API.PORT}` | `8443` | API port |
| `{$INCUS.API.SCHEME}` | `https` | Protocol (https recommended) |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` | Client certificate filename |
| `{$INCUS.TLS.KEY}` | `incus-client.key` | Client key filename |
| `{$INCUS.DATA.INTERVAL}` | `30s` | Data collection interval |
| `{$INCUS.STATE.INTERVAL}` | `1m` | Instance state polling interval |
| `{$INCUS.METRICS.INTERVAL}` | `1m` | Prometheus metrics interval |
| `{$INCUS.DISCOVERY.INTERVAL}` | `1h` | Discovery interval |
| `{$INCUS.INSTANCE.IGNORE}` | `^$` | Regex to ignore instances |
| `{$INCUS.NETWORK.IGNORE}` | `^(lo\|docker...)$` | Regex to ignore networks |

## Discovered Entities

The template automatically discovers:

- **Cluster Members**: All nodes in the cluster
- **Instances**: Containers and VMs across all projects
- **Network Interfaces**: Per-instance network interfaces
- **Storage Pools**: All configured storage pools
- **Images**: Available container/VM images
- **Projects**: Incus projects
- **Profiles**: Configuration profiles
- **Networks**: Managed networks

## Metrics Collected

### Cluster
- Cluster enabled status
- Member count and online status
- Per-member: status, architecture, roles, goroutines, heap memory, uptime

### Instances
- Status (Running, Stopped, Frozen)
- CPU usage (nanoseconds)
- Memory usage and peak
- Swap usage
- Disk usage (root device)
- Process count
- Network I/O per interface
- Snapshot count and age

### Storage Pools
- Total and used space
- Usage percentage
- Pool status

### Server
- API version
- Server version
- Warning count

## Triggers

The template includes triggers for:

- API connectivity issues
- Cluster member offline
- Instance stopped unexpectedly
- High CPU/memory usage
- Storage pool space warnings
- Old snapshots
- Active warnings

## License

MIT License

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
