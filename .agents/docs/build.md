# Build & tests

Out-of-source build is required (in-source is blocked).

```bash
mkdir -p build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/azeroth-server -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DSCRIPTS=static -DMODULES=static
make -j$(nproc) && make install
```

C++20 required (`CMAKE_CXX_STANDARD 20`). Useful flags: `BUILD_TESTING=ON` (Google Test), `NOPCH=1` (disable precompiled headers). Full set in `conf/dist/config.cmake`. `compile_commands.json` is exported automatically.

Tests (Google Test, in `src/test/`): configure `-DBUILD_TESTING=ON`, then `ctest` or `./src/test/unit_tests` from the build dir.

## Docker Compose stack

`docker-compose.yml` builds four targets from `apps/docker/Dockerfile`; `authserver`,
`worldserver` and `db-import` all derive from the `runtime` stage, which copies from `build`.

- **`dbimport` is not built by default.** The build stage declares `ARG CTOOLS_BUILD="none"`, and
  `dbimport` lives in `src/tools/`, not `src/server/apps/`. CMake therefore never configures it and
  the `db-import` stage fails with `"/azerothcore/env/dist/bin/dbimport": not found`. Nothing in
  `docker-compose.yml` or `.env` wires the arg, so pass `CTOOLS_BUILD: db-only` as a build arg.
  It must be **identical on all three services** — a differing value gives each its own build-stage
  cache key and recompiles the whole server per service.
- **Never `docker compose restart` worldserver.** Docker's default stop timeout is 10s, then
  SIGKILL. A clean worldserver shutdown flushes every online character and measured ~170s with 500
  playerbots, so `restart` kills it mid-save and players silently lose progress. Set
  `stop_grace_period` on every stateful service (worldserver, database) and use `stop` + `start`.
  Verify with `docker inspect -f '{{.State.ExitCode}}'` — 0 is clean, 137 is SIGKILL.
- **Module config lives on a bind mount** (`env/dist/etc`), so config edits need only a restart,
  not a rebuild. Module *code* under `modules/` is compiled into worldserver and does need one.

### After a Windows or WSL restart

The stack does **not** come back on its own, and two separate things are broken:

1. **Docker is stopped.** There is no systemd in this distro, so nothing restarts the
   daemon. Run `.agents/plans/wsl-docker-ollama-rig/start-docker.sh`, which also works
   around `/etc/init.d/docker` line 62 (`ulimit -Hn 524288` fails with EINVAL when the
   calling shell's *soft* limit is already higher, aborting before `dockerd` launches).
2. **The realm is flagged offline.** An ungraceful shutdown leaves
   `acore_auth.realmlist.flag = 3` (`REALM_FLAG_VERSION_MISMATCH | REALM_FLAG_OFFLINE`).
   The realm-list query excludes flagged realms, so authserver finds none, logs
   *"no realm address could be resolved"* and **exits 0** — a clean exit that restart-loops
   under `restart: unless-stopped` and looks nothing like a failure. Fix:

   ```sql
   UPDATE acore_auth.realmlist SET flag = 0 WHERE id = 1;
   ```

   AzerothCore prints this remedy itself, a few lines above the error most greps land on —
   read the whole block, not the last line.

Also check `netsh interface portproxy show v4tov4` against the current WSL IP
(`ip -4 addr show eth0`): the IP usually changes across a restart, and the forwarding rules
still point at the old one. It does not always change, so verify rather than assume.
