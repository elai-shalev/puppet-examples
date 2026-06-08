---
source-path: modules/profile_haproxy
---

# Migration Plan: profile_haproxy

**TLDR**: This module manages HAProxy load balancer installations with support for multiple backends, SSL termination, and statistics reporting. It provides a flexible configuration system with deep hierarchy support for environment, datacenter, cluster, and node-specific settings.

## Service Type and Instances

**Service Type**: Load Balancer (HAProxy)

**Configured Instances**:
- **HAProxy**: HTTP/HTTPS load balancer
  - Location/Path: /etc/haproxy/haproxy.cfg
  - Port/Socket: 80, 443 (when SSL enabled), 9000/9001 (stats)
  - Key Config: Multiple backends (webservers, api, internal_monitoring) with health checks and weighted load balancing

## File Structure

```
data/cluster/haproxy_prod_fra.yaml
data/common.yaml
data/datacenter/dc1_fra.yaml
data/environment/production.yaml
data/environment/staging.yaml
data/nodes/lb01.fra.example.com.yaml
data/os/Debian.yaml
lib/facter/haproxy_version.rb
manifests/config.pp
manifests/firewall.pp
manifests/init.pp
manifests/install.pp
manifests/service.pp
templates/backend.conf.epp
templates/haproxy.cfg.erb
```

## Module Explanation

The module performs operations in this order:

1. **profile_haproxy** (`manifests/init.pp`):
   - Sets class parameters from Hiera: package_name=haproxy, config_dir=/etc/haproxy, config_file=/etc/haproxy/haproxy.cfg, service_name=haproxy, user=haproxy, group=haproxy, stats_enabled=true/false (environment dependent), stats_port=9000/9001 (node dependent), stats_uri=/haproxy-stats, stats_user=admin, stats_password=encrypted, global_maxconn=4096/16384/32768 (environment/cluster dependent), client_timeout=30s/60s (environment dependent), server_timeout=30s/60s (environment dependent), connect_timeout=5s, retries=3, ssl_enabled=false/true (environment dependent), ssl_cert_path=/etc/ssl/certs, ssl_key_path=/etc/ssl/private, ssl_ciphers=ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256 (default) or ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384 (cluster specific), ssl_min_version=TLSv1.2/TLSv1.3 (cluster dependent), log_server=127.0.0.1/10.100.1.50 (datacenter dependent), log_facility=local0, log_level=info/warning/debug (environment dependent), backends=hash of backend configurations
   - Contains profile_haproxy::install
   - Contains profile_haproxy::config
   - Contains profile_haproxy::service
   - Contains profile_haproxy::firewall
   - Sets ordering: install -> config ~> service (config changes notify service restart)
   - Resources: None (orchestration only)

2. **profile_haproxy::install** (`manifests/install.pp`):
   - Installs packages: haproxy (from $package_name parameter)
   - On Debian systems, also installs extra packages: hatop
   - Resources: package (2)

