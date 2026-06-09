# `archie-job-reverse-code-generator` ↔ `archie-client-worker` image alignment

**Updated:** 2026-05-08 — reflects the in-flight Dockerfile work on
`archie-client-worker` branch `feature/chat-integration`.

This doc captures what was decided about image strategy, what was actually
implemented, and the known shortcomings of the chosen approach.

## What was decided

1. **No separate `archie-chat-worker` image.** Chat-worker = the same
   `archie-client-worker` image jobs already use, switched on at runtime via
   env var (e.g. `CHAT_WORKER_MODE=true`). One image, one build pipeline.
2. **Don't touch `archie-job-reverse-code-generator`.** Its CI / Cloud Run Job
   pipeline is separate from `archie-client-worker`; convergence happens by
   modeling client-worker's Dockerfile on rcg's pattern, not by sharing a base.
3. **Convergence target = rcg's runtime layout** (`ubuntu:24.04` + same security
   upgrades + same git-lfs install + same Chrome install + same env vars), with
   chat-worker-relevant trimming.

## What was implemented (chat phase 1)

**`archie-client-worker/Dockerfile`** — two-stage:

### Builder (verbatim from the original `archie-client-worker` Dockerfile)

```
FROM python:3.12 AS builder
... apt: git libxml2; upgrade libxslt1.1 openssl libc-bin libc6
... pip 25.3 + ensurepip cleanup
... ssh-keyscan github.com
... pip install keyrings.google-artifactregistry-auth
... pip install --prefix=/install --mount=type=secret,id=google_credentials -r requirements.txt
... pip install --prefix=/install ast-grep-cli==0.36.5  ← only addition
```

The builder stays Debian-based (python:3.12) and uses `--prefix=/install` —
exactly as the repo had it before chat work started. The single addition is
`ast-grep-cli`, which lands at `/install/bin/ast-grep` alongside other deps.

### Runtime (rcg-modeled `ubuntu:24.04`, trimmed)

Mirrors rcg's runtime block-by-block:

