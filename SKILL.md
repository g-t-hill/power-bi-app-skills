---
name: powerbi-app-links
description: Use when you want details from Power BI Apps via the REST API — listing Apps, their Reports/Dashboards, and direct URLs into them, and optionally best-effort Audience data via an undocumented endpoint.
---

# Power BI App Links

Fetches Power BI App metadata (Apps, their Reports, Dashboards, and direct `webUrl`
links) via the official Power BI REST API, using a service principal (client
credentials). Optionally, if the user explicitly wants Audience data too, can
attempt a best-effort pull via an undocumented internal endpoint (see below) —
clearly flagged as unsupported. Outputs either an inline markdown table in the
conversation, or a markdown/CSV/Excel file, depending on what's asked for.

## What the official Power BI REST API can and can't give you

- **Apps**: `GET https://api.powerbi.com/v1.0/myorg/apps` — id, name,
  description, publisher.
- **Reports in an App**: `GET https://api.powerbi.com/v1.0/myorg/apps/{appId}/reports`
  — each report includes `webUrl`, a direct link to open it inside the App.
- **Dashboards in an App**: `GET https://api.powerbi.com/v1.0/myorg/apps/{appId}/dashboards`
  — similarly includes `webUrl`.
- **A single report within an App**: `GET https://api.powerbi.com/v1.0/myorg/apps/{appId}/reports/{reportId}`.
- **Sections** (the navigation groupings within an App's content list): NOT
  exposed by the REST API, documented or undocumented. Microsoft support has
  confirmed on the Fabric community forum that there is currently no way —
  official or otherwise — to retrieve an App's section/navigation structure
  programmatically. Don't attempt to scrape this; just say plainly it isn't
  available and, if useful, suggest maintaining it as a manually-updated
  reference note, or filing a Fabric Ideas request.
- **Audiences** (App audience definitions/membership): NOT exposed by the
  documented REST API. `GetAppUsersAsAdmin` only returns the flattened union
  of all users across all audiences, with no per-audience breakdown. See the
  best-effort undocumented path below if the user explicitly asks for
  audience data anyway.

## Authentication (service principal / client credentials) — for Apps/Reports/Dashboards

Requires an Entra ID app registration set up as a Power BI service principal, with:

- Power BI tenant setting "Allow service principals to use Power BI APIs"
  enabled (or scoped to a security group containing the SP) — requires Fabric
  tenant admin rights to configure.
- API permission: `App.Read.All` (application permission), admin-consented.
- The service principal must also be added as a member/admin on the specific
  Workspaces/Apps it needs to read, OR the tenant setting needs to allow read
  access more broadly — Power BI App content visibility for service
  principals can be workspace-scoped.

Credentials expected as environment variables (never hardcode or print secret values):

- `POWERBI_TENANT_ID`
- `POWERBI_CLIENT_ID`
- `POWERBI_CLIENT_SECRET`

Token acquisition (client credentials grant against Entra ID):

```bash
curl -s -X POST "https://login.microsoftonline.com/${POWERBI_TENANT_ID}/oauth2/v2.0/token" \
  -d "client_id=${POWERBI_CLIENT_ID}" \
  -d "client_secret=${POWERBI_CLIENT_SECRET}" \
  -d "scope=https://analysis.windows.net/powerbi/api/.default" \
  -d "grant_type=client_credentials"
```

This returns an `access_token` — use it as a Bearer token on all subsequent
Power BI REST calls. Tokens expire (typically 1 hour); re-fetch if a call
returns 401.

If any of the three env vars are missing, stop and tell the user which ones
are needed and how to set them (don't try to prompt for the secret value
inline — that's sensitive).

## Optional best-effort path: Audiences via undocumented endpoint

Only attempt this if the user explicitly asks for Audience data (don't reach
for it by default — it's unsupported and can break without notice). It
cannot use the service-principal token above; it needs a **browser-session
bearer token**, which the user has to obtain and supply themselves — there's
no way to get this non-interactively.

- Endpoint: `GET https://wabi-{region}-l-primary-redirect.analysis.windows.net/metadata/appmodel/apps/{WorkspaceID}?requestDataType=7`
  - `{region}` is the Power BI tenant's region code (e.g. `north-europe`,
    `west-europe`) — ask the user if unknown, or infer from the redirect
    Power BI itself uses (visible in the browser's network tab).
  - `{WorkspaceID}` is the workspace ID backing the App (not the App ID) —
    get this from the App's settings page or by matching the App to its
    source workspace via `GET /v1.0/myorg/groups`.
- Auth: Bearer token copied from an active powerbi.com browser session (open
  the App in Chrome, open DevTools → Network tab, find a request to
  `wabi-*.analysis.windows.net`, and copy its `Authorization` header value).
  Have the user pass this in as an environment variable (e.g.
  `POWERBI_BROWSER_TOKEN`) rather than pasting it directly into chat, since
  it's a live credential.
- Rate limit yourself: add a ~2.5 second delay between calls to avoid throttling.
- Treat the response shape as unstable — parse defensively, and if the
  response doesn't look like audience data, say so rather than guessing at a
  schema.

When using this path, always tell the user plainly in the output that
Audience data came from an unsupported/undocumented endpoint and could stop
working at any time without notice from Microsoft.

## Steps

1. Check for `POWERBI_TENANT_ID`, `POWERBI_CLIENT_ID`, `POWERBI_CLIENT_SECRET`
   in the environment. If missing, explain what's needed and stop.
2. Get an access token via the client-credentials call above.
3. Call `GET /v1.0/myorg/apps` to list Apps. If the user named a specific
   App, filter/match by name (case-insensitive substring match is fine —
   confirm if ambiguous).
4. For each relevant App, call `GET /v1.0/myorg/apps/{appId}/reports` (and
   `/dashboards` if asked) to get report/dashboard names and `webUrl`.
5. Build a markdown table: columns App, Report/Dashboard, Direct URL.
6. If the user asked about Sections, state plainly that no API — documented
   or undocumented — exposes this; don't fabricate or infer it.
7. If the user asked about Audiences, follow the best-effort undocumented
   path above (only if they've supplied `POWERBI_BROWSER_TOKEN` or are
   willing to obtain it); otherwise note the same limitation as Sections.
8. Deliver output:
   - Default (no file requested): reply inline with the markdown table in
     the conversation.
   - If a file/export is requested: write a `.md` file (or `.csv`/`.xlsx`
     for a spreadsheet) and send it.

## Notes

- Never print or log the client secret, the service-principal access token,
  or the browser bearer token.
- If a call returns 403 on the official API, the most likely cause is the
  service principal isn't a member of the Workspace backing that App, or the
  tenant setting restricting SP API access — mention this as the likely
  cause rather than a generic error.
- Rate limits: the Power BI REST API has throttling; if listing many Apps,
  add small delays or batch sensibly rather than firing all requests at once.
