# Cluster cross-file findings

#### `CSCAN-cluster-notify-tag-taint-unvalidated`
- **Finding ID** — `CSCAN-cluster-notify-tag-taint-unvalidated`
- **Severity** — Low
- **Confidence** — high
- **Files involved**
  - `release_notifications.yml:30-34` (cluster file — the taint-passing side)
  - `.github/workflows/notify.exs:6,14,36,45,48,64-66` (sibling sink invoked by the cluster file; **no per-file report exists for it**)
- **Finding shape** — Taint across files (attacker-influenceable data parsed/passed in file A reaches outbound-request sinks in file B with no re-validation at the boundary).
- **Evidence**
  - `release_notifications.yml:34` runs `elixir .github/workflows/notify.exs "$GITHUB_REF_NAME"`, forwarding the release tag into the script. The per-file reviewer of `release_notifications.yml` verified only the *shell* boundary is safe (double-quoted argv) and explicitly deferred the rest: *"notify.exs is out of scope and is where the real risk lives … Whether the tag name or fetched release body is sanitized there is not verifiable from this file and must be audited in notify.exs directly."* That audit was never performed (the prior `CROSS_FILE.md` lists it as an **Unresolved** ledger row and "the single highest-value follow-up").
  - `notify.exs:6` `[tag] = System.argv()` binds the value with **no validation**, then interpolates it unvalidated into: a `Req.get!` URL path (`:14` `…/releases/tags/#{tag}`), the Postmark **`HtmlBody`** and **`Subject`** of an email sent with `ELIXIR_LANG_ANN_TOKEN` (`:36`,`:45`,`:48`,`:58`), and the Elixir Forum post **`title`**/**`raw`** sent with `ELIXIR_FORUM_TOKEN` under the official `"Elixir"` username (`:64-66`,`:77`). The forum `raw` (`:66`) *also* interpolates `release["body"]`, which `notify.exs` fetches fresh from `api.github.com` (`:14`) — untrusted-at-parse content that crosses into the same credentialed sink with no sanitization.
  - Confirmed by grep: `notify.exs` contains **no** `System.cmd`/`:os.cmd`/backtick/`Code.eval`/`binary_to_term` sink — so the threat model's "interpolates … into URLs **and shell**" is inaccurate; the real sinks are fixed-host HTTPS URLs and JSON bodies, which bounds this to Low. All hosts are `https://` (satisfies HTTPS_ENDPOINTS_ONLY) and tokens come from `System.fetch_env!` (satisfies SECRETS_FROM_ENV_ONLY), so the residual is input-discipline, not transport/secret exposure.
- **What the per-file pass missed** — No single-file reviewer could close this: the `release_notifications.yml` reviewer proved only the shell hand-off is safe and punted the sink audit to `notify.exs`, while `notify.exs` has no per-file report at all and, read alone, cannot establish that its `System.argv()` value is the attacker-influenceable `$GITHUB_REF_NAME` with no upstream validation — that fact only exists at the seam between the two files.
- **Suggested fix** — Validate `tag` at the top of `notify.exs` before any interpolation and treat the API-sourced body as untrusted, e.g.:
  ```elixir
  [tag] = System.argv()
  unless tag =~ ~r/\Av?\d+\.\d+(\.\d+)?(-[\w.]+)?\z/ do
    raise "refusing to notify for unexpected tag: #{inspect(tag)}"
  end
  ```
  (and scope the forum/email bodies to trusted fields rather than forwarding raw `release["body"]`/`tag` verbatim). This is defense-in-depth: the trigger requires repo-write access to publish a release, so the reachable attacker is a privileged tag-pusher/maintainer, not the primary fork-PR outsider — hence Low.
