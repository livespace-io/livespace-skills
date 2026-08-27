# JIRA Cloud integration — auth and scope gotchas

Load this file only when the upstream third-party service is **JIRA Cloud**. A handful of non-obvious failure modes show up at build time; they're documented here so future scaffolds don't re-discover them.

1. **Token shape determines URL.** Atlassian has two API token shapes, both stored at `id.atlassian.com`:
   - **Classic (unscoped)** — Basic Auth with `email:token` against the site URL (`https://<site>.atlassian.net`). Acts as the full account.
   - **Scoped** — Basic Auth with `email:token` (same shape), but against the **OAuth gateway** URL (`https://api.atlassian.com/ex/jira/<cloudId>`). The site URL silently treats scoped tokens as *unauthenticated* (no 401, just empty results). The cloudId is discoverable via `GET <site>/_edge/tenant_info` (no auth required).
   Detect: if your token has scopes attached, route through `api.atlassian.com`. Wrong URL is the most common silent failure — manifests as 200 with empty payloads, not 401.

2. **Granular scope names don't follow endpoint names.** `read:jql:jira` exists in Atlassian's scope catalog but does **not** gate `POST /search/jql`. The actual granular scopes for `/search/jql` are `read:issue-details:jira`, `read:field.default-value:jira`, `read:field.option:jira`, `read:field:jira`, `read:group:jira`. Don't guess — check Atlassian's OpenAPI spec's `x-atlassian-oauth2-scopes` extension:
   ```bash
   curl -sSL https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-<group>/ \
     | grep -oE '"operationId":"<op>".{0,15000}' \
     | grep -oE '"x-atlassian-oauth2-scopes":\[[^]]+\]'
   ```
   The `"state":"Beta"` block lists the granular scopes (required for scoped tokens); `"state":"Current"` lists the classic compound scope.

3. **Scopes are immutable on existing tokens.** Atlassian's UI lets you view scopes on a saved token but the field is read-only. Changing the scope checkboxes has no effect on the issued secret. To change scopes: **revoke and recreate** the token. Failure mode: `/serverInfo` works (scope-less), every other endpoint returns `401 "Unauthorized; scope does not match"`.

4. **`/search/jql` rejects unbounded JQL.** Atlassian's new endpoint requires every JQL to have at least one bounded clause (`field = value`, date comparison, etc.) — pure `ORDER BY` is rejected with a locale-dependent 400 error. Always include a bound: a custom-field filter, a project clause, or a date floor like `updated >= -30d`.

5. **Custom issue-type avatars are auth-protected.** System avatars (built-in Bug, Task, Story icons) are served publicly; custom-uploaded avatars require authentication. URL shape: `/rest/api/<ver>/universal_avatar/view/type/issuetype/avatar/<id>?size=<size>`. Proxy these through the backend per the "rewrite auth-protected upstream asset URLs" backend rule in `SKILL.md`.