3. **profile_haproxy::config** (`manifests/config.pp`):
   - Creates directory: /etc/haproxy/conf.d (mode: 0755, owner: root, group: root)
   - Deploys main HAProxy configuration template:
     - Template: haproxy.cfg.erb → /etc/haproxy/haproxy.cfg (mode: 0644, owner: root, group: root)
     - Sets: global_maxconn, user, group, log settings, timeouts, SSL settings if enabled, stats configuration if enabled
   - Iterations: Runs 3 times for backends defined in Hiera ($backends hash): webservers, api, internal_monitoring (cluster specific)
     - **webservers**:
       - Backend servers: web1-fra (10.100.1.10), web2-fra (10.100.1.11), web3-fra (10.100.1.12)
       - Load balancing: roundrobin
       - Port: 8080
       - Health check: httpchk GET /health
       - Health interval: 5s
       - Deploys backend config: backend.conf.epp → /etc/haproxy/conf.d/webservers.cfg (mode: 0644, owner: root, group: root)
     - **api**:
       - Backend servers: api1-fra (10.100.2.10, weight 200 on lb01.fra.example.com, otherwise 100), api2-fra (10.100.2.11, weight 100)
       - Load balancing: leastconn
       - Port: 3000
       - Health check: httpchk GET /api/health
       - Health interval: 10s
       - Deploys backend config: backend.conf.epp → /etc/haproxy/conf.d/api.cfg (mode: 0644, owner: root, group: root)
     - **internal_monitoring** (only in haproxy_prod_fra cluster):
       - Backend servers: prom1-fra (10.100.3.10)
       - Load balancing: roundrobin
       - Port: 9090
       - Health check: httpchk GET /-/healthy
       - Health interval: 15s
       - Deploys backend config: backend.conf.epp → /etc/haproxy/conf.d/internal_monitoring.cfg (mode: 0644, owner: root, group: root)
   - Resources: file (5) - main config plus one per backend
   - **notifies**: All file resources ~> Service['haproxy'] (restart on config change)

4. **profile_haproxy::service** (`manifests/service.pp`):
   - Manages service: haproxy
     - ensure: running
     - enable: true
     - hasstatus: true
     - hasrestart: true
   - Resources: service (1)

5. **profile_haproxy::firewall** (`manifests/firewall.pp`):
   - Opens firewall ports for HAProxy:
     - Port 80 (HTTP)
     - Port 443 (HTTPS) if ssl_enabled=true
     - Stats port (9000 or 9001 depending on node) if stats_enabled=true
   - Uses puppetlabs-firewall module to manage firewall rules
   - Resources: firewall (3) - one per port

## Variables

**Variable Flow Summary**: 29 variables across 6 Hiera levels

### Variable Definitions

**common.yaml (defaults)** → Migration note: Base defaults for all nodes
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::config_file`: `/etc/haproxy/haproxy.cfg` (type: string)
- `profile_haproxy::service_name`: `haproxy` (type: string)
- `profile_haproxy::user`: `haproxy` (type: string)
- `profile_haproxy::group`: `haproxy` (type: string)
- `profile_haproxy::stats_enabled`: `true` (type: boolean)
- `profile_haproxy::stats_port`: `9000` (type: integer)
- `profile_haproxy::stats_uri`: `/haproxy-stats` (type: string)
- `profile_haproxy::stats_user`: `admin` (type: string)
- `profile_haproxy::stats_password`: `ENC[PKCS7,MIIBygYJKoZIhvcNAQcDoIIBuzCCAbcCAQAxggEhMIIBHQIBADAFMAACAQEwDQYJKoZIhvcNAQEBBQAEggEAExample]` (type: string, encrypted)
- `profile_haproxy::global_maxconn`: `4096` (type: integer)
- `profile_haproxy::client_timeout`: `30s` (type: string)
- `profile_haproxy::server_timeout`: `30s` (type: string)
- `profile_haproxy::connect_timeout`: `5s` (type: string)
- `profile_haproxy::retries`: `3` (type: integer)
- `profile_haproxy::ssl_enabled`: `false` (type: boolean)
- `profile_haproxy::ssl_cert_path`: `/etc/ssl/certs` (type: string)
- `profile_haproxy::ssl_key_path`: `/etc/ssl/private` (type: string)
- `profile_haproxy::ssl_ciphers`: `ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256` (type: string)
- `profile_haproxy::ssl_min_version`: `TLSv1.2` (type: string)
- `profile_haproxy::log_server`: `127.0.0.1` (type: string)
- `profile_haproxy::log_facility`: `local0` (type: string)
- `profile_haproxy::log_level`: `info` (type: string)
- `profile_haproxy::backends`: (type: hash)
  ```yaml
  webservers:
    balance: roundrobin
    port: 8080
    health_check: httpchk GET /health
    health_interval: 5s
    servers:
      - name: web1
        address: 10.0.1.10
        weight: 100
      - name: web2
        address: 10.0.1.11
        weight: 100
      - name: web3
        address: 10.0.1.12
        weight: 100
  api:
    balance: leastconn
    port: 3000
    health_check: httpchk GET /api/health
    health_interval: 10s
    servers:
      - name: api1
        address: 10.0.2.10
        weight: 100
      - name: api2
        address: 10.0.2.11
        weight: 100
  ```

**os/Debian.yaml (OS-specific overrides)** → Migration note: OS-specific variables, loaded conditionally based on OS family
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::firewall_provider`: `ufw` (type: string)
- `profile_haproxy::extra_packages`: `[hatop]` (type: array)

