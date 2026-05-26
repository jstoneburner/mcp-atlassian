# MCP Atlassian (DroneDeploy fork)

@AGENTS.md

## Agent: monthly maintenance

When the user asks for **monthly maintenance**, **upstream sync**, **GAR/docker update**, or **refresh mcp-atlassian** for this repo, follow **[MAINTENANCE.md](./MAINTENANCE.md)** end to end before reporting done.

You must:

1. Sync with `upstream` (`sooperset/mcp-atlassian`) selectively (security + useful fixes only).
2. Merge or apply security dependency updates; defer **fastmcp 3.x** without a dedicated upgrade plan.
3. Run tests, commit, and `git push origin main`.
4. Run **`./push-to-gar.sh`** to publish to Google Artifact Registry.
5. **Force local Docker refresh** (stop containers, remove old images, `docker pull`) on the MCP host.
6. **Reload Cursor MCP** (or restart Cursor) and verify **`get_server_version`** `revision` matches `git rev-parse --short HEAD`.
7. Smoke-test **`jira_get_issue`** on **SECENG-744** (read-only).

Do not ask the user to run Docker cleanup steps unless the environment blocks you.

**Image:** `us-docker.pkg.dev/dronedeploy-code-delivery-0/docker-dronedeploy-us/mcp-atlassian:main`
