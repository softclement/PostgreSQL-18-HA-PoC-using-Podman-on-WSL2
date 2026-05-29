# PostgreSQL 18 HA PoC using Podman on WSL2

## Environment
- OS: Ubuntu 24.04 (Noble) on WSL2
- Podman: 4.9.3
- podman-compose: 1.0.6
- PostgreSQL: 18.4

<img width="332" height="241" alt="image" src="https://github.com/user-attachments/assets/f437a5ff-4345-488f-a6a6-63a328ea7d91" />

## Key Notes for PostgreSQL 18 + Podman

- Mount at `/var/lib/postgresql` not `/var/lib/postgresql/data`. Data lives at `/var/lib/postgresql/18/docker/`.
- Named volumes required — podman-compose prefixes them with the project name (`pg-podman-ha_pg-primary-data`).
- `primary_conninfo` is written to `postgresql.auto.conf` by `pg_basebackup -R`.
- Use plain container name `pg-primary` as hostname — no stack prefix needed (unlike Docker Swarm's `pgcluster_pg-primary`).
- `dns_enabled: true` on the Podman bridge network handles container name resolution automatically.
- Podman is daemonless and rootless — no `dockerd`, no root required.
- systemd user units replace Docker Swarm's replica auto-restart.
- `podman generate systemd` is deprecated in Podman 4.9+ — Quadlets are the modern replacement (future improvement).

---

## STEP 1 — Install Podman

```bash
sudo apt-get update
sudo apt-get install -y podman
podman --version
```

Expected: `podman version 4.9.3`

If the Ubuntu repo has an old version (below 4.x), install from the official kubic repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/unstable/xUbuntu_22.04/Release.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/devel_kubic_libcontainers.gpg > /dev/null

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/devel_kubic_libcontainers.gpg] \
  https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/unstable/xUbuntu_22.04/ /" \
  | sudo tee /etc/apt/sources.list.d/devel_kubic_libcontainers.list

sudo apt-get update && sudo apt-get install -y podman
```

---

## STEP 2 — Install podman-compose

On Ubuntu 24.04, `pip3 install podman-compose` fails due to PEP 668. Use apt directly:

```bash
sudo apt install -y podman-compose
podman-compose version
```

Expected:
```
podman-compose version: 1.0.6
podman version: 4.9.3
```

Alternatively install via pipx (gets newer version 1.5.0):

```bash
sudo apt-get install -y pipx
pipx install podman-compose
pipx ensurepath
# Open a new terminal for PATH to take effect
```

---

## STEP 3 — Verify Rootless Setup

```bash
podman info | grep -E "rootless|graphDriverName"
```

Expected:
```
rootless: true
graphDriverName: overlay
```

**Notes:**
- `WARN: "/" is not a shared mount` — cosmetic WSL2 warning, safe to ignore.
- `/proc/sys/kernel/unprivileged_userns_clone` not present on modern WSL2 kernels — this is correct, no action needed. User namespaces are always enabled.

---

## STEP 4 — Complete Cleanup (run before starting fresh)

```bash
podman stop -a 2>/dev/null
podman rm -af 2>/dev/null
podman volume prune -f 2>/dev/null
podman network prune -f 2>/dev/null
sudo rm -rf ~/pg-podman-ha
```

---

## STEP 5 — Pull PostgreSQL 18

```bash
podman pull docker.io/library/postgres:18
podman images
```

Always use the full registry prefix `docker.io/library/` to avoid short-name resolution warnings.

---

## STEP 6 — Create Working Directory

```bash
mkdir ~/pg-podman-ha
cd ~/pg-podman-ha
mkdir standby_base
```

---

## STEP 7 — Create Podman Network

```bash
podman network create pg-net
podman network inspect pg-net
```

Verify `"dns_enabled": true` in the output. This enables automatic container name DNS resolution — `pg-primary`, `pg-standby1`, and `pg-standby2` are resolvable by name within the network.

---

## STEP 8 — Create compose.yml

```bash
cat > ~/pg-podman-ha/compose.yml << 'COMPOSEFILE'
version: "3.9"

services:

  pg-primary:
    image: docker.io/library/postgres:18
    container_name: pg-primary
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - pg-primary-data:/var/lib/postgresql
    networks:
      - pg-net
    restart: unless-stopped

  pg-standby1:
    image: docker.io/library/postgres:18
    container_name: pg-standby1
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6001:5432"
    volumes:
      - pg-standby1-data:/var/lib/postgresql
    networks:
      - pg-net
    restart: unless-stopped

  pg-standby2:
    image: docker.io/library/postgres:18
    container_name: pg-standby2
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6002:5432"
    volumes:
      - pg-standby2-data:/var/lib/postgresql
    networks:
      - pg-net
    restart: unless-stopped

