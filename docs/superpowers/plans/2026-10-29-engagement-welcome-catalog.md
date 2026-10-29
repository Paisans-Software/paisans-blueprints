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
- Both blueprints that read `.Values` carry `requires_toolkit: ">= 0.3.0-rc.19"`, the first toolkit release after rc.18 (the newest tag today). Confirm the number against the release that actually carries PR #585 before tagging the catalog. No release or tag is cut here.
- Mbin's image track is unchanged. The agent client needs an Mbin fork release carrying `mbin:agent:sync`; none exists yet, so an older image ignores the five variables.
- Comments cite source by tag and line, as the files already do. No rejected alternatives in comments or README.

## Review Focus

- **No engagement app:** neither file changes, byte for byte. Checked by the `no-digest` render.
- **Engagement app without Matrix:** synapse renders no `auto_join_rooms` (the consumer has no alias), checked by the render where the toolkit omits the alias.
- **Two engagement apps on one forum:** Mbin has one declared agent client, so only the first consumer with a client id is rendered; the comment says so.
- **Room missing when an account registers:** Synapse logs and skips the join; registration still succeeds (`synapse/handlers/register.py:606-612` at v1.164.1). Stated in the comment so nobody reads a missing join as a failed signup.
- **Secret with shell or env specials:** rendered through `envquote`, as every other secret in the file.

## Synapse finding (read before Task 2)

At tag v1.164.1, MAS provisioning (`synapse/rest/synapse/mas/users.py:155-161`) creates an account with `registration_handler.register_user(..., by_admin=True)`. `register_user` runs `_auto_join_rooms` for every non-guest account unless consent is required at registration (`synapse/handlers/register.py:384-396`). With `autocreate_auto_join_rooms` false it goes straight to `_join_rooms` (`:631-643`), which resolves the alias and joins directly when the room's join rule is `public` (`:553-577`, `:595-605`). The engagement bot creates the room public, so no `auto_join_user_id` is needed. **Auto-join works for MAS-provisioned accounts; no invite fallback is needed.**

## Test harness (scratch, never committed)

As in paisans-cli `docs/superpowers/plans/2026-10-27-engagement-matrix-catalog.md`, *Test harness*, with
`TK=/Users/wash/Developer/paisans.community/src/paisans-stack/.worktrees/engagement-welcome-toolkit` and
`CAT=/Users/wash/Developer/paisans.community/src/blueprints/.worktrees/engagement-welcome`. `H` is a scratchpad directory. `render.sh` copies TK, replaces its test catalog with CAT, runs `go test ./internal/render -run '^TestGoldenTree$' -update`, and copies the goldens out.

- [ ] `sh $H/render.sh $H/base` on the unchanged catalog: PASS.
- [ ] Record `go test ./internal/blueprints/ ./internal/render/ -count=1` failures on the copy as `$H/base-fails.txt`.

---

### Task 1: mbin provides `forum` and declares the agent client

**Files:**
- Modify: `blueprints/mbin/blueprint.yaml` (add `requires_toolkit`, `provides`)
- Modify: `blueprints/mbin/.env.secret.tmpl` (append the agent block)
- Modify: `README.md` (the `provides` sentence names `forum` and `api_port`)

**Interfaces:**
- Consumes: `.Consumers "forum"` entries with `.Values.mbin_agent_client_id`, `.Values.mbin_agent_username`, `.Secrets.mbin_agent_client_secret`; `.Setting "contact_email" .SMTPSender`.
- Produces: `forum.api_port` = `8080`, which the toolkit renders into `ENGAGEMENT_MBIN_API_URL`.

- [ ] **Step 1: State the expected render.** `diff -ru $H/base $H/t1` shows exactly: the talk app's `.env` gains the five `MBIN_AGENT_*` lines (client id `engagement-agent`, username the digest app's `bot_localpart`, secret the fixture's `fixture-not-a-secret-digest-mbin-agent`, the scopes above, the contact email the file already renders as `KBIN_CONTACT_EMAIL`); the digest app's `.env` gains the toolkit's three Mbin agent lines with `api_port` 8080. Nothing else.
- [ ] **Step 2: Run before the change.** `sh $H/render.sh $H/t1`: the toolkit copy's own test catalog already provides `forum`, so CAT without it must render the digest without the Mbin lines. Not the state of step 1.
- [ ] **Step 3: Implement.** `provides: [{role: forum, values: {api_port: "'8080'"}}]` with a comment (who links, why the mesh port); `requires_toolkit` with the reason. In `.env.secret.tmpl`, after the OIDC block, a block guarded by the first `forum` consumer whose `Values` has `mbin_agent_client_id`, rendering the five variables, each value through `envquote`; else a comment line saying no engagement app links here. Comment: what `mbin:agent:sync` does with them, that only one agent client is declared, and that an image without the command ignores them.
- [ ] **Step 4: Run.** `sh $H/render.sh $H/t1` and the diff of step 1: exactly that. `sh $H/render.sh $H/t1n no-digest` and `diff -ru $H/base-n $H/t1n` (base rendered no-digest first): only the "no engagement app" comment.
- [ ] **Step 5: Lint.** `$H/paisans blueprint lint $CAT/blueprints/mbin`: PASS. Test failures on the copy equal `$H/base-fails.txt`.
- [ ] **Step 6: Commit.** `feat: mbin provides forum and declares the engagement agent client`

### Task 2: synapse auto-joins new accounts to the community room

**Files:**
- Modify: `blueprints/synapse/blueprint.yaml:8` (`requires_toolkit` to `>= 0.3.0-rc.19`, reason extended to `.Values`)
- Modify: `blueprints/synapse/homeserver.yaml.secret.tmpl` (after `auto_accept_invites`)

**Interfaces:**
- Consumes: `.Consumers "matrix-homeserver"` entries with `.Values.community_room_alias`; `.ServerName`.

- [ ] **Step 1: State the expected render.** `diff -ru $H/t1 $H/t2`: the chat app's `homeserver.yaml` gains
  ```yaml
  auto_join_rooms:
    - "#<alias>:<server_name>"
  autocreate_auto_join_rooms: false
  ```
  with the fixture's alias and server name. Nothing else.
- [ ] **Step 2: Implement.** Range the consumers, render one entry per alias (`quote (printf "#%s:%s" ...)`), both keys only when at least one alias exists. Comment cites the Synapse finding above by file and line, says the bot creates the room public so the join needs no invite, and that a room not yet created is skipped without failing the registration.
- [ ] **Step 3: Run.** `sh $H/render.sh $H/t2` and the diff of step 1; `no-digest` render: synapse unchanged from base. Lint `blueprints/synapse`: PASS.
- [ ] **Step 4: Commit.** `feat: synapse joins new accounts to the engagement app's community room`

### Task 3: Pull request

- [ ] Push `feat/engagement-welcome`, open a PR into `main` with the Generated with Claude Code footer. The body states the `requires_toolkit` floor to confirm, that no Mbin release carries the agent client yet, and the Synapse finding. Do not merge, tag or release.
