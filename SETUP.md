# Incus Cluster Monitoring Template - Setup Guide

This guide provides step-by-step instructions for setting up the Incus Cluster monitoring template in Zabbix 7.4+.

## Prerequisites

- Zabbix Server or Proxy 7.4.0 or later
- Incus 6.0 or later with REST API enabled
- Network connectivity from Zabbix to Incus API (default port 8443)
- OpenSSL 1.1.0 or later (for certificate generation)

## Overview

The template connects to the Incus REST API using TLS client certificate authentication. You will need to:

1. Generate a TLS client certificate for Zabbix
2. Add the certificate to Incus as a trusted client
3. Copy certificates to the Zabbix server/proxy
4. Import the template and configure the host

---

## Step 1: Generate Client Certificates

Run these commands on any machine with OpenSSL installed. You can do this on the Zabbix server, an Incus cluster member, or your workstation.

```bash
# Create a directory for the certificates
mkdir -p ~/incus-zabbix-certs
cd ~/incus-zabbix-certs

# Generate a 4096-bit RSA private key
openssl genrsa -out client.key 4096

# Generate a certificate signing request (CSR)
# The CN (Common Name) can be any descriptive name
openssl req -new -key client.key -out client.csr \
  -subj "/CN=zabbix-monitoring/O=Zabbix/OU=Monitoring"

# Generate a self-signed certificate valid for 10 years (3650 days)
openssl x509 -req -days 3650 -in client.csr \
  -signkey client.key -out client.crt \
  -sha256

# Clean up the CSR (no longer needed)
rm client.csr

# Set secure permissions
chmod 600 client.key
chmod 644 client.crt

# Verify the certificate
openssl x509 -in client.crt -text -noout | head -20
```

You should see output showing:
- Issuer: CN = zabbix-monitoring, O = Zabbix, OU = Monitoring
- Validity: 10 years from now
- Subject: CN = zabbix-monitoring, O = Zabbix, OU = Monitoring

---

## Step 2: Add Certificate to Incus Trust Store

Run this command on **any Incus cluster member** (for clusters) or on the Incus server (for standalone):

```bash
# Copy the client certificate to the Incus machine first
# Then add it to the trust store
incus config trust add-certificate /path/to/client.crt

# Verify it was added
incus config trust list
```

You should see your certificate listed with type "client".

### For Remote Certificate Addition

If you can't copy the certificate to the Incus machine, you can add it using the fingerprint:

```bash
# On the machine with the certificate, get the fingerprint
openssl x509 -in client.crt -noout -fingerprint -sha256

# On the Incus machine, create a trust token
incus config trust add zabbix-monitoring

# Use the token to add the client remotely (from Zabbix server)
# This requires the incus client on the Zabbix server
incus remote add incus-cluster <incus-server>:8443 --token=<token>
```

---

## Step 3: Get the Incus Server Certificate

Zabbix needs the Incus server certificate to verify the server's identity.

**Note for clusters:** Each Incus node has its own `server.crt` with its hostname. The template disables hostname verification when connecting to cluster members, so you only need to collect the certificate from your primary entry point node. The certificate itself is still validated, just not the hostname.

### For Clustered Incus

```bash
# On your primary entry point node (the one you'll use for {$INCUS.API.HOST})
sudo cp /var/lib/incus/server.crt ~/incus-zabbix-certs/server.crt
chmod 644 ~/incus-zabbix-certs/server.crt
```

### For Standalone Incus

```bash
# On the Incus server
sudo cp /var/lib/incus/server.crt ~/incus-zabbix-certs/server.crt
chmod 644 ~/incus-zabbix-certs/server.crt
```

---

## Step 4: Deploy Certificates to Zabbix Server/Proxy

Copy the certificates to Zabbix's default SSL directories on the Zabbix server (or proxy, if using one):