volumes:
  pg-primary-data:
  pg-standby1-data:
  pg-standby2-data:

networks:
  pg-net:
    external: true
COMPOSEFILE
```

Key differences from Docker Swarm stack file:
- `container_name:` set explicitly — becomes the DNS name on the network
- No `deploy:` section (Swarm-only syntax)
- `restart: unless-stopped` replaces Swarm replica auto-restart
- Full `docker.io/library/` prefix on image name
- Network declared as `external: true` (created manually in Step 7)
- podman-compose prefixes volume names with project name: actual names become `pg-podman-ha_pg-primary-data` etc.

---

## STEP 9 — Deploy Stack

```bash
cd ~/pg-podman-ha
podman-compose up -d
podman ps
```

Expected: three containers running.

```
CONTAINER ID  IMAGE                          COMMAND   CREATED        STATUS        PORTS                   NAMES
xxxxxxxxxxxx  docker.io/library/postgres:18  postgres  8 seconds ago  Up 8 seconds  0.0.0.0:5434->5432/tcp  pg-primary
xxxxxxxxxxxx  docker.io/library/postgres:18  postgres  5 seconds ago  Up 4 seconds  0.0.0.0:6001->5432/tcp  pg-standby1
xxxxxxxxxxxx  docker.io/library/postgres:18  postgres  1 second ago   Up 1 second   0.0.0.0:6002->5432/tcp  pg-standby2
```

---

## STEP 10 — Create Replication User

```bash
podman exec -it pg-primary psql -U postgres
```

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass';
SHOW wal_level;        -- expected: replica
SHOW max_wal_senders;  -- expected: 10
\q
```

---

## STEP 11 — Configure pg_hba.conf

Use `SHOW data_directory` dynamically to avoid hardcoding the PostgreSQL 18 path:

```bash
podman exec -it pg-primary bash -c \
  "echo 'host replication replicator 0.0.0.0/0 md5' >> \$(psql -U postgres -tAc 'SHOW data_directory')/pg_hba.conf"

podman exec -it pg-primary bash -c \
  "tail -3 \$(psql -U postgres -tAc 'SHOW data_directory')/pg_hba.conf"
```

Reload config without full restart:

```bash
podman exec -it pg-primary psql -U postgres -c 'SELECT pg_reload_conf();'
```

Or restart the container:

```bash
podman restart pg-primary
sleep 10
podman ps
```

Test replication login:

```bash
podman exec -it pg-primary psql -U replicator -d postgres -c "SELECT 1;"
```

---

## STEP 12 — Take Base Backup

Use the plain container name as host — this is the key Podman difference from Docker Swarm:

```bash
podman exec -it pg-primary bash -c "
  pg_basebackup \
    -h pg-primary \
    -D /tmp/standby \
    -U replicator \
    -P \
    -R \
    --port=5432
"
```

Password when prompted: `replpass`

Expected: `23722/23722 kB (100%), 1/1 tablespace`

---

## STEP 13 — Copy Backup to Host

```bash
podman cp pg-primary:/tmp/standby/. ~/pg-podman-ha/standby_base/
ls -ltr ~/pg-podman-ha/standby_base/
```

---
## STEP 14 — Verify primary_conninfo

The `-R` flag writes connection info to `postgresql.auto.conf`, not `postgresql.conf`:

```bash
cat ~/pg-podman-ha/standby_base/postgresql.auto.conf
```

Expected output:
```
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=replicator password=replpass channel_binding=prefer host=''pg-primary'' port=5432 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
```

**The file is correct as written by `pg_basebackup -R` — no editing needed.**

The `host=''pg-primary''` syntax (doubled single-quotes) is valid PostgreSQL connection string escaping. Inside a single-quoted string, `''` is the escape sequence for a literal single quote. PostgreSQL reads this correctly as `host=pg-primary`.

**Do not run sed on this file.** The pattern `sed -i "s/host='[^']*'/host='pg-primary'/"` will match the first empty `''` between the doubled quotes and corrupt the line to `host='pg-primary'pg-primary''`.

If the host is genuinely wrong (e.g. it shows `localhost` or `127.0.0.1` instead of `pg-primary`), restore the file manually:

```bash
cat > ~/pg-podman-ha/standby_base/postgresql.auto.conf << 'EOF'
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=replicator password=replpass channel_binding=prefer host=''pg-primary'' port=5432 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
EOF
```

---

## STEP 15 — Copy Standby Data into Containers

In PostgreSQL 18 the data directory inside the container is `/var/lib/postgresql/18/docker/`:

```bash
podman cp ~/pg-podman-ha/standby_base/. pg-standby1:/var/lib/postgresql/18/docker/
podman cp ~/pg-podman-ha/standby_base/. pg-standby2:/var/lib/postgresql/18/docker/

podman exec pg-standby1 ls /var/lib/postgresql/18/docker/standby.signal
podman exec pg-standby2 ls /var/lib/postgresql/18/docker/standby.signal
```

