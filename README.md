# VPS Infrastructure & Application Automation

Production-grade Ansible automation for provisioning multi-node VPS
infrastructure with a hardened OS baseline, an encrypted private WireGuard mesh
network, and an Apache edge reverse proxy with automated Let's Encrypt SSL/TLS
certificates.

The repository is designed to host multiple application stacks—currently
featuring a 3-node High-Availability (HA) Elasticsearch cluster and Kibana, with
architecture ready to support additional applications in the future.

Supports **Debian / Ubuntu** and **RedHat / AlmaLinux / Rocky Linux** operating
systems.

---

## Architecture Overview

```text
                          Internet / Public DNS
                                    │
           ┌────────────────────────┴────────────────────────┐
           │                                                 │
           ▼ (HTTPS / 443)                                   ▼ (HTTPS / 443)
┌──────────────────────────────┐                  ┌──────────────────────────────┐
│  es1 (187.77.1.89)           │                  │  es2 (31.97.136.158)         │
│  - kibana.sundalei.tech      │                  │  - kibana.sundalei.cloud     │
│  - es.sundalei.tech          │                  │  - es.sundalei.cloud         │
│  - [future applications]     │                  │  - [future applications]     │
├──────────────────────────────┤                  ├──────────────────────────────┤
│ Apache Edge (HTTP/2 + TLS)   │                  │ Apache Edge (HTTP/2 + TLS)   │
│   │ (127.0.0.1:5601)         │                  │   │ (127.0.0.1:5601)         │
│   ▼                          │                  │   ▼                          │
│ Kibana Daemon                │                  │ Kibana Daemon                │
│   │ (127.0.0.1:9200)         │                  │   │ (127.0.0.1:9200)         │
│   ▼                          │                  │   ▼                          │
│ Elasticsearch Node 1 (CA)    │                  │ Elasticsearch Node 2         │
└──────────────┬───────────────┘                  └──────────────┬───────────────┘
               │                                                 │
               │         WireGuard Mesh (10.10.0.0/24)           │
               └───────────────────────┬─────────────────────────┘
                                       │
                                       ▼ (Transport mTLS / 9300)
                        ┌──────────────────────────────┐
                        │  es3 (177.7.34.25)           │
                        ├──────────────────────────────┤
                        │ Elasticsearch Node 3         │
                        │ (HA Quorum / Voting Member)  │
                        └──────────────────────────────┘
```

### Key Topology Highlights

- **Shared Multi-Service Infrastructure**: Baseline system hardening, inter-node
  WireGuard encryption, and edge reverse proxying provide the foundational tier
  across all VPS instances. Distinct application services (e.g. Elasticsearch,
  Kibana, and future workloads) bind locally or over the mesh.
- **3-Node Elasticsearch Quorum (HA)**: 3 master-eligible data nodes forming a
  quorum (2 of 3 votes) resilient to single-node failure. Pinned to Elastic
  Stack `9.4.4`.
- **Zero External Transport Exposure**: Elasticsearch Transport (`9300`) binds
  exclusively to the private WireGuard mesh interface (`10.10.0.x`) with mutual
  TLS certificate verification and firewall whitelist rules.
- **Localhost Loopback HTTP**: Elasticsearch HTTP (`9200`) and Kibana (`5601`)
  bind exclusively to `127.0.0.1`. Ingress traffic is strictly routed through
  Apache.
- **Two-Phase ACME TLS Bootstrap**: Solves the chicken-and-egg bootstrap paradox
  by bringing up Apache on port `80` to serve ACME `http-01` challenge tokens,
  obtaining Let's Encrypt certificates via Certbot webroot, and then enabling
  `:443` HTTPS with HTTP/2 and permanent redirects.
- **Hardened Security**:
  - HTTP/2 multiplexing (`Protocols h2 http/1.1`) for accelerated asset loading.
  - Strict-Transport-Security (HSTS, 1 year), `X-Content-Type-Options: nosniff`,
    and `Referrer-Policy`.
  - Dynamic WebSocket connection upgrade support (`mod_proxy_wstunnel`) for
    Kibana live tails and Dev Tools.
  - Systemd renewal timer and deploy hook for automatic zero-downtime
    certificate reloads.
  - Passwordless sudo granted via validated drop-in `/etc/sudoers.d/`
    (`visudo -cf %s`).

