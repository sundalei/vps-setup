# MongoDB

A three-member replica set across es1, es2 and es3, installed from MongoDB's
official tarballs and reachable only over the WireGuard mesh.

```
          es1 10.10.0.1 ──┐
          es2 10.10.0.2 ──┼── replica set "rs0", port 27017 on wg0
          es3 10.10.0.3 ──┘
                   ▲
                   │ WireGuard
          macbook-pro 10.10.0.10
          macbook-air 10.10.0.11
```

`mongod` listens on `127.0.0.1` and the node's tunnel address only. It is never
bound to the public interface, so it cannot be reached from the internet
regardless of firewall state.

## Running it

```bash
ansible-playbook playbooks/mongodb.yml
```

Always against the whole `mongo_cluster` group. `playbooks/mongodb.yml` imports
`bootstrap.yml`, `base.yml` and `network.yml` first, so it is safe on a fresh
host; `site.yml` runs those once itself and passes `skip_prereqs: true`.

**Never `--limit` a subset.** Three things are derived from the whole group:
the replica set member list, each node's firewall rules, and the health gate's
expected node count. Running against one host produces a wrong member list.

## Why tarballs instead of packages

A package would differ per distro — different paths, different unit files,
different user names — and this fleet is mixed (es1 Debian-family, es2 and es3
RHEL-family). Installing from the tarball means the layout is identical
everywhere, and only the tarball *name* varies.

The cost is that everything a package would have done, the role does: creates
the `mongod` user and group, the data and log directories, the systemd unit, the
logrotate config, and the `/etc/profile.d` PATH entry.

`mongodb_tarball_targets` maps `<distribution>-<major version>` to MongoDB's
build name, and `mongodb_tarball_sha256` pins each one. Two asserts at the top of
the role refuse to run on an unmapped OS or an unchecksummed build rather than
downloading something unverified.

> Checksums currently exist only for `rhel10` and `ubuntu2604`. The other mapped
> targets (`rhel93`, `ubuntu2404`, `debian12`) will fail the second assert until
> their sha256 is added from <https://downloads.mongodb.org/current.json>.

`/opt/mongodb` is a symlink to the versioned directory. **Moving that symlink is
the whole upgrade, and moving it back is the whole rollback.**

## Task order

| file | what it does |
|---|---|
| `install.yml` | user, directories, tarballs, symlinks, systemd unit, logrotate. Does **not** start anything |
| `tools.yml` | mongodump/mongorestore and friends. Optional, `mongodb_tools_install` |
| `config.yml` | keyfile and `mongod.conf`, so the very first start is already correct |
| `service.yml` | firewall, start, wait for the port |
| `replicaset.yml` | runs once: create the set, then the admin user |

The order matters. Config lands before the first start, so `mongod` never runs
with a default configuration, and the replica set is only created once every node
is already listening.

## Variables

**`inventory/group_vars/mongo_cluster/main.yml`**

| variable | meaning |
|---|---|
| `mongodb_version` | server version; must match on every node |
| `mongodb_mongosh_version` | shell, versioned separately |
| `mongodb_tools_version` | database tools, versioned separately |
| `mongodb_port` | 27017 |
| `mongodb_replset` | `rs0` |
| `mongodb_extra_client_ips` | derived from `wg_extra_peers` entries with `mongodb: true` |

**`inventory/group_vars/all/vault.yml`**

| variable | meaning |
|---|---|
| `vault_mongodb_keyfile` | shared secret proving members belong to the same set |
| `vault_mongodb_admin_password` | password for the `admin` user |

Role defaults in `roles/mongodb/defaults/main.yml` hold the paths, the tarball
maps and the checksums; they rarely need overriding.

## How the set bootstraps

MongoDB has a *localhost exception*: before any user exists, it accepts commands
from localhost without authentication, and only until the first user is created.
`replicaset.yml` works inside that window — `rs.initiate()`, wait for a primary,
then create the `admin` user. Every command after that needs credentials.

It is guarded: the first task tries `rs.status()` with credentials, and the whole
creation block is skipped if that succeeds. Re-running the playbook on a live
cluster does nothing.

Members are registered by **tunnel address**:

```js
{ _id: 0, host: "10.10.0.1:27017" }, { _id: 1, host: "10.10.0.2:27017" }, ...
```

This is the single most important fact about the deployment. Every driver does
topology discovery — it connects to a seed, asks `hello`, and is handed that
list. So **any client must be able to reach `10.10.0.0/24`**, which is why laptop
access goes over WireGuard rather than an SSH tunnel or a public port.

It also means changing a node's `wg_ip` requires an `rs.reconfig()`. The playbook
will not fix it, because `rs.initiate()` only runs when the set does not exist.

## Security model

Four independent layers, all of which must pass:

1. **Not publicly bound.** `bindIp: 127.0.0.1,<wg_ip>` — no public interface
2. **Firewall.** 27017 allowed only from peer tunnel IPs and `mongodb_extra_client_ips`
3. **Keyfile.** members authenticate to each other with a shared secret, mode 0400
4. **Authorisation.** `authorization: enabled`; clients must log in

> **No TLS on the wire.** `net.tls` is not configured, so member-to-member and
> client traffic is plaintext MongoDB wire protocol. That is acceptable *only*
> because every path runs inside WireGuard, which encrypts and authenticates it.
> Any client reached another way would be in the clear. If MongoDB is ever
> exposed outside the tunnel, TLS stops being optional.

