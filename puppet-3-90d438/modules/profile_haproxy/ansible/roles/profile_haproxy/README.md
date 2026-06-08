# profile_haproxy

An Ansible role to install and configure HAProxy load balancer.

## Requirements

- Ansible 2.9 or higher
- Supported OS: Ubuntu, Debian, RHEL/CentOS

## Role Variables

### Basic Configuration

```yaml
# Package and service configuration
package_name: haproxy
config_dir: /etc/haproxy
config_file: /etc/haproxy/haproxy.cfg
service_name: haproxy
user: haproxy
group: haproxy

# Performance settings
global_maxconn: 4000
client_timeout: 30s
server_timeout: 30s
connect_timeout: 5s
retries: 3
```

### Stats Configuration

```yaml
stats_enabled: true
stats_port: 9000
stats_uri: /stats
stats_user: admin
stats_password: "{{ vault_haproxy_stats_password | default('changeme') }}"
```

### SSL Configuration

```yaml
ssl_enabled: false
ssl_cert_path: /etc/ssl/certs
ssl_key_path: /etc/ssl/private
ssl_ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:..."
ssl_min_version: "TLSv1.2"
```

### Logging Configuration

```yaml
log_server: 127.0.0.1
log_facility: local0
log_level: notice
```

### Backend Configuration

Define backends in your inventory variables:

```yaml
backends:
  web:
    balance: roundrobin
    port: 80
    health_check: httpchk
    health_interval: 5s
    servers:
      - name: web1
        address: 192.168.1.101
        weight: 100
      - name: web2
        address: 192.168.1.102
        weight: 100
  app:
    balance: leastconn
    port: 8080
    servers:
      - name: app1
        address: 192.168.1.201
        weight: 100
```

## Example Playbook

```yaml
- hosts: loadbalancers
  vars:
    ssl_enabled: true
    backends:
      web:
        balance: roundrobin
        port: 80
        servers:
          - name: web1
            address: 192.168.1.101
            weight: 100
          - name: web2
            address: 192.168.1.102
            weight: 100
  roles:
    - profile_haproxy
```

## License

MIT

## Author Information

Ansible Migration Team