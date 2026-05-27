# pg-patroni-etcd

PostgreSQL 18 high-availability cluster using Patroni and a 3-node etcd ensemble,
running entirely in Docker Compose inside a Fedora 42 VM provisioned by Vagrant.

Patroni handles leader election and automatic failover. etcd provides the distributed
consensus backend. The standby runs in synchronous mode — a transaction is not
acknowledged until it is written on both nodes, so no data is lost if the leader dies.

---

## Architecture

```
                        ┌──────────────────────────────────────┐
                        │   Fedora 42 VM (Vagrant + libvirt)   │
                        │                                      │
                        │  ┌──────────────────────────────┐    │
                        │  │   Docker network: pgnet      │    │
                        │  │                              │    │
                        │  │  etcd1 ─┐                    │    │
                        │  │  etcd2 ─┼── Raft consensus   │    │
                        │  │  etcd3 ─┘       │            │    │
                        │  │                 │ DCS        │    │
                        │  │         ┌───────┴────────┐   │    │
                        │  │         │   pg-node1   │   │    │
                        │  │         │  Patroni+PG18  │   │    │
                        │  │         │  (role varies) │   │    │
                        │  │         └───────┬────────┘   │    │
                        │  │                 │            │    │
                        │  │      synchronous replication │    │
                        │  │                 │            │    │
                        │  │         ┌───────┴────────┐   │    │
                        │  │         │    pg-node2     │   │    │
                        │  │         │  Patroni+PG18  │   │    │
                        │  │         │  (role varies) │   │    │
                        │  │         └────────────────┘   │    │
                        │  └──────────────────────────────┘    │
                        └──────────────────────────────────────┘
```

### Components

| Container | Role | Ports |
|-----------|------|-------|
| `etcd1`, `etcd2`, `etcd3` | Raft consensus cluster — Patroni DCS backend | 2379 (client), 2380 (peer) |
| `pg-node1` | PostgreSQL 18 + Patroni — leader or sync standby (role varies) | 5432, 8008 (REST API) |
| `pg-node2` | PostgreSQL 18 + Patroni — leader or sync standby (role varies) | 5432, 8008 (REST API) |

### How failover works

1. Patroni on each PostgreSQL node holds a lease in etcd. The leader renews it every
   `loop_wait` seconds (10 s).
2. If the leader misses enough renewals, etcd expires the key and the cluster enters
   an election.
3. The sync standby is guaranteed to be up to date (synchronous replication), so it
   is promoted immediately without any data loss.
4. The old primary comes back, detects it is no longer the leader, and uses
   `pg_rewind` to fast-forward its WAL to the new leader's timeline. It then
   rejoins as a replica — no manual intervention required.

---

## Stack

| Component | Version | Source |
|-----------|---------|--------|
| PostgreSQL | 18 (PGDG) | `pgdg-fedora-repo` |
| Patroni | 4.x | `pip install patroni[psycopg3,etcd3]` |
| etcd | 3.5 | `dnf install etcd` |
| Base image | `fedora:42` | Docker Hub |

---

## Prerequisites

- Vagrant with the `vagrant-libvirt` provider
- libvirt / KVM on the host
- 2 GB RAM and 2 vCPUs available for the VM

---

## Quick start

```bash
git clone <repo>
cd pg-patroni-etcd
vagrant up
```

Vagrant provisions a Fedora 42 VM, installs Docker CE, builds the images, and starts
all five containers. First run takes a few minutes while images build and Patroni
initialises the cluster.

SSH into the VM and check cluster state:

```bash
vagrant ssh
docker exec pg-node1 patronictl -c /etc/patroni/patroni.yml list
```

Expected output once the cluster is healthy:

```
+ Cluster: pg-cluster (xxxxxxxxxxxxxxxx) +-----------+----+-----------+
| Member     | Host       | Role         | State     | TL | Lag in MB |
+------------+------------+--------------+-----------+----+-----------+
| pg-node1 | pg-node1 | Leader       | running   |  1 |           |
| pg-node2    | pg-node2    | Sync Standby | streaming |  1 |         0 |
+------------+------------+--------------+-----------+----+-----------+
```

---

## Verifying replication

Connect to the leader and write data:

```bash
docker exec -it pg-node1 psql -U postgres -c "
  CREATE TABLE test (id serial, val text);
  INSERT INTO test (val) VALUES ('hello from primary');
"
docker exec -it pg-node2 psql -U postgres -c "SELECT * FROM test;"
```

## Check Patroni REST API

```bash
docker exec pg-node1 curl -s http://localhost:8008/leader | python3 -m json.tool
docker exec pg-node1 curl -s http://localhost:8008/health
docker exec pg-node2 curl -s http://localhost:8008/health
```

---

## Testing failover

```bash
# 1. Stop the leader
docker stop pg-node1

# 2. Watch pg-node2 get promoted 
docker exec pg-node2 patronictl -c /etc/patroni/patroni.yml list

# 3. Confirm writes now go to the promoted node
docker exec -it pg-node2 psql -U postgres -c "INSERT INTO test(val) VALUES ('after failover');"

# 4. Bring the old primary back — it rejoins as a replica via pg_rewind
docker start pg-node1
sleep 5
docker exec pg-node1 patronictl -c /etc/patroni/patroni.yml list
```

After step 4 the cluster is healthy again on timeline 2, with the roles swapped.

---

## Connecting

| Parameter | Value |
|-----------|-------|
| Host | `localhost` (from inside the VM) |
| Port | `5432` |
| Superuser | `postgres` / `postgres` |
| Replication user | `replicator` / `replicator` |
| Auth method (TCP) | SCRAM-SHA-256 |
| Auth method (socket) | peer |

---

## Project files

```
.
├── Dockerfile          # PostgreSQL 18 + Patroni image
├── Dockerfile.etcd     # etcd image (minimal — just dnf install etcd)
├── docker-compose.yml  # 3 etcd + 2 Patroni/PostgreSQL services
├── patroni.yml         # Shared Patroni config; per-node identity via env vars
├── docker.sh           # Vagrant shell provisioner — installs Docker, starts cluster
├── Vagrantfile         # Fedora 42 VM, libvirt provider, rsync shared folder
└── README.md
```

`patroni.yml` is bind-mounted read-only into both PostgreSQL containers. The values
that differ between nodes (`PATRONI_NAME`, `PATRONI_POSTGRESQL_CONNECT_ADDRESS`) are
injected as environment variables in `docker-compose.yml`, so a single config file
drives the whole cluster.

---

## Resetting the cluster

```bash
# Destroy all containers and volumes, then rebuild from scratch
docker compose down -v && docker compose up -d --build
```

The `-v` flag removes named volumes, forcing PostgreSQL to reinitialise and Patroni
to bootstrap a fresh cluster on the next start.
