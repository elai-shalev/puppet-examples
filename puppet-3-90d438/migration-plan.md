# MIGRATION FROM PUPPET TO ANSIBLE

## Executive Summary

This repository contains a Puppet-based infrastructure configuration for a multi-tier application stack consisting of Python applications, HAProxy load balancers, and Redis clusters. The migration to Ansible will require converting Puppet modules, classes, and Hiera data structures to Ansible roles, playbooks, and variable files. Based on the complexity and scope of the existing Puppet code, this migration is estimated to be of medium complexity with an estimated timeline of 4-6 weeks for a complete migration.

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

- **profile_app_stack**:
    - Description: Python application stack with PostgreSQL database, virtualenv management, and systemd service configuration
    - Path: modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Git-based deployment, Python virtualenv management, database migrations with Alembic, environment configuration, systemd service management

- **profile_haproxy**:
    - Description: HAProxy load balancer with multi-backend support, SSL termination, and statistics interface
    - Path: modules/profile_haproxy
    - Technology: Puppet
    - Key Features: Backend configuration, SSL certificate management, firewall rules, service monitoring

- **profile_redis_cluster**:
    - Description: Redis cluster configuration with PuppetDB node discovery
    - Path: modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis server configuration, memory management, persistence settings

- **puppetdb_query_stub**:
    - Description: Stub module for PuppetDB queries (likely for testing)
    - Path: modules/puppetdb_query_stub
    - Technology: Puppet
    - Key Features: Mocking PuppetDB functionality for testing

### Infrastructure Files

- `Puppetfile`: Defines external module dependencies including stdlib, concat, firewall, vcsrepo, redis, and apt modules
- `Vagrantfile`: Defines a development/testing environment using Ubuntu 24.04 with libvirt provider
- `vagrant-provision.sh`: Provisions the Vagrant VM with Puppet 8 and required modules
- `hiera.yaml`: Defines the Hiera hierarchy for configuration data with support for encrypted values (eyaml)
- `environment.conf`: Sets the module path for the Puppet environment
- `data/common.yaml`: Common configuration values for all environments
- `data/environment/production.yaml`: Production-specific configuration overrides
- `data/environment/staging.yaml`: Staging-specific configuration overrides
- `test/site.pp`: Test manifest that creates a stub git repository and includes all profiles

### Target Details

- **Operating System**: Ubuntu 24.04 (Noble Numbat) based on Vagrantfile and vagrant-provision.sh
- **Virtual Machine Technology**: libvirt (KVM) based on Vagrantfile provider configuration
- **Cloud Platform**: Not specified in the repository, appears to be targeting on-premises or generic cloud VMs

## Migration Approach

### Key Dependencies to Address

- **puppetlabs-stdlib (9.7.0)**: Replace with Ansible built-in filters and modules
- **puppetlabs-concat (9.0.2)**: Replace with Ansible's template module and blockinfile
- **puppetlabs-firewall (8.1.3)**: Replace with Ansible's ufw or iptables modules
- **puppetlabs-vcsrepo (6.1.0)**: Replace with Ansible's git module
- **puppet-redis (11.0.0)**: Replace with Ansible Redis role (community.general.redis or custom role)
- **puppetlabs-apt (9.4.0)**: Replace with Ansible's apt module

### Security Considerations

- **Hiera eyaml encryption**: Migrate encrypted data to Ansible Vault
  - The hiera.yaml file indicates encrypted node-specific data using eyaml_lookup_key
  - Each module may have sensitive data that needs to be identified and migrated to Ansible Vault

- **Database credentials**: 
  - The profile_app_stack module contains database credentials that need to be secured
  - Migration approach: Move to Ansible Vault variables

- **Redis password**: 
  - The profile_redis_cluster module contains Redis authentication password
  - Migration approach: Move to Ansible Vault variables

- **HAProxy statistics credentials**:
  - The profile_haproxy module contains credentials for the statistics interface
  - Migration approach: Move to Ansible Vault variables

- **SSL certificates**:
  - The profile_haproxy module manages SSL certificates
  - Migration approach: Use Ansible's crypto modules or external certificate management

- **Vault/secrets management**:
  - profile_app_stack: 2 credentials (db_password, secret_key)
  - profile_haproxy: 3 credentials (stats_password, ssl_cert_path, ssl_key_path)
  - profile_redis_cluster: 1 credential (redis_password)

### Technical Challenges

- **PuppetDB queries**: 
  - The profile_redis_cluster module uses PuppetDB queries to discover Redis nodes
  - Migration approach: Replace with Ansible inventory groups or dynamic inventory

- **Strict dependency chains**: 
  - Puppet code uses explicit dependency chains (-> and ~>) between classes
  - Migration approach: Use Ansible's handlers, pre_tasks, and post_tasks to maintain execution order

- **Custom functions**: 
  - The profile_app_stack module uses a custom function (profile_app_stack::app_db_url)
  - Migration approach: Replace with Ansible Jinja2 filters or custom Python filters

- **Hiera data hierarchy**: 
  - Complex Hiera hierarchy with environment-specific overrides
  - Migration approach: Use Ansible group_vars and host_vars with variable precedence

### Migration Order

1. **Common data structures** (low risk, foundation)
   - Migrate Hiera data to Ansible group_vars and host_vars
   - Set up Ansible Vault for secrets

2. **profile_redis_cluster** (low complexity)
   - Simple Redis configuration with minimal dependencies
   - Create Ansible role for Redis configuration

3. **profile_haproxy** (moderate complexity)
   - HAProxy configuration with SSL and firewall rules
   - Create Ansible role for HAProxy with templates

4. **profile_app_stack** (high complexity, dependencies)
   - Complex Python application deployment with database integration
   - Create Ansible roles for Python app deployment and database configuration

5. **Testing infrastructure** (final integration)
   - Migrate Vagrant testing environment to use Ansible provisioner
   - Create integration tests for the complete stack

### Assumptions

1. The target environment will remain Ubuntu 24.04 as specified in the Vagrantfile.
2. The application structure (Python app with PostgreSQL and Redis) will remain unchanged.
3. The HAProxy configuration structure (backends, SSL, stats) will remain similar.
4. The Redis cluster configuration will remain similar.
5. The current Puppet modules are functioning correctly and represent the desired state.
6. The migration will not involve architectural changes to the application stack.
7. The Ansible inventory will need to be created based on the current Puppet node definitions.
8. The eyaml encrypted data will need to be decrypted before migration to Ansible Vault.
9. The custom Puppet functions will need manual translation to Ansible equivalents.
10. The strict dependency chains in Puppet will need careful handling in Ansible to maintain proper execution order.