---

## Node Inventory

Hosts are grouped by functional role in `inventory/hosts.yml`:

| Node    | Public IP       | WireGuard IP | Distro Family   | Groups & Roles                                                            |
| :------ | :-------------- | :----------- | :-------------- | :------------------------------------------------------------------------ |
| **es1** | `187.77.1.89`   | `10.10.0.1`  | Debian / RedHat | `vps`, `wireguard_mesh`, `es_cluster` (CA, Node 1), `kibana`, `web_proxy` |
| **es2** | `31.97.136.158` | `10.10.0.2`  | Debian / RedHat | `vps`, `wireguard_mesh`, `es_cluster` (Node 2), `kibana`, `web_proxy`     |
| **es3** | `177.7.34.25`   | `10.10.0.3`  | Debian / RedHat | `vps`, `wireguard_mesh`, `es_cluster` (Node 3, Quorum/Data)               |

---

## Directory Structure

```text
.
├── ansible.cfg                  # Ansible controller configuration
├── inventory/
│   ├── hosts.yml                # Inventory grouped by role (vps, wireguard_mesh, es_cluster, kibana, web_proxy)
│   ├── group_vars/
│   │   └── all/
│   │       ├── main.yml         # Non-secret global variables (ports, versions, subnets, domain toggles)
│   │       └── vault.yml        # Ansible Vault encrypted secrets (passwords, keys, hashes)
│   └── host_vars/
│       ├── es1.yml              # Host-specific network configs, public domain names, CA toggle
│       ├── es2.yml
│       └── es3.yml
├── playbooks/
│   ├── site.yml                 # Master playbook executing all layers in dependency order
│   ├── bootstrap.yml            # Initial root connection & admin user setup
│   ├── base.yml                 # System baseline (sysctl, packages, firewall defaults)
│   ├── network.yml              # WireGuard mesh installation and connectivity check
│   └── elasticsearch.yml        # Application deployment: ES Cluster -> Kibana -> Web Proxy
└── roles/
    ├── bootstrap/               # Creates non-root admin, deploys SSH key, sets sudoers
    ├── common/                  # Configures vm.max_map_count (262144), base packages, UFW/firewalld
    ├── wireguard/               # Generates keypairs, exchanges public keys via facts, brings up wg0
    ├── elasticsearch/           # Pinned install, CA/node cert generation, keystore seeding, quorum health gate
    ├── kibana/                  # Pinned install, static encryption keys, loopback service check
    └── apache/                  # Reverse proxy, Let's Encrypt TLS, and teardown (remove.yml)
```

---

## Prerequisites

1. **Ansible**: `ansible-core >= 2.14` installed locally with required
   collections:

   ```bash
   ansible-galaxy collection install ansible.posix community.general
   ```

2. **Ansible Vault Password**: Place the vault encryption password into
   `.vault_pass` in the project root (git-ignored):

   ```bash
   echo "your_vault_password" > .vault_pass
   chmod 600 .vault_pass
   ```

3. **Public DNS**: The domain names defined in `host_vars/*.yml`
   (`kibana_domain` and `es_domain`) must resolve directly to the public IPs of
   `es1` and `es2` before running the Apache / Let's Encrypt role.

---

## Playbook Execution & Workflows

### 1. Full Infrastructure Deployment

To execute the entire setup from scratch in correct dependency order:

```bash
ansible-playbook playbooks/site.yml
```

`site.yml` orchestrates the layers in sequence:

1. **`bootstrap.yml`**: Probes for root SSH access. If available (fresh server),
   provisions the administrative user (`dalei`), installs the SSH public key,
   and configures validated sudoers. Validates escalation via `id -u`.
2. **`base.yml`**: Configures kernel parameters (`vm.max_map_count = 262144`),
   refreshes package caches, installs base utilities (`iputils-ping`, `curl`,
   `unzip`), and enables host firewalls with OpenSSH allowed.