```bash
# Create the certificate directories if they don't exist
sudo mkdir -p /usr/share/zabbix/ssl/certs
sudo mkdir -p /usr/share/zabbix/ssl/keys

# Copy the certificates (from the machine where you generated them)
sudo cp client.crt /usr/share/zabbix/ssl/certs/incus-client.crt
sudo cp client.key /usr/share/zabbix/ssl/keys/incus-client.key

# Set ownership to zabbix user
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/keys/incus-client.key

# Set secure permissions
sudo chmod 644 /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chmod 600 /usr/share/zabbix/ssl/keys/incus-client.key
```

**Important:** Zabbix HTTP Agent items look for certificates relative to these directories:
- Certificates: `/usr/share/zabbix/ssl/certs/`
- Private keys: `/usr/share/zabbix/ssl/keys/`

The template macros use just the filename (e.g., `incus-client.crt`) and Zabbix automatically prepends the appropriate directory.

### Verify Permissions

```bash
ls -la /usr/share/zabbix/ssl/certs/incus-client.crt
ls -la /usr/share/zabbix/ssl/keys/incus-client.key
```

Expected output:
```
-rw-r--r-- 1 zabbix zabbix 1234 ... /usr/share/zabbix/ssl/certs/incus-client.crt
-rw------- 1 zabbix zabbix 3243 ... /usr/share/zabbix/ssl/keys/incus-client.key
```

---

## Step 5: Configure Incus to Listen on Network

By default, Incus only listens on a Unix socket. You need to enable HTTPS access.

### On Each Cluster Member (or standalone server)

```bash
# Enable HTTPS API on all interfaces, port 8443
incus config set core.https_address :8443

# Alternatively, bind to a specific IP
incus config set core.https_address 192.168.1.100:8443
```

### Verify Incus is Listening

```bash
# Check if Incus is listening on port 8443
sudo ss -tlnp | grep 8443
```

---

## Step 6: Configure Firewall

Ensure port 8443 is open for Zabbix to connect.

### Using firewalld (RHEL/CentOS/Fedora)

```bash
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload
```

### Using ufw (Ubuntu/Debian)

```bash
sudo ufw allow 8443/tcp
```

### Using iptables

```bash
sudo iptables -A INPUT -p tcp --dport 8443 -j ACCEPT
```

---

## Step 7: Test Connectivity

Before configuring Zabbix, verify connectivity from the Zabbix server:

```bash
# Test TLS connection (using -k to skip server cert verification, matching template config)
curl -v --cert /usr/share/zabbix/ssl/certs/incus-client.crt \
        --key /usr/share/zabbix/ssl/keys/incus-client.key \
        -k \
        https://<incus-host>:8443/1.0
```

**Expected Output:**
```json
{
  "type": "sync",
  "status": "Success",
  "status_code": 200,
  "metadata": {
    "api_extensions": [...],
    "api_version": "1.0",
    ...
  }
}
```

### Troubleshooting Connection Issues

| Error | Cause | Solution |
|-------|-------|----------|
| `connection refused` | Incus not listening | Run `incus config set core.https_address :8443` |
| `certificate verify failed` | Wrong server cert | Get correct cert from `/var/lib/incus/` |
| `SSL handshake failed` | Certificate not trusted | Add client cert with `incus config trust add-certificate` |
| `403 Forbidden` | Client cert not authorized | Verify cert is in `incus config trust list` |
| `connection timed out` | Firewall blocking | Open port 8443 in firewall |

---

## Step 8: Import the Template

1. Log into Zabbix web interface
2. Navigate to **Data collection** → **Templates**
3. Click **Import** (top right)
4. Select the `template_incus_cluster_http.yaml` file
5. Check all import options:
   - Create new
   - Update existing
6. Click **Import**

---

## Step 9: Create a Host for the Incus Cluster

1. Navigate to **Data collection** → **Hosts**
2. Click **Create host**
3. Configure:

