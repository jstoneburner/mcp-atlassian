# MCP Atlassian monthly maintenance (DroneDeploy fork)

**Audience:** Cursor / Claude Code agents and maintainers.

**Repo path:** `~/dev/repos/mcp-atlassian`  
**Origin:** `https://github.com/dronedeploy/mcp-atlassian`  
**Upstream:** `https://github.com/sooperset/mcp-atlassian` (`upstream` remote)

**Docker image (GAR):**  
`us-docker.pkg.dev/dronedeploy-code-delivery-0/docker-dronedeploy-us/mcp-atlassian`  
Tags: `:main`, `:latest`, `:<git-short-hash>`

**Extended playbook (security-ops):**  
`~/dev/security-ops/projects/docker/atlassian/MCP_ATLASSIAN_MAINTENANCE_PLAYBOOK.md`  
**Fork / GAR overview:**  
`~/dev/security-ops/projects/docker/atlassian/mcp-atlassian-fork-deployment.md`

---

## When to run this

- **Monthly** (or when asked): sync upstream, merge security/useful dependency updates, scan, rebuild image, refresh local Docker, verify Cursor MCP.
- **Trigger phrases:** "monthly mcp-atlassian maintenance", "sync mcp-atlassian upstream", "update mcp-atlassian docker", "push mcp-atlassian to GAR".

**Agent rule:** Execute all steps below yourself (including local Docker cleanup and verification). Do not ask the user to run cleanup commands unless Docker/gcloud is unavailable.

---

## Prerequisites

- Docker Desktop running (same daemon Cursor uses: `docker context show` → `default` or `desktop-linux`).
- `gcloud` authenticated: `gcloud auth login` and `gcloud auth configure-docker us-docker.pkg.dev --quiet`.
- Permission to push to `dronedeploy-code-delivery-0` / `docker-dronedeploy-us`.
- Git remotes: `origin` → `dronedeploy/mcp-atlassian`, `upstream` → `sooperset/mcp-atlassian`.

---

## 1. Upstream sync (selective)

```bash
cd ~/dev/repos/mcp-atlassian
git checkout main
git fetch upstream origin
```

**Compare:**

```bash
# Upstream commits we do not have (candidates to merge)
git log --oneline main..upstream/main

# Our fork-only commits (do not lose these)
git log --oneline upstream/main..main
```

- If `main..upstream/main` is **empty**, upstream is already merged; skip merge.
- If non-empty, review each commit. **Merge or cherry-pick** only:
  - Security fixes
  - Bug fixes that affect Jira/Confluence MCP tools we use
  - Dependency pins required for installs
- **Do not** blindly `git merge upstream/main` without reviewing. Skip upstream features we do not need.
- Prefer merging upstream on `main`, then `git push origin main`. Use feature branches only for large or risky changes.

---

## 2. Dependency and security updates

**Dependabot PRs** (`gh pr list --repo dronedeploy/mcp-atlassian`):

| Usually merge | Usually defer |
|---------------|----------------|
| urllib3, requests, cryptography, authlib, python-multipart, idna, lxml, pip | fastmcp major (e.g. 2.x → 3.x) |
| Other patches with clear CVE / advisory | Dev-only: pytest, uv, black |
| | Large minor bumps without security justification |

After merging security PRs, align **floors** in `pyproject.toml` if needed (e.g. `urllib3>=2.7.0`, `requests>=2.33.0`), then:

```bash
uv lock
uv sync --no-dev
```

**Verify installed versions (project venv, not global Python):**

```bash
.venv/bin/python -c "import urllib3, requests, authlib; print(urllib3.__version__, requests.__version__, authlib.__version__)"
```

**Tests:**

```bash
uv run pytest -q --ignore=tests/integration
```

Known flaky area: OAuth proxy unit tests may fail with `sqlite3.OperationalError` in some environments; confirm failures are pre-existing.

**Residual risk (document, do not “fix” in a monthly pass without a plan):**

- `fastmcp` 2.14.x pins block upgrading to fastmcp 3.x (fixes several CVEs).
- `starlette` 0.52.x may report advisories fixed only in 1.x; tied to fastmcp.

---

## 3. Commit and push code

```bash
git status
git add <files>
git commit -m "security(deps): ..."   # or fix(jira): / chore(deps): per Conventional Commits
git push origin main
```

DroneDeploy fork conventions: see `AGENTS.md` (use `uv` only, conventional commit scopes).

---

## 4. Build and push to Google Artifact Registry

From repo root:

```bash
./push-to-gar.sh
```

This builds **linux/amd64** and **linux/arm64**, tags `:main`, `:latest`, and `:<git-short-hash>`, and pushes to GAR.