Expected:
```
/var/lib/postgresql/18/docker/standby.signal
/var/lib/postgresql/18/docker/standby.signal
```

---

## STEP 16 — Restart Standby Containers

```bash
podman restart pg-standby1 pg-standby2
sleep 15
podman ps
```

---

## STEP 17 — Verify Streaming Replication

```bash
podman exec -it pg-primary psql -U postgres -c \
  "SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;"
```

Expected: two rows, both `state = streaming`:

```
 application_name | client_addr |   state   | sync_state
------------------+-------------+-----------+------------
 walreceiver      | 10.89.0.11  | streaming | async
 walreceiver      | 10.89.0.12  | streaming | async
(2 rows)
```

---

## STEP 18 — Test Replication End-to-End

Write on primary:

```bash
podman exec -it pg-primary psql -U postgres -c "
  CREATE TABLE test1 (id int, name text);
  INSERT INTO test1 VALUES (1, 'Clement');
"
```

Read from standbys:

```bash
podman exec -it pg-standby1 psql -U postgres -c "SELECT * FROM test1;"
podman exec -it pg-standby2 psql -U postgres -c "SELECT * FROM test1;"
```

Expected: `1 | Clement` on both.

Confirm standbys are read-only:

```bash
podman exec -it pg-standby1 psql -U postgres -c "INSERT INTO test1 VALUES (2, 'test');"
```

Expected: `ERROR: cannot execute INSERT in a read-only transaction`

---

## STEP 19 — Wire Auto-Restart via systemd

This replaces Docker Swarm's replica restart behaviour. systemd user units watch each container and restart it automatically if it dies.

```bash
mkdir -p ~/.config/systemd/user

podman generate systemd --new --name pg-primary  > ~/.config/systemd/user/container-pg-primary.service
podman generate systemd --new --name pg-standby1 > ~/.config/systemd/user/container-pg-standby1.service
podman generate systemd --new --name pg-standby2 > ~/.config/systemd/user/container-pg-standby2.service
```

Note: `DEPRECATED command` warning is harmless — the generated unit files work correctly in Podman 4.9.x. Quadlets are the modern replacement (see Future Improvements).

Enable and start:

```bash
systemctl --user daemon-reload
systemctl --user enable --now container-pg-primary.service
systemctl --user enable --now container-pg-standby1.service
systemctl --user enable --now container-pg-standby2.service
```

Verify all three are active:

```bash
systemctl --user status container-pg-primary.service --no-pager
systemctl --user status container-pg-standby1.service --no-pager
systemctl --user status container-pg-standby2.service --no-pager
```

Expected for each: `Active: active (running)`

Key lines to confirm in standby logs:
```
entering standby mode
consistent recovery state reached
started streaming WAL from primary on timeline 1
redo starts at 0/xxxxxxx
```

---

## STEP 20 — HA Failure Testing

### Test 1 — Kill Primary, Watch Auto-Restart

```bash
# Note current primary container
podman ps -f name=pg-primary

# Kill it
podman rm -f pg-primary

# Standbys still running
podman ps

# systemd restarts pg-primary automatically (~5-10 seconds)
podman ps

# Wait for standbys to reconnect (~15-20 seconds), then verify
podman exec -it pg-primary psql -U postgres -c \
  "SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;"
```

Expected timeline:
```
# Immediately — primary gone, standbys still up
# ~7s  — systemd recreates pg-primary container
# ~20s — both standbys show state = streaming again
```

### Test 2 — Kill Standby, Primary Unaffected

```bash
# Kill standby1
podman rm -f pg-standby1

# Primary keeps running, only one standby remains
podman exec -it pg-primary psql -U postgres -c \
  "SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;"

# systemd recreates standby1 automatically (~30s)
# Verify data survived on recovered standby
podman exec -it pg-standby1 psql -U postgres -c "SELECT * FROM test1;"
```

---

## STEP 21 — Useful Verification Commands

```bash
podman ps
podman ps -a
podman logs pg-primary
podman logs pg-standby1
podman inspect pg-primary
podman volume ls
podman network ls
podman network inspect pg-net
podman stats --no-stream
podman exec pg-primary psql -U postgres -tAc "SHOW data_directory"
systemctl --user status container-pg-primary.service --no-pager
```

---

## STEP 22 — Complete Cleanup

Run in this order — systemd units must be stopped before containers, and explicit removes are needed for named volumes and the network:

```bash
# 1. Stop and disable systemd units first
systemctl --user stop \
  container-pg-primary.service \
  container-pg-standby1.service \
  container-pg-standby2.service

systemctl --user disable \
  container-pg-primary.service \
  container-pg-standby1.service \
  container-pg-standby2.service

rm -f ~/.config/systemd/user/container-pg-*.service
systemctl --user daemon-reload

# 2. Stop and remove all containers
podman stop -a
podman rm -af

# 3. Force remove the image
podman rmi -f docker.io/library/postgres:18

# 4. Remove named volumes explicitly (volume prune skips named volumes)
podman volume rm \
  pg-podman-ha_pg-primary-data \
  pg-podman-ha_pg-standby1-data \
  pg-podman-ha_pg-standby2-data

# 5. Remove the network
podman network rm pg-net

# 6. Remove working directory
sudo rm -rf ~/pg-podman-ha

# 7. Verify clean state
podman ps -a && podman images && podman volume ls && podman network ls
```

Expected final state — only the default podman network remains:
```
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES

REPOSITORY  TAG  IMAGE ID  CREATED  SIZE

DRIVER  VOLUME NAME

NETWORK ID    NAME    DRIVER
xxxxxxxxxxxx  podman  bridge
```

---

## Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| `WARN: short name` on pull | No search registry configured | Use `docker.io/library/postgres:18` fully qualified |
| `pip3 install podman-compose` fails | Ubuntu 24.04 PEP 668 system Python protection | Use `sudo apt install podman-compose` instead |
| `podman-compose` not found after pipx install | PATH not reloaded | Open new terminal or run `source ~/.bashrc` |
| `WARN: "/" is not a shared mount` | WSL2 root filesystem is private mount | Cosmetic warning only — safe to ignore |
| `/proc/sys/kernel/unprivileged_userns_clone` missing | Modern WSL2 kernel — restriction does not exist | Normal — user namespaces always enabled, no action needed |
| `pg_basebackup` fails with connection error | Wrong hostname | Use plain `pg-primary`, not `pgcluster_pg-primary` |
| Container exits immediately, empty logs | Wrong mount path | Mount `/var/lib/postgresql` not `/var/lib/postgresql/data` |
| `image is in use by a container` on rmi | systemd restarted container after stop | Stop systemd units first, then `podman rm -af`, then `podman rmi -f` |
| Named volumes survive `volume prune` | Prune skips named volumes by design | Remove explicitly: `podman volume rm <name>` |
| `error getting current working directory` | Shell was inside a directory that was deleted | Harmless — run `cd ~` to recover |
| `DEPRECATED command` on generate systemd | Quadlets are the recommended modern approach | Warning only — generated unit files still work correctly |

---

## HA Test Results Summary

| Test | Primary Behaviour | Standby Behaviour | Data Loss | Recovery Time |
|---|---|---|---|---|
| Primary killed | systemd restarts new container | Lose connection, retry, reconnect automatically | None (named volume persists) | ~7s restart + ~20s standby reconnect |
| Standby killed | Unaffected, continues serving | systemd restarts new container, reconnects to primary | None (named volume persists) | ~30s full reconnect |

---

## Important HA Limitations of This PoC

- **No automatic failover** — if primary goes down, standbys do NOT auto-promote. systemd only restarts the same primary container. This is container-level HA, not database-level HA.
- **For automatic PostgreSQL failover** (promote standby to primary on failure), Patroni, repmgr, or pg_auto_failover is needed on top of this setup.
- **Single host** — all containers run on the same WSL2 machine. True HA requires multiple physical or virtual nodes.

---

## Docker Swarm vs Podman — Command Cheat Sheet

| Docker Swarm | Podman equivalent |
|---|---|
| `docker swarm init` | Not needed |
| `docker stack deploy -c file.yml name` | `podman-compose up -d` |
| `docker service ls` | `podman ps` |
| `docker service update --force svc` | `podman restart <container>` |
| `docker network create --driver overlay` | `podman network create` |
| `docker exec -it $(docker ps -q -f name=X)` | `podman exec -it X` (use name directly) |
| `docker cp src dest` | `podman cp src dest` |
| `docker logs $(docker ps -q -f name=X)` | `podman logs X` |
| `docker stack rm name` | Stop systemd units + `podman-compose down` + explicit volume/network rm |
| Service DNS: `pgcluster_pg-primary` | Container DNS: `pg-primary` |
| Volume name: `pgcluster_pg-primary-data` | Volume name: `pg-podman-ha_pg-primary-data` |
| Restart managed by Swarm replica controller | Restart managed by systemd user units |

---

## Future Improvements

- **Quadlets** — replace `podman generate systemd` (deprecated) with `.container` unit files managed natively by Podman. Cleaner, declarative, no separate generation step needed.
- **Patroni** — add automatic primary election and standby promotion on failure on top of this setup.
- **Multi-node Podman** — extend to multiple WSL2 or VM instances for true infrastructure-level HA.