| Field | Value |
|-------|-------|
| Host name | `Incus Cluster` (or your preferred name) |
| Visible name | `Incus Production Cluster` (optional) |
| Templates | Link: `Incus Cluster by HTTP` |
| Host groups | `Virtual machines` or create `Incus` |
| Interfaces | Add Agent interface (required but not used) |

### Configure Host Macros

Go to the **Macros** tab and set these inherited macros:

| Macro | Value | Description |
|-------|-------|-------------|
| `{$INCUS.API.HOST}` | `incus-node1.example.com` | Primary cluster member or standalone server |
| `{$INCUS.API.PORT}` | `8443` | API port (default) |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` | Client certificate filename (in `/usr/share/zabbix/ssl/certs/`) |
| `{$INCUS.TLS.KEY}` | `incus-client.key` | Client private key filename (in `/usr/share/zabbix/ssl/keys/`) |

### Optional Macro Overrides

| Macro | Default | Override When |
|-------|---------|---------------|
| `{$INCUS.STORAGE.PUSED.WARN}` | `80` | You want different warning threshold |
| `{$INCUS.STORAGE.PUSED.CRIT}` | `95` | You want different critical threshold |
| `{$INCUS.SNAPSHOT.MAXAGE}` | `7d` | Snapshots should be more/less frequent |
| `{$INCUS.INSTANCE.IGNORE}` | (empty) | Ignore specific instances: `^(test-.*|temp-.*)$` |
| `{$INCUS.CLUSTER.FAILOVER}` | `1` | Set to `0` for standalone Incus |
| `{$INCUS.INSTANCE.TRACK_CHANGES}` | `1` | Set to `0` to disable state change events |

4. Click **Add**

---

## Step 10: Verify Discovery

Wait for the discovery interval (default: 1 hour) or manually run discovery:

1. Navigate to **Monitoring** → **Hosts**
2. Click on your Incus host
3. Go to **Discovery rules**
4. Click **Execute now** on each discovery rule

### Check Discovered Resources

After discovery runs, check:

1. **Monitoring** → **Latest data** - Should show data for:
   - Server version and info
   - Cluster member count
   - Instance counts
   - Storage pool usage

2. **Data collection** → **Hosts** → [Your Host] → **Items** - Should show:
   - Items for each discovered cluster member
   - Items for each discovered instance
   - Items for each storage pool, network, etc.

---

## Cluster Failover Configuration

The template supports automatic failover to other cluster members if the primary entry point becomes unavailable.

### How It Works

1. Template connects to `{$INCUS.API.HOST}` (primary entry point)
2. Discovers all cluster members
3. If primary becomes unreachable, tries other discovered members
4. Generates an Info-level event when failover occurs

### Configuration

For **clustered Incus** (default):
- `{$INCUS.CLUSTER.FAILOVER}` = `1`
- Template will try other members on failure

For **standalone Incus**:
- Set `{$INCUS.CLUSTER.FAILOVER}` = `0`
- No failover logic (simpler, less overhead)

---

## Maintenance Mode

To suppress all triggers during planned maintenance:

1. Go to **Data collection** → **Hosts** → [Your Host] → **Macros**
2. Set `{$INCUS.MAINTENANCE}` = `1`
3. Perform your maintenance
4. Set `{$INCUS.MAINTENANCE}` = `0` when done

Or use Zabbix's built-in maintenance windows for the host.

---

## Template Components

### Discovery Rules

| Rule | Discovers | Update Interval |
|------|-----------|-----------------|
| Cluster members | All cluster members | 1h |
| Projects | All Incus projects | 1h |
| Instances | Containers and VMs | 1h |
| Instance network interfaces | NICs per instance | 1h |
| Storage pools | All storage pools | 1h |
| Networks | Managed networks | 1h |
| Images | Cached images | 1h |
| Profiles | Configuration profiles | 1h |

### Key Metrics Collected

**Per Cluster Member:**
- Status (Online/Offline/Evacuated)
- Daemon uptime
- Goroutines count
- Heap memory
- Active operations

**Per Instance:**
- Status (Running/Stopped/Frozen)
- CPU usage (nanoseconds)
- Memory usage/peak
- Swap usage
- Root disk usage
- Network I/O per interface
- Process count
- Snapshot count and age

**Per Storage Pool:**
- Status
- Space used/total/percentage
- Inodes used/total

**Per Network:**
- Status
- Type
- Used-by count

---

## Troubleshooting

### No Data Received

1. Check Zabbix server/proxy logs:
   ```bash
   tail -f /var/log/zabbix/zabbix_server.log | grep -i incus
   ```

2. Verify items are enabled:
   - Go to host → Items
   - Check "State" column - should be "Enabled"

3. Check for item errors:
   - Go to host → Items
   - Look for items with "Error" state
   - Click on item to see error message

### Certificate Errors

```bash
# Verify certificate matches what Incus expects
openssl x509 -in /usr/share/zabbix/ssl/certs/incus-client.crt -noout -fingerprint -sha256

