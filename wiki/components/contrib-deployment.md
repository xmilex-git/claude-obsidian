---
type: component
parent_module: "[[modules/contrib|contrib]]"
path: "contrib/cloud/, contrib/init.d/, contrib/bash/"
status: developing
purpose: "Contributor-maintained deployment and operational tooling: Docker + Kubernetes StatefulSet, SysV init script, bash tab-completion"
tags:
  - component
  - cubrid
  - deployment
  - docker
  - kubernetes
  - contrib
related:
  - "[[modules/contrib|contrib]]"
  - "[[sources/cubrid-contrib|cubrid-contrib]]"
  - "[[components/utility-binaries|utility-binaries]]"
  - "[[components/heartbeat|heartbeat]]"
created: 2026-04-23
updated: 2026-04-23
---

# Contrib Deployment & Operations Tooling

Three `contrib/` subdirectories cover the operational life of a CUBRID installation: cloud-native container deployment, Linux service management, and shell ergonomics.

## cloud/ — Docker + Kubernetes StatefulSet

`contrib/cloud/` is the most actively maintained part of `contrib/`, referenced by the project's CI documentation and the AGENTS.md top-level notes.

### Files

| File | Purpose |
|------|---------|
| `Dockerfile` | Multi-stage build: CentOS builder stage compiles CUBRID from git; runtime stage creates the deployable image |
| `build.sh` | Shell wrapper around `docker build` and `kubectl apply/delete` |
| `cubrid-statefulset.yaml` | Kubernetes StatefulSet + headless Service + LoadBalancer Service definitions |
| `README.md` | Full deployment guide with minikube setup walkthrough |

### Dockerfile Details

Two-stage build:

**Stage 1 (builder):** CentOS base; installs ant, bison, cmake, flex, gcc/g++, JDK 1.8, make, ncurses, sysstat, systemtap-sdt-devel; clones `https://github.com/$GIT_FORK/cubrid.git`; builds with CMake (`-DCMAKE_BUILD_TYPE`, `-DWITH_SOURCES`, `-DUNIT_TESTS=OFF`).

**Stage 2 (runtime):** Fresh CentOS; copies compiled install from stage 1.

Build arguments:

| ARG | Default | Description |
|-----|---------|-------------|
| `BUILD_TYPE` | `Debug` | CMake build type (`Debug`, `RelWithDebInfo`, `Release`) |
| `MAKEFLAGS` | `-j4` | Parallel make jobs |
| `GIT_FORK` | `CUBRID` | GitHub org — clones `https://github.com/CUBRID/cubrid.git` |
| `GIT_REVISION` | `develop` | Branch or commit SHA |
| `INSTALL_SOURCES` | `ON` | Whether to embed source tree at `/opt/cubrid/src/` (for GDB) |
| `UID` | `985` | UID/GID for the `cubrid` OS user |
| `DB_NAME` | `cubdb` | Database name created at entrypoint |
| `DB_LOCALE` | `en_US` | Database locale |
| `DB_VOLUME_SIZE` | `128M` | Initial volume size |
| `CUBRID_COMPONENTS` | `ALL` | Which components to start (`BROKER`, `SERVER`, `MASTER`, `SLAVE`, `HA`, `ALL`) |

### Kubernetes Layout

`cubrid-statefulset.yaml` defines:
- A headless `ClusterIP: None` Service (DNS-stable pod names: `cubrid-0.cubrid`, `cubrid-1.cubrid`)
- A `LoadBalancer` Service (`cubrid-read`) on port 33000 — connect read clients here
- A `StatefulSet` — pods named `cubrid-0`, `cubrid-1`, …; external volume mount at `/var/lib/cubrid` (persists core dumps and DB data across restarts)

HA note: `cubrid-0` is the expected master; write clients should connect directly to `cubrid-0.cubrid:33000`. Read clients use the LoadBalancer Service.

### build.sh Usage

```bash
# Build debug image from develop branch
./build.sh --build-type=Debug --git-fork=CUBRID --git-revision=develop build-docker-image

# Deploy to minikube cluster
./build.sh --image=cubrid/cubrid:CUBRID-develop kubernetes-apply

# Tear down
./build.sh kubernetes-delete
```

### Debugging Inside a Pod

Because `INSTALL_SOURCES=ON` is the default, GDB finds source files automatically:

```bash
kubectl exec -it cubrid-0 -- /bin/bash
gdb /opt/cubrid/bin/cub_server
```

Core files land in `/var/lib/cubrid/<core-file>` (externally mounted, persistent).

## init.d/ — SysV Init Script

`contrib/init.d/cubrid` is a standard SysV-style init script for Linux systems using `/etc/init.d/` (RHEL/CentOS 6 and earlier, or systemd compat mode).

Key behaviors:
- Calls `cubrid service start/stop/status` as the `cubrid` OS user (via `su -` when run as root)
- `check_config()` parses `cubrid.conf` for `ha_mode`; if HA is enabled (`yes`, `on`, or `role-change`) the script skips the operation (HA mode requires manual operator control)
- Lock file at `/var/lock/subsys/cubrid` (root-only path)
- `chkconfig` header: runlevels 2–5, start priority 98, stop priority 05

> [!note] Systems using systemd should prefer a proper `.service` unit file. The init.d script can run under systemd's SysV compatibility layer but lacks `Type=forking` metadata, which may cause systemd to misreport service status.

## bash/ — Tab Completion

`contrib/bash/` provides a bash tab-completion script for the `cubrid` CLI (`cubrid service`, `cubrid server`, `cubrid broker`, `cubrid createdb`, etc.). Source into shell via `.bashrc`:

```bash
source /path/to/contrib/bash/cubrid_bash_completion
```

No README was found in `contrib/bash/` at inventory time. The completion script name is `cubrid_bash_completion` (inferred from directory contents).

## Related

- [[components/utility-binaries|utility-binaries]] — the `cubrid` service front-end that init.d and build.sh invoke
- [[components/heartbeat|heartbeat]] — HA heartbeat process; `check_config()` in init.d skips when HA is active
- [[components/broker-impl|broker-impl]] — one of the components started by `CUBRID_COMPONENTS=ALL`
- [[components/cub-master-main|cub-master-main]] — `cub_master` process started at entrypoint
- [[modules/contrib|contrib]] — parent module page