| rcg block | Status in chat-worker |
|---|---|
| `FROM ubuntu:24.04` | ✅ |
| libpam + gnutls security upgrades | ✅ verbatim |
| `apt: software-properties-common, sudo, wget, ca-certs, curl, gnupg, lsb-release, file, bsdmainutils + python3.12 + python3.12-venv + python3.12-dev + python3-pip + python-is-python3 + git + xz-utils` | ⚠️ Most kept; **dropped** `software-properties-common`, `sudo`, `lsb-release`, `python3-pip` (each only needed by rcg's docker-repo / system-pip-install paths chat doesn't use) |
| `apt: bash, jq, procps, openssh-client` | ✅ added (rcg gets these transitively via other deps; we ask explicitly) |
| `apt: ripgrep, fd-find` + symlink `fd → fdfind` | ✅ chat-worker addition (not in rcg) |
| git-lfs 3.7.1 install (curl + tar + install.sh + system config) | ✅ verbatim |
| Node.js 20 + npm 11 + glob/brace-expansion/diff swaps | ❌ dropped (chat doesn't run JS) |
| `chrome-devtools-mcp@latest` global install | ❌ dropped (chat doesn't browse) |
| Google Chrome (signing key + apt repo + install) | ✅ verbatim (RUNNER_CAPABILITIES.CHROME path) |
| pip 25.3 system install + setuptools 70 upgrade | ❌ dropped (deps come from builder via `/install`) |
| `rm -rf PyJWT/jwt dist-packages` | ❌ dropped (no system pip install in runtime) |
| Docker Engine 28.x + containerd + compose | ❌ dropped (no DinD) |
| supervisor + iptables + fuse-overlayfs | ❌ dropped (no DinD) |
| `start.sh` dockerd bootstrap | ❌ dropped — `CMD ["python", "main.py"]` directly |
| ENV `DBUS_SESSION_BUS_ADDRESS=/dev/null` + `CHROME_DEVEL_SANDBOX=0` | ✅ verbatim |
| `WORKDIR /app` + `COPY . .` | ⚠️ split: `COPY main.py` + `COPY src/` (chowned to non-root user) |
| Runs as root | ❌ chat-worker uses non-root `appuser` (UID 1000) |

Cross-stage bridge:
```
COPY --from=builder /install /opt/python-deps
ENV PATH=/opt/python-deps/bin:$PATH
ENV PYTHONPATH=/opt/python-deps/lib/python3.12/site-packages:/app:/app/src/blitzy_worker
RUN ln -sf /usr/bin/python3.12 /usr/local/bin/python3.12  # plus python, python3
```

## Shortcomings of the implemented approach

This is honest documentation of fragility we accepted:

1. **Cross-Python-build bridge.** Builder uses Debian's python:3.12 (Python at `/usr/local/bin/python3.12`); runtime uses Ubuntu 24.04's apt python3.12 (Python at `/usr/bin/python3.12`). CPython 3.12.* ABI is stable in theory, but in practice they're different builds with potentially different OpenSSL linkages, optimization flags, etc. Wheels compiled in builder *should* run in runtime, but a wheel that links against a Debian-specific libssl version could break.

2. **Symlink hack to bridge interpreter paths.** `RUN ln -sf /usr/bin/python3.12 /usr/local/bin/python3.12` exists solely so pip entry-point scripts copied from `/install/bin/` (with shebang `#!/usr/local/bin/python3.12`) resolve in the runtime. It works, but it's a smell — anyone reading the Dockerfile has to mentally model the shebang dance. Subsequent edits could accidentally clobber the symlink.

3. **Non-standard site-packages location.** Deps live at `/opt/python-deps/lib/python3.12/site-packages` accessed via `PYTHONPATH`. If a future maintainer adds a `pip install <something>` in the runtime layer (legitimate quick fix), it goes to `/usr/local/lib/python3.12/dist-packages` (Ubuntu's default) — *not* to `/opt/python-deps`. Two parallel deps trees; debugging becomes confusing.

4. **Two distros to track for security advisories.** Builder is Debian (python:3.12 = Bookworm). Runtime is Ubuntu 24.04 (Noble). Vulnerability scanning has to look at both.

5. **Convergence with rcg is partial.** Runtime stage matches rcg apt-by-apt — that's the convergence we wanted. But the builder is `python:3.12` (Debian), nothing like rcg's `ubuntu:24.04`. So the build-time toolchain diverges. If we ever want a true single-base image (`archie-base-toolchain`) shared with rcg, the builder will need rebasing too.

## A better approach we considered and didn't take

**Ubuntu-everywhere with a venv.** The version I sketched two iterations earlier:

```dockerfile
FROM ubuntu:24.04 AS builder
RUN apt install python3.12 python3.12-venv ...
RUN python3.12 -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
RUN pip install -r requirements.txt
RUN pip install ast-grep-cli==0.36.5

FROM ubuntu:24.04 AS runtime
RUN apt install python3.12 ...
COPY --from=builder /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
```

Why it would have been cleaner:

- **Same Python build in both stages.** No cross-distro ABI risk; venv's symlinks resolve cleanly because the runtime has the exact Python the builder used to create the venv.
- **No symlink hack.** Venv is a Python-blessed pattern; no `/usr/local/bin/python3.12` synthesis required.
- **No PYTHONPATH gymnastics.** The venv is self-contained; activating it (via `PATH=/opt/venv/bin:$PATH`) makes `python` find its own `site-packages` automatically.
- **One distro to track for security.** Both stages on Ubuntu 24.04.
- **Tighter rcg convergence.** Builder and runtime both `FROM ubuntu:24.04` aligns with rcg's structure (rcg is single-stage but on the same base).

Why we didn't take it: the user (correctly) pointed out that the original
client-worker Dockerfile used `FROM python:3.12` for the builder, and
preserving that structure was a stated goal. The trade-off was preserving
the existing pattern vs. cleaner cross-stage semantics. We chose preserving.

## Recommendation for the next image refactor

When the team revisits image hygiene (e.g., extracting a shared
`archie-base-toolchain` as discussed in PLAN.md §3.5):

1. **Drop the python:3.12 builder.** Replace with `FROM ubuntu:24.04` + apt
   python3.12 + venv at `/opt/venv`. Same pattern in both stages.
2. **Delete the `/usr/local/bin/python3.12` symlinks.** No longer needed once
   the runtime has the same interpreter the builder used.
3. **Switch `/opt/python-deps` references back to `/opt/venv`.** Standard
   Python venv layout; `PATH` is sufficient (no `PYTHONPATH` override).
4. **At that point**, both client-worker and rcg are on `FROM ubuntu:24.04`
   with the same security-upgrade + apt-bootstrap pattern. Extracting a
   shared base image becomes a mechanical refactor.

This is documented here so the next refactor doesn't re-litigate the
shortcomings — they're known, accepted for now, and have a clear retirement
plan.

## Smoke test (current implementation)

```bash
cd /Users/m.makhzomi/Projects/archie-client-worker
DOCKER_BUILDKIT=1 docker build \
  --secret id=google_credentials,src=$SERVICE_ACCOUNT_KEY_PATH \
  -t archie-client-worker:ubuntu-test .

docker run --rm archie-client-worker:ubuntu-test bash -c "
  python --version &&
  python -c 'import blitzy_worker; print(\"worker import ok\")' &&
  ast-grep --version &&
  rg --version &&
  fd --version &&
  git --version &&
  git-lfs --version &&
  google-chrome --version
"
```

If `ast-grep` errors with `/usr/local/bin/python3.12: No such file or directory`,
the symlink wasn't applied — re-check the runtime stage's `ln -sf` block.