**Confirm push:** note the image digest from the build output (e.g. `sha256:dc1aa96b...`).

---

## 5. Refresh local Docker (required on every MCP host)

Old containers keep running the previous image until removed. Run on the **same machine as Cursor** (or wherever `mcp-atlassian-wrapper` runs):

```bash
IMAGE="us-docker.pkg.dev/dronedeploy-code-delivery-0/docker-dronedeploy-us/mcp-atlassian"

# Stop and remove containers using any mcp-atlassian tag
for cid in $(docker ps -a -q --filter "ancestor=${IMAGE}:main"); do docker stop "$cid" 2>/dev/null; docker rm -f "$cid"; done
for cid in $(docker ps -a -q --filter "ancestor=${IMAGE}:latest"); do docker stop "$cid" 2>/dev/null; docker rm -f "$cid"; done
for cid in $(docker ps -a --format "{{.ID}} {{.Image}}" | grep "${IMAGE}" | awk '{print $1}'); do docker stop "$cid" 2>/dev/null; docker rm -f "$cid"; done

# Remove local images so pull is fresh
docker rmi -f "${IMAGE}:main" "${IMAGE}:latest" 2>/dev/null || true
docker images "${IMAGE}" -q | while read id; do docker rmi -f "$id"; done

# Pull and verify
gcloud auth configure-docker us-docker.pkg.dev --quiet
docker pull "${IMAGE}:main"
docker images "${IMAGE}" --format "{{.Repository}}:{{.Tag}} {{.ID}} {{.CreatedAt}}"
```

**Verify image contents (replace `EXPECTED` with `git rev-parse --short HEAD`):**

```bash
EXPECTED=$(git rev-parse --short HEAD)
docker run --rm --entrypoint /bin/sh "${IMAGE}:main" -c \
  'echo revision=$MCP_ATLASSIAN_BUILD_REVISION; python -c "import urllib3,requests; print(urllib3.__version__, requests.__version__)"'
# revision must equal EXPECTED
```

---

## 6. Update Cursor MCP

1. **Reload MCP** in Cursor (Command Palette → “MCP: Reload Servers”) **or restart Cursor**.
2. Wrapper (if used): `~/.local/bin/mcp-atlassian-wrapper` from  
   `~/dev/security-ops/projects/docker/atlassian/mcp-atlassian-wrapper.sh`  
   Image in MCP config: `.../mcp-atlassian:main` (see `~/dev/security-ops/playbooks/MCP_PLAYBOOK.md`).

**Verify via MCP tools:**

| Tool | Expected |
|------|----------|
| `get_server_version` | `revision` = current `git rev-parse --short HEAD` (e.g. `4172705`) |
| `jira_get_issue` | `SECENG-744`, fields `summary,status` only, `comment_limit: 0` (read-only test ticket) |

If `revision` is **stale** after step 5:

- Repeat step 5 (force stop all `mcp-atlassian` containers, `docker pull`).
- Reload MCP / restart Cursor again.
- Re-check `get_server_version`.

**Do not** use SECENG-742 or other production tickets for write tests.

---

## 7. Optional checks

- **GitHub Dependabot:** `https://github.com/dronedeploy/mcp-atlassian/security/dependabot`
- **Docker Scout** (requires `docker login`):  
  `docker scout cves us-docker.pkg.dev/dronedeploy-code-delivery-0/docker-dronedeploy-us/mcp-atlassian:main`
- **Cluster:** Processing clusters using the same image pull `:main` with `imagePullPolicy: Always` on next rollout (see `security-ops/projects/k8s/mcp-atlassian/`).

---

## Quick checklist (agents)

- [ ] `git fetch upstream` — reviewed `main..upstream/main`
- [ ] Merged security/useful deps; skipped risky major bumps (fastmcp 3.x)
- [ ] `uv lock` / tests run; changes committed and pushed to `origin main`
- [ ] `./push-to-gar.sh` succeeded
- [ ] Local containers stopped, old images removed, `docker pull` done
- [ ] Image `MCP_ATLASSIAN_BUILD_REVISION` matches `git rev-parse --short HEAD`
- [ ] Cursor MCP reloaded; `get_server_version.revision` matches
- [ ] `jira_get_issue` on SECENG-744 succeeds

---

## Reference: last known good baseline (May 2026)

- **Git:** `4172705` on `dronedeploy/mcp-atlassian` `main`
- **Image digest:** `sha256:dc1aa96bdd52f6d2c7b675a0af88820376eac2ad4fe0faec1ac23f9c81760cee`
- **Key deps in image:** urllib3 2.7.0, requests 2.33.0, authlib 1.6.12, starlette 0.52.1, fastmcp 2.14.5

Update this section when completing maintenance if the baseline changed.