# Compare with Incus trust list
incus config trust list
```

### API Timeout Errors

Increase timeout in template macros or check network latency:

```bash
# Test response time
time curl --cert /usr/share/zabbix/ssl/certs/incus-client.crt \
          --key /usr/share/zabbix/ssl/keys/incus-client.key \
          -k \
          https://<incus-host>:8443/1.0/instances?recursion=2
```

### Discovery Not Finding Resources

1. Verify API returns data:
   ```bash
   curl --cert /usr/share/zabbix/ssl/certs/incus-client.crt \
        --key /usr/share/zabbix/ssl/keys/incus-client.key \
        -k \
        https://<incus-host>:8443/1.0/instances?all-projects=true
   ```

2. Check for filter macros excluding resources:
   - `{$INCUS.INSTANCE.IGNORE}`
   - `{$INCUS.POOL.IGNORE}`
   - `{$INCUS.NETWORK.IGNORE}`

---

## Security Considerations

### Certificate Best Practices

1. **Use unique certificates per environment** - Don't share the same cert across dev/staging/prod

2. **Rotate certificates periodically** - Set a reminder to regenerate before expiration

3. **Limit certificate scope** - The template only needs read access; consider using restricted certificates if Incus supports them

4. **Secure key storage** - Ensure `client.key` is only readable by the zabbix user

### Network Security

1. **Use firewall rules** - Only allow Zabbix server/proxy to access port 8443

2. **Consider VPN/private network** - Don't expose Incus API to the internet

3. **Enable audit logging** - Track API access in Incus

---

## Upgrading

When upgrading the template:

1. Export current template (backup)
2. Import new template with "Update existing" checked
3. Review any new macros and set values
4. Force re-discovery to pick up new items

---

## Support

- Template issues: Check the template repository
- Incus issues: https://discuss.linuxcontainers.org/
- Zabbix issues: https://www.zabbix.com/forum/

---

## Quick Reference

### File Locations

| File | Location | Purpose |
|------|----------|---------|
| Client certificate | `/usr/share/zabbix/ssl/certs/incus-client.crt` | Authenticate to Incus |
| Client private key | `/usr/share/zabbix/ssl/keys/incus-client.key` | Sign requests |

### Default Ports

| Port | Service |
|------|---------|
| 8443 | Incus HTTPS API |
| 8444 | Incus Metrics (optional, separate) |

### Key Macros

| Macro | Purpose |
|-------|---------|
| `{$INCUS.API.HOST}` | Incus server hostname |
| `{$INCUS.MAINTENANCE}` | Suppress triggers (0/1) |
| `{$INCUS.CLUSTER.FAILOVER}` | Enable failover (0/1) |
| `{$INCUS.INSTANCE.TRACK_CHANGES}` | Track state changes (0/1) |
