# vaultwarden-bh

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) releases with **"Share via link"** turned on.

Bitwarden's own clients already contain "Share via link" (Bitwarden calls it *temporary item sharing*,
feature flag `pm-34203-temporary-item-sharing`). In a vault item's ⋮ menu it opens a form: email
addresses, an expiry (1 hour to 30 days) and *One time share*. It then copies a link. Under the hood the
link is a Send of a new type, **Item** (`type: 2`): the item is encrypted in the client with the
Send's key, and that key lives only in the link's `#` fragment, so the server never sees it.

Upstream Vaultwarden does not support it yet (no Item type, the flag isn't served). This fork adds
both. The official browser extension and web vault are used unchanged.

**No email verification, on purpose.** Bitwarden makes the recipient prove their email with a one-time
code. The owner of this fork asked for a plain link: whoever has the link can open it, like a text
Send. The clients still require at least one email address in the form; here it is only a label in
the owner's *Active links* list. The recipient never receives it, no mail is sent, and no code is
asked for.

## Branches and versions

- `bh` (default): an upstream release tag with our changes merged on top. `bh/UPSTREAM` names that tag.
- `sync/X.Y.Z`: a pull request that merges upstream `X.Y.Z` into `bh`.
- Remote `upstream` = `https://github.com/dani-garcia/vaultwarden.git`. Upstream tags have no `v`
  (`1.37.5`). This repository never builds upstream `main`.
- Images: `ghcr.io/datcal/vaultwarden-bh:<upstream>-bh.<CI run number>` (e.g. `1.37.5-bh.3`) and
  `:latest`, linux/amd64 only. Every image also gets a GitHub release with the same name.

## What we change in upstream code

Keep it this small. Every change is marked with a `bh` comment.

| File | Change |
|---|---|
| `src/db/models/send.rs` | `SendType::Item = 2`; `BH_ITEM_EMAILS_KEY`; `to_json` returns `data` for Item Sends, `emails`, and `authType` Email when emails are stored; `to_json_access` returns `data` and never the emails. |
| `src/db/models/mod.rs` | Re-exports `BH_ITEM_EMAILS_KEY`. |
| `src/api/core/sends.rs` | `SendData.data`; `bh_item_send_data` (needs `metadata`, stores the emails label); Item branch in `create_send` and `update_send_from_data`; `emails` refused only for non-Item Sends; `post_access` counts Item opens (one-time links close); `bh_tests`. |
| `src/api/core/mod.rs` | `/api/config` serves `pm-34203-temporary-item-sharing: true`. |
| `src/config.rs` | The flag is in `SUPPORTED_FEATURE_FLAGS`. |
| `.github/workflows/*` | Upstream's workflows are deleted: this repository has none of their secrets, and they must not run here. Only `bh-*.yml` stays. |
| `bh/`, `.github/workflows/bh-*.yml` | Ours. |

`git diff "$(cat bh/UPSTREAM)" bh -- src` shows exactly this.

The emails are stored inside the Send's existing `data` JSON (key `bhEmails`), so there is **no
database migration**: the database stays readable by upstream Vaultwarden. Going back to an upstream
image only makes existing Item Sends unusable until they expire.

### The wire format we follow

From Bitwarden's server (`src/Api/Tools/Models/`) and SDK (`bitwarden-api-api` models), as of
November 2026:

- Request (`POST /api/sends`, `PUT /api/sends/{id}`): `type: 2`, `authType: 0`, `emails: "a@x,b@y"`,
  `data: { encryptionVersion: 1, data: "<EncString>", metadata: { itemId, folderName,
  collectionNames, organizationName, creationDate, revisionDate } }`, plus the usual Send fields.
- Response (`send`, also in `/api/sync`): the same `data` object, `emails`, `authType`.
- Access (`POST /api/sends/access` with the `send_access` token): `type: 2`, `data`.

## Syncing a new upstream release

A scheduled agent runs this. A person can too.

1. `git fetch upstream --tags`. The newest release is
   `git tag -l --sort=-v:refname | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | head -1`.
2. **Open sync PR:** if a `sync/…` pull request is already open, check its CI.
   - CI failed: fix the cause on that branch and push.
   - CI passed or is still running: leave it; the owner merges it.
   - Either way, stop here.
3. **Up to date:** if the newest release equals `bh/UPSTREAM`, go to step 8 (client check) and stop.
4. **Start a sync:**
   - `git switch -c sync/X.Y.Z origin/bh`
   - `git merge --no-ff X.Y.Z -m "Merge upstream X.Y.Z"`
   - Resolve conflicts: keep upstream's code and re-apply our changes as in the table above, minimal
     and marked `bh`.
   - Delete every workflow the merge brings back or adds: `git rm` anything in `.github/workflows/`
     not named `bh-*` (a change upstream made to one of them shows as a modify/delete conflict;
     resolve it by deleting).
   - **If upstream now supports Item Sends itself** (`Item = 2` in `src/db/models/send.rs`, or the
     flag in `src/api/core/mod.rs`): keep the link working **without email verification** and say
     in the pull request which of our changes became unnecessary. Don't redesign.
5. **Update the marker:** write `X.Y.Z` to `bh/UPSTREAM` and commit.
6. **Test:**
   - `cargo fmt --all -- --check`
   - `cargo clippy --features sqlite -- -D warnings`
   - `cargo test --features sqlite`

   Fix what fails. If the environment cannot build in time, say so in the pull request and let CI decide.
7. **Open the pull request:**
   - Push `sync/X.Y.Z` and open a pull request into `bh` titled `Sync with upstream X.Y.Z`.
   - In the body: upstream's release notes in short, the conflicts and how you resolved them, the
     test results, and the client check below.
   - Never push to `bh`, never merge, never tag or release: the owner approves every merge.
8. **Client check** (every run, also when up to date): compare the newest Bitwarden code with
   *The wire format we follow* above:
   - `bitwarden/server` `main`: `src/Api/Tools/Models/Request/SendRequestModel.cs`,
     `Response/SendResponseModel.cs`, `Response/SendAccessResponseModel.cs`, `SendDataModel.cs`,
     `SendItemMetadataModel.cs`.
   - `bitwarden/clients`, newest `browser-v*` tag: `libs/tools/share/` and the flag in
     `libs/common/src/enums/feature-flag.enum.ts`.

   If a field, the flag name or the access flow changed, fix it (in the open sync pull request, or a
   new `fix/client-YYYY-MM-DD` pull request into `bh`) and update this section. If nothing changed,
   say so in your summary. If the flag is gone because the feature is generally available, nothing
   breaks; note it.

Never add a domain name, address, token or anything about the owner's servers to this repository.

## CI: `.github/workflows/bh-build.yml`

- **Pull request into `bh`:** format, clippy and tests, then builds the image (not pushed).
- **Push to `bh`** (a merged pull request): the same checks, then builds and pushes
  `ghcr.io/datcal/vaultwarden-bh:<version>` and `:latest`, and creates the GitHub release `<version>`.
- It also turns off any other workflow GitHub still knows about, in case one slipped back in.

## Running it

Use the image instead of `vaultwarden/server` in your compose file. Settings are upstream's. The owner's
server picks up each new release on its own; that setup lives outside this repository.
