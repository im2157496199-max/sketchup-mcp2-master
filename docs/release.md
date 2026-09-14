# Releasing a New Version

Step-by-step for the next PyPI/GitHub release. PyPI tokens live in `~/.pypirc` (chmod 600); `twine` reads them automatically.

## Breaking changes (v0.1.0)

- **Wire protocol: one-time handshake on connect.** Every TCP connection
  must now begin with a JSON-RPC `hello` request carrying
  `params.client_version`; the server replies with `server_version` and
  `client_id`. Per-request `client_version` / per-response
  `server_version` envelopes are **removed**. Old Python clients (any
  release prior to this one) lack the `hello` handshake entirely and
  the server rejects their first frame with JSON-RPC `-32600`
  (`"first method must be 'hello'"`); the client and the `.rbz` must
  be upgraded together.
- **Multi-client support.** The Ruby plugin now accepts N concurrent TCP
  clients; the previous "single-client-at-a-time" behavior is gone — a
  second concurrent connection no longer blocks until Python's
  `SKETCHUP_MCP_TIMEOUT` fires. The previous workaround (restart the
  plugin or temporarily disable the `sketchup` MCP server in Claude
  Code to run `smoke_check.py` alongside an attached MCP session) is no
  longer necessary.

## 0. Pre-flight

```bash
git fetch origin
git log --oneline origin/master..HEAD   # local-only commits ahead of remote
git status                              # tree should have no tracked-file modifications
```

If HEAD has diverged from `origin/master`, decide **rebase** vs **merge** before the bump commit.

## 1. Bump version in 6 places (must match)

- `pyproject.toml` — `version = "X.Y.Z"`
- `src/sketchup_mcp/__init__.py` — `__version__ = "X.Y.Z"`
- `src/sketchup_mcp/compat.py` — `MAX_RUBY = "X.Y.Z"` (and `MIN_RUBY` only if this release breaks wire/handler contract with the previous Ruby plugin)
- `mcp_for_sketchup/package.rb` — `VERSION = 'X.Y.Z'`
- `mcp_for_sketchup/mcp_for_sketchup.rb` — `ext.version = 'X.Y.Z'`
- `mcp_for_sketchup/mcp_for_sketchup/core/compat.rb` — `SERVER_VERSION = "X.Y.Z"` and `MAX_PYTHON = "X.Y.Z"` (and `MIN_PYTHON` only if this release breaks wire/handler contract with the previous Python client)

**MIN/MAX policy:** default to bumping only `MAX_*` to the new release; keep `MIN_*` pointing to the oldest counterpart this side still *accepts at the handshake*. Note that this is unilateral acceptance, not end-to-end interop: because each side's `MAX_*` is pinned to its own version (the two tests below), a pair only works when both versions are equal, whatever the floors say. `MIN_*` therefore controls which side reports the mismatch and what the error text claims — not which pairs can talk. Three invariant tests defend against typos and forgotten bumps:

* `test_min_le_max_invariant` (Python + Ruby) — range cannot be empty.
* `test_max_ruby_matches_python_version` (Python) — Python's view of Ruby max must equal current `CLIENT_VERSION` at release time.
* `test_max_python_matches_server_version` (Ruby) — Ruby's view of Python max must equal plugin `SERVER_VERSION` at release time.

**Contract break — floors bumped in v0.3.0 (2026-07-02; batches 1+2, branch `fix/deep-review-p2`):** `transform_component.position` switched from a relative offset to an absolute bbox-min target (`feat!`, commit `6b7d133`): an old/new client–server mix would pass the handshake but silently misplace geometry. Batch 2 widened the same break — new tool parameters (`name`, `limit`/`offset`/`response_format`), stricter validation (min dimensions 0.1 mm for cube / 1.0 mm for curved types, dovetail angle ≤ 60°, non-zero scale), and changed response shapes (`list/find_components` pagination envelope, `bbox_mm: null` for empty bounds, screenshot metadata block, `export` warning field). v0.3.0 bumps **both MIN floors to `0.3.0`** (`MIN_RUBY` Python-side, `MIN_PYTHON` Ruby-side) — from that release the handshake was exact-match `0.3.0`↔`0.3.0`, so an incompatible mix is rejected at the handshake instead of silently misbehaving. Call out the new semantics in the GitHub release notes. `0.3.1` is packaging and copy only, so neither floor moved and both 0.3.1 artifacts declare `0.3.0..0.3.1` — but that does not make a mixed pair work: each side's `MAX_*` tracks its own release, so an installed 0.3.0 plugin rejects a 0.3.1 client at the handshake, and a 0.3.0 client rejects a 0.3.1 plugin. Ship the Python package and the `.rbz` as a pair.

Run `uv lock` to refresh `uv.lock` with the new project version (otherwise the next `uv` call updates it post-release and you end up with a stray `chore: sync uv.lock` commit). Commit (`chore: bump to vX.Y.Z`) and push.

## 2. Pre-flight tests

```bash
uv run pytest tests/ -q          # Python — must be green
ruby test/run_all.rb             # Ruby — must be green
```