3. **`network.yml`**: Deploys WireGuard on all nodes in parallel, generates
   keypairs, exchanges public keys across hosts, brings up `wg0`, and verifies
   mesh ping reachability.
4. **`elasticsearch.yml`**:
   - Deploys the 3-node Elasticsearch cluster, creates CA and node certificates
     on `es1`, seeds keystore passwords, waits for the 3-node cluster quorum
     (`_cluster/health`), and seals the bootstrap state.
   - Deploys and starts Kibana on `es1` and `es2`.
   - Configures Apache reverse proxy and automated Let's Encrypt TLS on
     `web_proxy` nodes.

---

### 2. Running Individual Layers

Each playbook is standalone and can be triggered independently:

- **Bootstrap Only**:

  ```bash
  ansible-playbook playbooks/bootstrap.yml
  ```

- **System Baseline & Firewall**:

  ```bash
  ansible-playbook playbooks/base.yml
  ```

- **WireGuard Private Mesh**:

  ```bash
  ansible-playbook playbooks/network.yml
  ```

- **Elastic Stack & Reverse Proxy**:

  ```bash
  ansible-playbook playbooks/elasticsearch.yml
  ```

---

### 3. Role-Based Tag Execution

Use tags to target specific subsystems:

```bash
# Reconfigure only Apache reverse proxy and TLS
ansible-playbook playbooks/elasticsearch.yml --tags apache

# Re-evaluate Elasticsearch cluster configuration
ansible-playbook playbooks/elasticsearch.yml --tags elasticsearch

# WireGuard network maintenance
ansible-playbook playbooks/network.yml --tags wireguard
```

---

## Apache Reverse Proxy Lifecycle: Setup & Teardown

The `apache` role manages the edge reverse proxy tier and supports both
deployment (`present`) and complete teardown (`absent`) via `apache_state`.

### 1. Deployment (`apache_state: present`, Default)

Under `apache_state: present`, the role executes:

- Installs Apache and required modules (`proxy`, `proxy_http`, `proxy_wstunnel`,
  `ssl`, `rewrite`, `headers`, `http2`).
- Configures host firewalls (`firewalld` on RedHat, `ufw` on Debian) to allow
  ports 80/TCP and 443/TCP.
- Sets SELinux boolean `httpd_can_network_connect` on RedHat.
- Deploys initial HTTP `:80` vhosts for Let's Encrypt ACME `http-01` challenge
  validation.
- Obtains certificates via Certbot `--webroot`.
- Enables systemd renewal timers and installs deploy reload hooks.
- Renders HTTPS `:443` vhosts with HTTP/2, HSTS, security headers, reverse proxy
  rules, and WebSocket upgrade handling.

### 2. Teardown & Removal (`apache_state: absent`, `remove.yml`)

To cleanly decommission or remove the Apache proxy without disrupting underlying
application services:

```bash
ansible-playbook playbooks/elasticsearch.yml --tags apache -e apache_state=absent
```

When `apache_state == 'absent'`,
[roles/apache/tasks/remove.yml](roles/apache/tasks/remove.yml) executes:

- **Stop and Disable Service**: The Apache daemon (`apache2` or `httpd`) is
  stopped and disabled in systemd, freeing ports 80 and 443.
- **Remove VirtualHost Configurations**:
  - Deletes vhost files (`*-http.conf` and `*-tls.conf`) from
    `/etc/httpd/conf.d/` (RedHat) or `/etc/apache2/sites-available/` (Debian).
  - Removes enabled vhost symlinks from `/etc/apache2/sites-enabled/` (Debian).
- **Remove Renewal Deploy Hooks**: Deletes
  `/etc/letsencrypt/renewal-hooks/deploy/10-reload-apache.sh`.
- **Close Firewall Ports**: Disables HTTP (`80`) and HTTPS (`443`) rules in
  `firewalld` (RedHat) or `ufw` (Debian).

#### Deliberately Preserved During Removal

- **Certificates (`/etc/letsencrypt/`)**: Let's Encrypt certificates and account
  keys are **not deleted**. Let's Encrypt rate limits issuance (5 duplicate
  certificates per domain per week). Retaining existing certificates ensures
  that re-running `apache_state=present` later will succeed immediately without
  hitting issuance limits.
