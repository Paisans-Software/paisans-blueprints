# Welcoming new members: the catalog side, Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The `mbin` blueprint provides the `forum` role and declares the engagement app's Mbin agent client in its `.env`; the `synapse` blueprint auto-joins every new account to the engagement app's community room.

**Architecture:** Both changes read the engagement consumer's `Values` and `Secrets` through the toolkit's `.Consumers "<role>"`, added by paisans-cli PR #585. Neither blueprint computes anything: the alias, client id and username come from the toolkit.

**Tech Stack:** Paisans blueprint YAML (`paisans.blueprint/v3`), Go `text/template`, CEL for provided values.

**Spec:** paisans-cli `docs/specs/2026-10-29-engagement-welcome.md`; contract paisans-cli `docs/superpowers/plans/2026-10-29-engagement-welcome-contract.md`, section *Catalog*.

## Global Constraints

- Scopes rendered exactly: `agent:magazine:subscribe agent:magazine:create read write moderate:magazine_admin:update`.
- Variable names exactly: `MBIN_AGENT_CLIENT_ID`, `MBIN_AGENT_CLIENT_SECRET`, `MBIN_AGENT_CLIENT_SCOPES`, `MBIN_AGENT_USERNAME`, `MBIN_AGENT_CONTACT_EMAIL`.
- `forum` offers `api_port: "'8080'"`, nothing else.
- Synapse renders `auto_join_rooms: ["#<alias>:<server_name>"]` and `autocreate_auto_join_rooms: false`, only when an engagement consumer has `community_room_alias`.
- Both blueprints that read `.Values` carry `requires_toolkit: ">= 0.3.0-rc.20"`, the first toolkit release after rc.19 (the newest tag today). Confirm the number against the release that actually carries PR #585 before tagging the catalog. No release or tag is cut here.
- Mbin's image track is unchanged. The agent client needs an Mbin fork release carrying `mbin:agent:sync`; none exists yet, so an older image ignores the five variables.
- Comments cite source by tag and line, as the files already do. No rejected alternatives in comments or README.

## Review Focus

- **No engagement app:** synapse renders unchanged, and mbin renders only the "no engagement app" comment. Checked by rendering a deployment without the engagement app.
- **Engagement app without Matrix:** synapse renders no `auto_join_rooms` (the consumer has no alias), checked by the render where the toolkit omits the alias.
- **Two engagement apps on one forum:** Mbin has one declared agent client, so only the first consumer with a complete client is rendered; the comment says so.
- **Incomplete consumer values:** the agent block renders only when the client id, the username and the shared secret are all non-empty; otherwise the comment renders.
- **Room missing when an account registers:** Synapse logs and skips the join; registration still succeeds (`synapse/handlers/register.py:608-613` at v1.164.1). Stated in the comment so nobody reads a missing join as a failed signup.
- **Secret with shell or env specials:** rendered through `envquote`, as every other secret in the file.

## Synapse finding (read before Task 2)

Read in Synapse at tag v1.160.0, the version the synapse blueprint's comments pin, and unchanged at v1.164.1: MAS provisioning (`synapse/rest/synapse/mas/users.py:155-161`) creates an account with `registration_handler.register_user(..., by_admin=True)`. `register_user` runs `_auto_join_rooms` for every non-guest account unless consent is required at registration (`synapse/handlers/register.py:384-396`). With `autocreate_auto_join_rooms` false it goes to `_join_rooms` (`:615-643`), which resolves the alias and joins the account directly when the room's join rule is `public` (`:557-605`). The engagement bot creates the room public. A room that does not exist yet is logged and skipped, and the account is still created (`:608-613`). MAS-provisioned accounts are auto-joined.

## Verification

The catalog has no tests of its own. Each task is checked by rendering paisans-cli's golden test deployment (`internal/render/testdata/deployment.yaml`, `TestGoldenTree`) on a copy of the paisans-cli branch carrying #585, with that copy's test catalog `mbin` and `synapse` replaced by this branch's. Every task compares against a render of the unchanged catalog, and the test failures on the copy must not grow. The same render without the engagement app covers the deployment that has none. `paisans blueprint lint` runs on each changed blueprint.