## 3. Build artifacts

```bash
rm -rf dist/ mcp_for_sketchup/*.rbz
uv build                                              # → dist/*.whl + dist/*.tar.gz
uvx twine check dist/*                                # validate metadata / README rendering
(cd mcp_for_sketchup && ruby package.rb)   # → mcp_for_sketchup_vX.Y.Z.rbz
```

`package.rb` needs the `rubyzip` gem: `gem install --user-install rubyzip`.

## 4. TestPyPI rehearsal

```bash
uvx twine upload --repository testpypi dist/*
```

Verify install in a fresh venv (the project's own `.venv` would conflict):

```bash
mkdir -p /tmp/verify && cd /tmp/verify && uv venv -q && \
  uv pip install -q --index-url https://test.pypi.org/simple/ \
    --extra-index-url https://pypi.org/simple/ \
    --index-strategy unsafe-best-match \
    sketchup-mcp2==X.Y.Z && \
  .venv/bin/python -c "import sketchup_mcp; print(sketchup_mcp.__version__)"
rm -rf /tmp/verify
```

`--extra-index-url` is required — TestPyPI doesn't host the `mcp` dependency.
`--index-strategy unsafe-best-match` is required because uv otherwise locks onto the first index that contains the package at all; once `sketchup-mcp2` exists on pypi.org, uv won't look at TestPyPI for the new version without this flag.

## 5. Production PyPI

```bash
uvx twine upload dist/*
```

**Warning:** PyPI versions are **immutable**. Once `X.Y.Z` is uploaded, it can never be re-uploaded — even after deletion. If something is broken post-upload, bump to `X.Y.(Z+1)`.

## 6. Git tag + GitHub Release

Attach the `.rbz` (see [§3](#3-build-artifacts)) plus the Python wheel/sdist. The `.rbz` must already be self-signed via the [Trimble signing service](https://extensions.sketchup.com/developer/sign-extension) — an unsigned extension is flagged as unidentified, and SketchUp blocks it outright under the strictest loading policy (*Identified Extensions Only*).

**The service hands back a different file from the one you upload.** It appends a `-signed` suffix to the name and encrypts every `.rb` under the extension folder to `.rbe`, adding `mcp_for_sketchup.susig`; the root loader and `settings.html` stay in the clear, and `main.rb`'s `LOAD_ORDER` names paths without an extension precisely so `Sketchup.require` picks up the `.rbe`. Attach **that** file. The unsuffixed artifact §3 produced is the unsigned one and must not be published:

```bash
git tag vX.Y.Z -m "Release X.Y.Z" && git push origin vX.Y.Z
gh release create vX.Y.Z \
  --title "vX.Y.Z" \
  --notes "..." \
  dist/sketchup_mcp2-X.Y.Z-py3-none-any.whl \
  dist/sketchup_mcp2-X.Y.Z.tar.gz \
  mcp_for_sketchup/mcp_for_sketchup_vX.Y.Z-signed.rbz
```

Release notes must call out anything a user upgrading in place would otherwise
discover the hard way. For `0.3.1`:

- `eval_ruby` now ships **enabled by default**; close the gate by unchecking
  **Enable Ruby evaluation** in `Plugins → MCP Server → Settings...`.
- Upgrading over an installation where the user had explicitly disabled
  `eval_ruby` leaves it disabled — the stored preference outranks the new
  default. Intended behaviour; say so, or it reads as a bug.
- Upgrading over an installation where the user **never opened Settings** does
  the opposite: with no stored preference the new default applies, so the gate
  opens. This hits everyone running a `-warehouse` build — published as a
  release asset for both v0.2.0 and v0.3.0 — where an absent preference
  previously resolved to *closed* through the build profile. No dialog is
  shown: `confirm_eval_enable` fires only on an off→on transition inside the
  Settings dialog, and an upgrade never passes through it. Spell this out; it
  is the one case a user cannot discover by reading their own settings.
- The Python package and the `.rbz` must be upgraded **together**: an installed
  0.3.0 plugin rejects a 0.3.1 client at the handshake (`-32001`), and a 0.3.0
  client rejects a 0.3.1 plugin (see [§1](#1-bump-version-in-6-places-must-match)).

## Notes

- `LICENSE` and `NOTICE` ship inside the wheel via `license-files` in `pyproject.toml` — no manual copying needed.
- After the first publish, swap the account-wide PyPI tokens in `~/.pypirc` for **project-scoped** ones (PyPI → Settings → API tokens → Scope: `Project: sketchup-mcp2`). Compromise of a scoped token only affects that project.
- **The Extension Warehouse is not a distribution channel for this project.** Trimble denied the v0.2.0 submission in August 2026 on policy grounds, not on fixable defects: they publish no externally developed MCP servers, reserving the catalogue for tools they build and secure themselves. Do not spend another two-month review cycle on it. GitHub Releases is the only channel — the `.rbz` still goes through the [Trimble signing service](https://extensions.sketchup.com/developer/sign-extension), which is a separate, self-serve flow with no review.