**environment/production.yaml (environment overrides)** → Migration note: Environment-specific variables, loaded based on deployment environment
- `profile_haproxy::global_maxconn`: `16384` (type: integer)
- `profile_haproxy::ssl_enabled`: `true` (type: boolean)
- `profile_haproxy::log_level`: `warning` (type: string)
- `profile_haproxy::client_timeout`: `60s` (type: string)
- `profile_haproxy::server_timeout`: `60s` (type: string)
- `profile_haproxy::stats_enabled`: `false` (type: boolean)
- `profile_haproxy::stick_table_enabled`: `true` (type: boolean)
- `profile_haproxy::stick_table_size`: `200k` (type: string)
- `profile_haproxy::stick_table_expire`: `30m` (type: string)

**environment/staging.yaml (environment overrides)** → Migration note: Environment-specific variables, loaded based on deployment environment
- `profile_haproxy::global_maxconn`: `2048` (type: integer)
- `profile_haproxy::ssl_enabled`: `false` (type: boolean)
- `profile_haproxy::log_level`: `debug` (type: string)
- `profile_haproxy::stats_enabled`: `true` (type: boolean)
- `profile_haproxy::stick_table_enabled`: `false` (type: boolean)

**datacenter/dc1_fra.yaml (datacenter overrides)** → Migration note: Datacenter-specific variables, loaded based on datacenter fact
- `profile_haproxy::log_server`: `10.100.1.50` (type: string)
- `profile_haproxy::ntp_servers`: `[ntp1.dc1.fra.example.com, ntp2.dc1.fra.example.com]` (type: array)
- `profile_haproxy::backends`: (type: hash, partial override)
  ```yaml
  webservers:
    servers:
      - name: web1-fra
        address: 10.100.1.10
        weight: 100
      - name: web2-fra
        address: 10.100.1.11
        weight: 100
      - name: web3-fra
        address: 10.100.1.12
        weight: 100
  api:
    servers:
      - name: api1-fra
        address: 10.100.2.10
        weight: 100
      - name: api2-fra
        address: 10.100.2.11
        weight: 100
  ```

**cluster/haproxy_prod_fra.yaml (cluster overrides)** → Migration note: Cluster-specific variables, loaded based on cluster name fact
- `profile_haproxy::global_maxconn`: `32768` (type: integer)
- `profile_haproxy::ssl_ciphers`: `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384` (type: string)
- `profile_haproxy::ssl_min_version`: `TLSv1.3` (type: string)
- `profile_haproxy::backends`: (type: hash, partial override)
  ```yaml
  internal_monitoring:
    balance: roundrobin
    port: 9090
    health_check: httpchk GET /-/healthy
    health_interval: 15s
    servers:
      - name: prom1-fra
        address: 10.100.3.10
        weight: 100
  ```

**nodes/lb01.fra.example.com.yaml (node overrides)** → Migration note: Node-specific variables, loaded based on node certname
- `profile_haproxy::stats_enabled`: `true` (type: boolean)
- `profile_haproxy::stats_port`: `9001` (type: integer)
- `profile_haproxy::backends`: (type: hash, partial override)
  ```yaml
  api:
    servers:
      - name: api1-fra
        address: 10.100.2.10
        weight: 200
      - name: api2-fra
        address: 10.100.2.11
        weight: 100
  ```