- **Packages**: Apache packages are not purged to avoid cascading package
  removals on RedHat (where removing `httpd` can pull down dependencies).

#### Restoring the Proxy

To re-enable and restore the Apache proxy at any time:

```bash
ansible-playbook playbooks/elasticsearch.yml --tags apache
```

---

## Multi-Application Architecture & Future Apps

While the current deployment focuses on Elasticsearch and Kibana, the repository
is structured to support hosting additional applications across the VPS nodes:

1. **Shared Foundation**: Every node joins the `vps` and `wireguard_mesh`
   groups, providing hardened SSH, sysctl tuning, host firewalls, and private
   mesh communication (`10.10.0.0/24`).
2. **Adding New Services**:
   - New applications (e.g., Docker containers, custom web applications,
     monitoring stacks) can be deployed via their own role and playbook.
   - Applications can bind locally on `127.0.0.1` or communicate securely across
     nodes over their private WireGuard IPs (`10.10.0.x`).
3. **Public Exposure via Apache**:
   - Public-facing services can be registered with the `web_proxy` tier by
     adding site definitions to `apache_sites` in host variables or group
     variables, specifying the public domain, loopback backend port, and
     WebSocket requirements.

---

## Security Model

| Layer                       | Implementation                         | Security Control                                                                                                                                                                                                 |
| :-------------------------- | :------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SSH Access**              | `bootstrap` role                       | Key-only authentication (`authorized_key` with `exclusive: true`), passwordless sudo restricted via validated drop-in.                                                                                           |
| **System Firewall**         | `common`, `wireguard`, `elasticsearch` | Default `deny` incoming policy. Explicitly opens port 22, 51820/UDP, 80/TCP, 443/TCP. Port 9300 is firewalled to allow only WireGuard mesh peers.                                                                |
| **Inter-Node Mesh**         | `wireguard` role                       | Peer-to-peer noise-protocol encryption with persistent keepalive (25s) across public endpoints.                                                                                                                  |
| **Elasticsearch Transport** | `elasticsearch` role                   | Dedicated internal CA, per-node PEM certificates, Subject Alternative Names (SAN) bound to WireGuard IPs, mutual TLS certificate verification.                                                                   |
| **Edge Reverse Proxy**      | `apache` role                          | HTTP/2 enabled (`Protocols h2 http/1.1`), HSTS (`max-age=31536000`), MIME sniffing protection (`nosniff`), Referrer Policy (`strict-origin-when-cross-origin`), forward proxying disabled (`ProxyRequests off`). |
| **Secrets Management**      | `ansible-vault`                        | All passwords (ES superuser, Kibana system, user password hash) and static Kibana encryption keys encrypted in `inventory/group_vars/all/vault.yml`.                                                             |

---

## Maintenance & Day-2 Operations

### Managing Vault Secrets

To inspect or edit encrypted credentials:

```bash
# View vault contents
ansible-vault view inventory/group_vars/all/vault.yml

# Edit vault variables
ansible-vault edit inventory/group_vars/all/vault.yml
```

### Let's Encrypt Staging Mode

When testing new nodes or re-provisioning to avoid hitting Let's Encrypt rate
limits (5 duplicate certs per week):

1. Set `apache_certbot_staging: true` in `inventory/group_vars/all/main.yml` (or
   via `-e apache_certbot_staging=true`).
2. Run the Apache role:

   ```bash
   ansible-playbook playbooks/elasticsearch.yml --tags apache -e apache_certbot_staging=true
   ```

3. Once validated, set `apache_certbot_staging: false` and re-run to obtain
   production certificates.

### Checking Cluster Status

From any node running Kibana or ES (e.g. `es1`):

```bash
# Cluster health
curl -s -u elastic:<elastic_password> http://127.0.0.1:9200/_cluster/health?pretty

# Cluster nodes
curl -s -u elastic:<elastic_password> http://127.0.0.1:9200/_cat/nodes?v
```

### Code Quality & Linting

Verify lint standards across the repository:

```bash
# Run ansible-lint
ansible-lint

# Syntax check all playbooks
ansible-playbook --syntax-check playbooks/*.yml
```