---

### Task 1: mbin provides `forum` and declares the agent client

**Files:**
- Modify: `blueprints/mbin/blueprint.yaml` (add `requires_toolkit`, `provides`)
- Modify: `blueprints/mbin/.env.secret.tmpl` (append the agent block)
- Modify: `README.md` (the `provides` sentence names `forum` and `api_port`)

**Interfaces:**
- Consumes: `.Consumers "forum"` entries with `.Values.mbin_agent_client_id`, `.Values.mbin_agent_username`, `.Secrets.mbin_agent_client_secret`; `.Setting "contact_email" .SMTPSender`.
- Produces: `forum.api_port` = `8080`, which the toolkit renders into `ENGAGEMENT_MBIN_API_URL`.

- [ ] **Step 1: State the expected render.** Against the unchanged render, exactly: the talk app's `.env` gains the five `MBIN_AGENT_*` lines (client id `engagement-agent`, username the digest app's `bot_localpart`, secret the fixture's `fixture-not-a-secret-digest-mbin-agent`, the scopes above, the contact email the file already renders as `KBIN_CONTACT_EMAIL`); the digest app's `.env` gains the toolkit's three Mbin agent lines with `api_port` 8080. Nothing else.
- [ ] **Step 2: Render before the change.** Without `forum` the digest renders no Mbin lines and the talk app no agent block. Not the state of step 1.
- [ ] **Step 3: Implement.** `provides: [{role: forum, values: {api_port: "'8080'"}}]` with a comment (who links, why the mesh port); `requires_toolkit` with the reason. In `.env.secret.tmpl`, after the OIDC block, a block guarded by the first `forum` consumer whose client id, username and shared secret are all non-empty, rendering the five variables, each value through `envquote`; else a comment line saying no engagement app links here. Comment: what `mbin:agent:sync` does with them, that only one agent client is declared, and that an image without the command ignores them.
- [ ] **Step 4: Render.** Exactly the changes of step 1. Without the engagement app: only the "no engagement app" comment.
- [ ] **Step 5: Lint.** `paisans blueprint lint blueprints/mbin`: PASS. No new test failures on the copy.
- [ ] **Step 6: Commit.** `feat: mbin provides forum and declares the engagement agent client`

### Task 2: synapse auto-joins new accounts to the community room

**Files:**
- Modify: `blueprints/synapse/blueprint.yaml:8` (`requires_toolkit` to `>= 0.3.0-rc.20`, reason extended to `.Values`)
- Modify: `blueprints/synapse/homeserver.yaml.secret.tmpl` (after `auto_accept_invites`)

**Interfaces:**
- Consumes: `.Consumers "matrix-homeserver"` entries with `.Values.community_room_alias`; `.ServerName`.

- [ ] **Step 1: State the expected render.** Against Task 1's render, the chat app's `homeserver.yaml` gains
  ```yaml
  auto_join_rooms:
    - "#<alias>:<server_name>"
  autocreate_auto_join_rooms: false
  ```
  with the fixture's alias and server name. Nothing else.
- [ ] **Step 2: Implement.** Range the consumers, render one entry per alias (`quote (printf "#%s:%s" ...)`), both keys only when at least one alias exists. Comment cites the Synapse finding above by file and line at the pinned tag, says the bot creates the room public and the account joins it directly, and that a room not yet created is skipped and the account still created.
- [ ] **Step 3: Render.** Exactly the change of step 1; without the engagement app, synapse is unchanged. `paisans blueprint lint blueprints/synapse`: PASS.
- [ ] **Step 4: Commit.** `feat: synapse joins new accounts to the engagement app's community room`

### Task 3: Pull request

- [ ] Push `feat/engagement-welcome`, open a PR into `main` with the Generated with Claude Code footer. The body states that the `requires_toolkit` floor must match the first toolkit release carrying paisans-cli #585 before any catalog tag, that no Mbin release carries the agent client yet, and the Synapse finding. Do not merge, tag or release.