The `admin` user holds the `root` role. There are no least-privilege application
users yet; anything connecting uses full administrative credentials.

## Adding a client

To let a new machine reach the replica set, put it on the WireGuard mesh and
grant it the firewall rule. Five steps.

**1. Pick a tunnel address.** Convention is `.1`-`.9` for servers, `.10`+ for
clients. Nothing checks for collisions — you are the address authority.

**2. Generate a keypair on the machine that will use it**, so the private key
never travels:

```bash
mkdir -p ~/.wireguard && chmod 700 ~/.wireguard
(umask 077; wg genkey > ~/.wireguard/<name>.key)
wg pubkey < ~/.wireguard/<name>.key
```

The file must be named after the peer — `wireguard-client.yml` derives the path
from `-e wg_client=<name>`.

**3. Add the peer** to `inventory/group_vars/all/main.yml`:

```yaml
wg_extra_peers:
  - name: <name>
    public_key: "<output of wg pubkey>"
    allowed_ips: 10.10.0.12/32
    mongodb: true          # omit to grant tunnel access WITHOUT database access
```

`mongodb: true` is what feeds `mongodb_extra_client_ips`. Leaving it off is how
you add a phone to the mesh without opening the database to it.

> Do not add an entry with a placeholder public key. An invalid key makes
> `wg setconf` reject `wg0.conf`, and the tunnel then fails to start **on every
> server**. Add the peer only once you have its real key.

**4. Push it to the servers.** Both are required:

```bash
ansible-playbook playbooks/network.yml      # servers learn the peer
ansible-playbook playbooks/mongodb.yml      # firewall opens 27017 for its IP
```

Skipping the first means the servers silently drop the client's packets.
Skipping the second means connections hang rather than being refused. Note that
`network.yml` rewrites `wg0.conf` on every node and restarts WireGuard — the
handler is `throttle: 1`, so nodes restart one at a time rather than partitioning
the mesh.

**5. Generate and use the client config:**

```bash
ansible-playbook playbooks/wireguard-client.yml -e wg_client=<name>
```

This writes `~/.wireguard/<name>.conf` with the servers' current public keys. It
verifies that `wg pubkey` of the local private key matches what `group_vars`
claims, and refuses rather than producing a config that would silently never
handshake.

Import the file into WireGuard.app, or bring it up from the shell:

```bash
BREW="$(brew --prefix)"
sudo env PATH="$BREW/bin:$PATH" "$BREW/bin/bash" "$BREW/bin/wg-quick" \
  up ~/.wireguard/<name>.conf
sudo "$BREW/bin/wg" show        # every peer needs a "latest handshake"
ping -c3 10.10.0.1
```

(The `bash` dance is because `wg-quick` needs bash 4+ and macOS ships 3.2, and
`sudo` strips the Homebrew path.)

**Re-run step 5 after rebuilding any server.** A reinstalled host generates a new
WireGuard keypair, and a stale `PublicKey` in a client config fails silently —
no error, just no handshake with that one peer.

## Connecting

```bash
mongosh "mongodb://admin:<pw>@10.10.0.1:27017,10.10.0.2:27017,10.10.0.3:27017/?replicaSet=rs0&authSource=admin"
```

Special characters in the password must be percent-encoded (`@` → `%40`,
`:` → `%3A`, `/` → `%2F`, `#` → `%23`, `%` → `%25`). Avoid the problem with:

```bash
mongosh "mongodb://10.10.0.1:27017,10.10.0.2:27017,10.10.0.3:27017/?replicaSet=rs0&authSource=admin" \
  --username admin
```

which prompts, reads the password literally, and keeps it out of shell history.

Expect ~200ms round trips from a laptop. Compass is chatty and will feel slow;
that is distance, not a fault.

## Checks

```bash
# on a node
mongosh --port 27017 -u admin -p --authenticationDatabase admin --eval 'rs.status().members.map(m => m.name + " " + m.stateStr)'

# same cluster everywhere -- different UUIDs mean two clusters, not one
ansible mongo_cluster -a '/opt/mongosh/bin/mongosh --quiet --port 27017 --eval "db.getMongo().getDBNames" '

# is it listening where we think?
ansible mongo_cluster -a 'ss -lntp'
ansible es2 -a 'firewall-cmd --list-rich-rules'
```

## Operations

**Upgrade.** Bump `mongodb_version`, add the new tarball's sha256, re-run. The
symlink moves and the handler restarts one node at a time (`throttle: 1`), so the
set keeps a primary throughout.

**Add a node.** Put it in the `mongo_cluster` group and re-run. Note that
`rs.initiate()` does *not* run again — a new member has to be added with
`rs.add()` by hand. The role builds the initial set, not subsequent membership
changes.

**Rebuild a node.** Two independently bootstrapped clusters have different UUIDs
and can never be merged. Clearing `path.data` is the only way to turn a
self-bootstrapped node back into a joiner — stop `mongod`, remove
`/var/lib/mongo`, recreate it `mongod:mongod` mode 2750, and re-run. Leave it
stopped until the playbook has rewritten its config.

## Known gaps

- No TLS on the wire (see above)
- No backups. The tools are installed but nothing is scheduled
- No least-privilege users; everything uses `admin` with the `root` role
- Checksums missing for `rhel93`, `ubuntu2404` and `debian12`
- `rs.add()` for new members is manual
- `mongodb_bootstrap_host` is `groups['mongo_cluster'] | first`; removing es1
  from the group silently moves the bootstrap role to another host