### Variable Migration Summary

- **Common defaults**: 25 variables from common.yaml (base configuration for all nodes)
- **OS-specific variables**: 4 variables that vary by operating system family
- **Environment-specific variables**: 9 variables that vary by deployment environment (production, staging)
- **Datacenter-specific variables**: 3 variables that vary by datacenter location
- **Cluster-specific variables**: 4 variables that vary by cluster name
- **Host-specific variables**: 3 variables for individual host overrides
- **Encrypted variables**: 1 variable that is encrypted (eyaml) and needs secure storage

### Cross-Level Overrides

Variables defined at multiple Hiera levels:
- **profile_haproxy::global_maxconn**: defined at common, environment/production, environment/staging, cluster/haproxy_prod_fra, merge strategy: first
- **profile_haproxy::ssl_enabled**: defined at common, environment/production, environment/staging, merge strategy: first
- **profile_haproxy::log_level**: defined at common, environment/production, environment/staging, merge strategy: first
- **profile_haproxy::stats_enabled**: defined at common, environment/production, environment/staging, nodes/lb01.fra.example.com, merge strategy: first
- **profile_haproxy::stats_port**: defined at common, nodes/lb01.fra.example.com, merge strategy: first
- **profile_haproxy::ssl_ciphers**: defined at common, cluster/haproxy_prod_fra, merge strategy: first
- **profile_haproxy::ssl_min_version**: defined at common, cluster/haproxy_prod_fra, merge strategy: first
- **profile_haproxy::log_server**: defined at common, datacenter/dc1_fra, merge strategy: first
- **profile_haproxy::backends**: defined at common, datacenter/dc1_fra, cluster/haproxy_prod_fra, nodes/lb01.fra.example.com, merge strategy: deep

### Merge Strategy Notes

- Variables using `deep` merge - Hash values from multiple levels are recursively merged (deep merge)
  - `profile_haproxy::backends`: Deep merge allows datacenter, cluster, and node-specific backend configurations to override only specific parts of the backend definition while inheriting the rest

## Custom Types and Providers

### Custom Fact: haproxy_version
- **Name**: haproxy_version
- **Purpose**: Retrieves the installed HAProxy version on Linux systems
- **Implementation**: Executes 'haproxy -v' command and extracts version using regex
- **Platform constraints**: Linux only (confine kernel: 'Linux')
- **Migration notes**: This fact provides the installed HAProxy version for potential version-specific configurations. In Ansible, this should be implemented as a fact using the command module with a similar regex extraction.

## Dependencies

**External module dependencies**:
- puppetlabs-stdlib (forge, version: 9.7.0)
- puppetlabs-concat (forge, version: 9.0.2)
- puppetlabs-firewall (forge, version: 8.1.3)
- puppetlabs-vcsrepo (forge, version: 6.1.0)
- puppet-redis (forge, version: 11.0.0)
- puppetlabs-apt (forge, version: 9.4.0)

**System package dependencies**:
- haproxy
- hatop (on Debian systems)

**Service dependencies**:
- syslog service for logging

### Dependency Details

- **puppetlabs-stdlib**: Core utility functions and type validations, version 9.7.0
  - Source: forge
  - Used for: Parameter validation, data manipulation functions

- **puppetlabs-concat**: File concatenation functionality, version 9.0.2
  - Source: forge
  - Used for: Potentially combining configuration fragments

- **puppetlabs-firewall**: Firewall management, version 8.1.3
  - Source: forge
  - Used for: Opening required ports in the firewall

- **puppetlabs-vcsrepo**: Version control repository management, version 6.1.0
  - Source: forge
  - Used for: Not directly used in this module

- **puppet-redis**: Redis management, version 11.0.0
  - Source: forge
  - Used for: Not directly used in this module

