# Codex dispatch prompt — Add IP Access Management tools to mcp-sendgrid-server

You are working on a TypeScript MCP server at `~/justinritchie-mcp-servers/sendgrid-mcp/` (cloned from upstream `iPraBhu/mcp-sendgrid-server`). Your job: add four new tools that expose SendGrid's IP Access Management API endpoints. The user (Justin Ritchie) currently gets constant "IP access alert" emails from SendGrid every time an unknown IP tries to authenticate, and wants Claude to be able to inspect and manage the IP whitelist via MCP instead of logging into SendGrid's web UI every time.

## What to add

Four new MCP tools, mapping to SendGrid v3 API endpoints:

| Tool name | HTTP method | Endpoint | Purpose |
|---|---|---|---|
| `sendgrid_list_ip_access_activity` | GET | `/v3/access_settings/activity` | List recent IP access attempts (THE feed that generates the alert emails) |
| `sendgrid_list_ip_whitelist` | GET | `/v3/access_settings/whitelist` | List currently whitelisted IPs |
| `sendgrid_add_ip_to_whitelist` | POST | `/v3/access_settings/whitelist` | Add one or more IPs to whitelist (write — gated by approval token) |
| `sendgrid_remove_ip_from_whitelist` | DELETE | `/v3/access_settings/whitelist/{rule_id}` | Remove a whitelisted IP by rule ID (write — gated by approval token) |

API reference: https://www.twilio.com/docs/sendgrid/api-reference/ip-access-management

## How to do it

1. **Fork the upstream repo** to `justinritchie/mcp-sendgrid-server`:
   ```bash
   cd ~/justinritchie-mcp-servers/sendgrid-mcp
   gh repo fork --remote --remote-name origin
   # The fork's `origin` will replace the upstream. Rename old origin to upstream:
   git remote rename origin upstream || true
   gh repo fork --remote --remote-name origin   # now origin = justinritchie's fork
   ```
   (If the remote setup is awkward — you can manually add: `git remote rename origin upstream && git remote add origin https://github.com/justinritchie/mcp-sendgrid-server.git`.)

2. **Create a feature branch** `feat/ip-access-management`.

3. **Read the existing codebase** to understand the patterns:
   - `src/server.ts` (or wherever tools are registered) — how tools register their metadata
   - One existing simple read tool (e.g., `sendgrid_get_account_summary`) for the GET pattern
   - One existing write tool (e.g., `sendgrid_delete_bounce`) for the write-with-approval-token pattern
   - The HTTP client wrapper (probably `src/api/sendgrid-client.ts` or similar) for how API calls are made

4. **Add the four tools** following the codebase's existing conventions:
   - Same file structure as other tools (one tool per file, or grouped — match what's there)
   - Same Zod schema validation patterns for input
   - Same response shapes (with `planWarning` field if relevant — check if `/v3/access_settings/*` endpoints have any plan gating)
   - Read tools should be allowed in `analytics` mode AND `full` mode
   - Write tools should be gated behind `SENDGRID_READ_ONLY=false`, `SENDGRID_WRITES_ENABLED=true`, AND a runtime `approval_token` matching `SENDGRID_WRITE_APPROVAL_TOKEN` (mirror the existing write-tool pattern exactly)
   - Update the README's tool table with a new "IP Access Management" section

5. **Update API key permissions documentation** in the README to mention IP Access Management read/write needs.

6. **Run `npm run build` and `npm test`** — make sure everything compiles and existing tests still pass.

7. **Commit with a clean message**:
   ```
   feat: add IP Access Management tools

   Adds four new tools for inspecting and managing the SendGrid IP whitelist:
   - sendgrid_list_ip_access_activity (GET /v3/access_settings/activity)
   - sendgrid_list_ip_whitelist (GET /v3/access_settings/whitelist)
   - sendgrid_add_ip_to_whitelist (POST /v3/access_settings/whitelist)
   - sendgrid_remove_ip_from_whitelist (DELETE /v3/access_settings/whitelist/{rule_id})

   Read tools (list_*) work in both full and analytics modes.
   Write tools (add/remove) require SENDGRID_READ_ONLY=false +
   SENDGRID_WRITES_ENABLED=true + runtime approval_token, matching the
   existing pattern for suppression-write tools.

   Closes the most common pain point with the existing toolset: investigating
   the IP access alert emails SendGrid sends when an unknown IP tries to
   authenticate.
   ```

8. **Push the branch to the fork** (`git push -u origin feat/ip-access-management`).

9. **Optionally open an upstream PR** to `iPraBhu/mcp-sendgrid-server` with the same body. Use:
   ```bash
   gh pr create --repo iPraBhu/mcp-sendgrid-server \
     --base main --head justinritchie:feat/ip-access-management \
     --title "feat: add IP Access Management tools" \
     --body-file <body file>
   ```

10. **Report back**: list the URL of the fork branch, the URL of the upstream PR (if opened), the file paths you added/modified, and confirm `npm run build` succeeded.

## Constraints

- **Don't change tool calling conventions.** Match the codebase's existing patterns for tool registration, input schema, response shape, and error handling exactly. The upstream maintainer will judge the PR partly on whether it feels native to their codebase.
- **Don't add new dependencies.** Use the existing HTTP client and Zod setup.
- **Don't bump the major version.** This is an additive minor change — bump 1.0.x to 1.1.0 if there's version logic.
- **Don't commit the .env or any credentials.** The user has SENDGRID_API_KEY in the env; never log it or commit it.
- **Mark all read tools as available in `analytics` mode** in addition to `full` mode (matches how stats and email activity tools work).
- **The `approval_token` field is REQUIRED on write tools** — mirror the exact validation logic from existing write tools.

## What "done" looks like

- Fork exists at https://github.com/justinritchie/mcp-sendgrid-server
- Feature branch `feat/ip-access-management` pushed to fork
- `npm run build` succeeds, `npm test` passes
- README updated with IP Access Management section in the tool table
- (Optional) Upstream PR opened
- A summary report from you with: branch URL, PR URL (if opened), files changed, test results

## Working directory

`~/justinritchie-mcp-servers/sendgrid-mcp/`

## Time budget

This should take 30-60 minutes. If you hit a blocker after 90 minutes, stop and report back with what's stuck rather than burning time. Common blockers and how to handle:

- **Endpoint shape unclear**: read https://www.twilio.com/docs/sendgrid/api-reference/ip-access-management directly via WebFetch and adapt.
- **Tool registration pattern unclear**: read 2-3 existing tools first; don't guess.
- **Tests failing on unrelated paths**: skip with a comment; report which tests in the summary.

Go.