- **puppetlabs-apt**: APT package management, version 9.4.0
  - Source: forge
  - Used for: Managing APT repositories on Debian systems

## Puppet Facts Used

- `$facts['kernel']` - Kernel type (Linux, Windows, etc.)
- `$facts['os']['family']` - Operating system family (Debian)
- `$facts['os']['name']` - Operating system name
- `$facts['os']['release']['full']` - Full OS version
- `$facts['os']['release']['major']` - Major OS version
- `$facts['architecture']` - System architecture
- `$facts['is_virtual']` - Whether system is virtualized
- `$facts['virtual']` - Virtualization technology
- `$::environment` - Puppet environment (production, staging)
- `$::datacenter` - Datacenter location (dc1_fra)
- `$::cluster_name` - Cluster name (haproxy_prod_fra)
- `$::role` - Node role
- `$::app_tier` - Application tier
- `$::application` - Application name
- `$::team` - Team ownership
- `$::business_unit` - Business unit
- `$::lifecycle` - Lifecycle stage
- `$::network_zone` - Network zone
- `$::region` - Geographic region
- `$::country` - Country
- `$::haproxy_version` - Custom fact for HAProxy version

## Template Conversion Notes

### haproxy.cfg.erb
- **Variables used**: log_server, log_facility, log_level, global_maxconn, user, group, ssl_enabled, ssl_ciphers, connect_timeout, client_timeout, server_timeout, retries, stats_enabled, stats_port, stats_uri, stats_user, stats_password, backends
- **Conditional rendering**: 
  - SSL configuration block rendered only if ssl_enabled=true
  - Stats configuration block rendered only if stats_enabled=true
  - HTTPS binding only if ssl_enabled=true
- **Iterations**: Loops through backends hash to reference backend configuration files

### backend.conf.epp
- **Variables used**: backend_name, balance, port, servers (array of hashes), health_check, health_interval, ssl_enabled
- **Conditional rendering**:
  - Health check option rendered only if health_check is defined
  - Health interval rendered only if health_interval is defined
  - SSL verification options added to server lines if ssl_enabled=true
- **Iterations**: Loops through servers array to create server lines with name, address, port, weight

## Checks for the Migration

**Files to verify**:
- /etc/haproxy/haproxy.cfg
- /etc/haproxy/conf.d/webservers.cfg
- /etc/haproxy/conf.d/api.cfg
- /etc/haproxy/conf.d/internal_monitoring.cfg (if in haproxy_prod_fra cluster)

**Service endpoints to check**:
- HTTP: port 80
- HTTPS: port 443 (if ssl_enabled=true)
- Stats page: port 9000 or 9001 (if stats_enabled=true)

**Templates rendered**:
- haproxy.cfg.erb → /etc/haproxy/haproxy.cfg (1 instance)
- backend.conf.epp → /etc/haproxy/conf.d/webservers.cfg (1 instance)
- backend.conf.epp → /etc/haproxy/conf.d/api.cfg (1 instance)
- backend.conf.epp → /etc/haproxy/conf.d/internal_monitoring.cfg (1 instance, cluster-specific)

## Pre-flight checks:
```bash
# Service status commands
systemctl status haproxy

# Instance-specific checks
haproxy -c -f /etc/haproxy/haproxy.cfg
curl -I http://localhost:80
curl -I https://localhost:443 --insecure
curl http://localhost:9000/haproxy-stats
curl http://localhost:9001/haproxy-stats

# Configuration validation commands
haproxy -c -f /etc/haproxy/haproxy.cfg
haproxy -c -f /etc/haproxy/conf.d/webservers.cfg
haproxy -c -f /etc/haproxy/conf.d/api.cfg
haproxy -c -f /etc/haproxy/conf.d/internal_monitoring.cfg

# Network/connectivity checks
netstat -tulpn | grep haproxy
ss -tulpn | grep haproxy
```