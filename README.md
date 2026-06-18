# rust-gem-release

> **This is a reusable GitHub Actions *workflow*, not an action.** You call it with
> `uses:` at the **job** level (`jobs.<id>.uses:`), never as a step. It declares
> `on: workflow_call`, fans out a `strategy.matrix`, and spans two runner OSes
> (`ubuntu-24.04` + `macos-26`) with `needs` edges and per-job `permissions` — none of
> which a composite/JS/Docker action can do. The repo is `scientist-labs/rust-gem-release`;
> see [Repo name](#repo-name-reusable-workflow-not-an-action) for naming and versioning.

A single reusable workflow that turns a Rust-backed Ruby gem (Magnus / rb-sys /
rake-compiler) into a full set of **precompiled, install-without-a-toolchain gems** on
every tag, plus a **source gem** as the universal fallthrough:

| Platform | How it's built | Who it serves |
| --- | --- | --- |
| `arm64-darwin` | **Natively** on `macos-26` (Apple Silicon), one `.bundle` per Ruby ABI | every `arm64-darwin-NN` macOS user (generic platform; RubyGems does not fall back across darwin majors) |
| `x86_64-linux` | `oxidize-rb/cross-gem` on `ubuntu-24.04` | glibc amd64 |
| `aarch64-linux` | **Cross-compiled** from the `x86_64` `ubuntu-24.04` host via `cross-gem` | glibc arm64 |
| source (`ruby`) | `gem build` on `ubuntu-24.04`, pushed **first**, **creates the GitHub Release** | any ABI/platform outside the matrix (compiles Rust on install) |

Publishing to RubyGems is **off by default** and a clean no-op without a token — a new
gem can dry-run the entire 4-target matrix before it owns a RubyGems API key. A caller
can also expose a **`workflow_dispatch`** trigger (with a `publish` boolean, default
false) to run that full-matrix **dry-run on demand from a branch** — every leg builds,
but no GitHub Release is created and nothing is pushed (see
[Publishing](#publishing)).

The canonical consumer is **[red-candle](https://github.com/scientist-labs/red-candle)**
(`red-candle` 1.8.0 shipped all four platforms from this workflow). Its full caller lives
at [`examples/red-candle-release.yml`](examples/red-candle-release.yml).

---

## 2-minute quickstart

A plain CPU-only gem (e.g. `parsekit`, `tokenkit`, `spellkit`) needs ~6 lines. Create
`.github/workflows/release.yml` **in the consumer repo**:

```yaml
name: Release

on:
  push:
    tags:
      - "[0-9]+.[0-9]+.[0-9]+"
      - "[0-9]+.[0-9]+.[0-9]+.*"     # prereleases: 1.2.0.rc1
      - "v[0-9]+.[0-9]+.[0-9]+"
      - "v[0-9]+.[0-9]+.[0-9]+.*"

jobs:
  release:
    permissions:
      contents: write              # REQUIRED — the workflow can only reduce, never raise, this
    uses: scientist-labs/rust-gem-release/.github/workflows/release.yml@0.10.0
    with:
      gem-name: parsekit
      version-command: ruby -r./lib/parsekit/version -e 'print Parsekit::VERSION'
      publish: true                # false (or a workflow_dispatch dry-run) = build only, no RubyGems push; see Publishing
    secrets:
      rubygems-api-key: ${{ secrets.RUBYGEMS_API_KEY }}
```

That's it for a gem where `gem-name == gemspec basename == extension dir` (true for
`parsekit`/`tokenkit`/`spellkit`/`clusterkit`/`lancelot`/`phrasekit`/`fastsheet`).

> **`version-command` is the one input you almost always override** and the single most
> likely silent break for a new adopter. Its default is red-candle's exact command
> (`ruby -r./lib/candle/version -e 'print Candle::VERSION'`); every other gem must point
> it at its own version constant. The `prepare` job runs this command and **fails the run
> fast** unless the v-stripped pushed tag equals its output.

> Before the precompiled darwin/linux gems will actually *load*, the consumer also needs
> two small code changes — see [Consumer prerequisites](#consumer-prerequisites). Until
> then the source gem still works everywhere; the precompiled gems publish but won't
> `require`.

---

## Inputs

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `gem-name` | string | **yes** | — | RubyGems package **name**. The only required input. Drives every push/attach glob: `gem push <gem-name>-*.gem`, `<gem-name>-*-<darwin-platform>.gem`, `gh release upload <gem-name>-*.gem`. Keep it distinct from the gemspec basename and `ext-name` when they differ (`red-candle` ≠ ext `candle`; `indradb-ruby` ≠ ext `indradb`). |
| `ext-name` | string | no | `""` → `gem-name` | Compiled extension / `lib/` subdir name. Artifact lands at `lib/<ext-name>/<ext-name>.bundle` and the ABI subdir `lib/<ext-name>/<minor>/<ext-name>.bundle`; the gemspec packs `Dir['lib/<ext-name>/*/<ext-name>.bundle']`. Literal default is the **empty string** (a `workflow_call` input cannot default to another input); `prepare` resolves empty → `gem-name` and exposes the result as an output. Only aliasing gems set it (`red-candle`→`candle`, `indradb-ruby`→`indradb`). |
| `gemspec` | string | no | `""` → `<ext-name>.gemspec` | Gemspec **file** passed to `gem build` in both the source-gem and darwin-package jobs. Literal default is the empty string; `prepare` resolves empty → `<resolved-ext-name>.gemspec` and **asserts the file exists**, emitting a `::error::` with the exact override to add if not. `red-candle`'s file is `candle.gemspec` (tracks ext-name, served by the default once `ext-name: candle` is set). `indradb-ruby` must set this explicitly (`indradb-ruby.gemspec`). |
| `version-command` | string | no | `ruby -r./lib/candle/version -e 'print Candle::VERSION'` | Shell command (run at repo root in `prepare`) that prints the bare version to stdout. `prepare` guards that the v-stripped pushed tag equals it. The default is red-candle's exact command, so **every other consumer must override** (e.g. `ruby -r./lib/parsekit/version -e 'print Parsekit::VERSION'`). Kept as one full command so a gem whose version lives somewhere unusual is still served. |
| `ruby-versions` | string | no | `3.1,3.2,3.3,3.4,4.0` | Comma-separated Ruby ABI set compiled into every fat gem (floor = gemspec `required_ruby_version` 3.1; 3.3 = rx prod; 4.0 = GA). **Single source of truth:** `prepare` derives all three forms from this one CSV — the CSV verbatim for cross-gem's `ruby-versions` on the linux legs, a JSON array for the darwin build matrix, and a space-list for the darwin per-ABI verify loop — so they cannot drift. A higher-floor gem can trim it (e.g. `3.3,3.4,4.0`) to cut darwin matrix minutes. |
| `compile-command` | string | no | `bundle exec rake compile` | Native macOS compile entrypoint on `darwin-build` (one `.bundle` per ABI) — the rake-compiler/rb_sys common case. Override only for a different build entrypoint. The **linux legs do not use this** (cross-gem owns the linux build invocation). Runner labels are deliberately not exposed. |
| `platform-gem-env` | string | no | `RUST_GEM_PLATFORM` | Name of the env var the consumer's gemspec reads to enter its precompiled-platform branch (set `spec.platform`, clear `spec.extensions`, pack the bundles). `darwin-package` exports `ENV[<this>]=<darwin-platform>` before `gem build`. red-candle's gemspec reads `RED_CANDLE_PLATFORM_GEM`, so red-candle **must override** to that; **new adopters write their gemspec branch to read the generic default** `RUST_GEM_PLATFORM` and pass nothing. Inert until the consumer adds the gemspec branch ([prerequisite #2](#consumer-prerequisites)). |
| `darwin-platform` | string | no | `arm64-darwin` | Darwin platform string for the fat gem and its push/attach/verify globs. **Never intel darwin** — the generic `arm64-darwin` is what serves every `arm64-darwin-NN` user (RubyGems does not fall back across darwin majors). Exposed only so a hypothetical future target could differ; `arm64-darwin` is the only value red-candle ships. |
| `x86_64-cargo-config` | string | no | `""` (no-op) | **Optional** raw `.cargo/config.toml` text written at the repo root on the **`x86_64-linux` leg only**, *before* cross-gem runs (cross-gem bind-mounts the repo, so cargo reads it). Empty default makes the write a no-op (correct for every surveyed consumer). red-candle sets the `aws-lc-sys` gcc-95189 workaround `[env]\nAWS_LC_SYS_CMAKE_BUILDER = "1"`. The x86_64-only scoping is **hardcoded** (the gcc panic is host==target specific); only the config *text* is the input. When `linux-cargo-config` is also set, this **overwrites** the base on the x86_64 leg (last-writer-wins, not a TOML merge). |
| `linux-cargo-config` | string | no | `""` (no-op) | **Optional** raw `.cargo/config.toml` text written on **both** linux legs (x86_64 **and** aarch64) before cross-gem. The base layer for `[env]` both legs need (uniform `BINDGEN_EXTRA_CLANG_ARGS`, jobserver flags, a `PROTOC`/`CC` override pointing at a binary the rb-sys-dock image **already ships**). The per-leg `x86_64-cargo-config` / `aarch64-cargo-config` **overwrite** this on their own leg. **Cannot apt-install** — cross-gem@v1.4.4 runs the build inside a fixed rb-sys-dock image with no in-container install hook, so an `[env]` is only useful when its referenced tools are present (clang/llvm/cmake/cross-sysroot **yes**; protoc/gfortran **no**). |
| `aarch64-cargo-config` | string | no | `""` (no-op) | **Optional** raw `.cargo/config.toml` text written on the **`aarch64-linux` leg only** (mirror of `x86_64-cargo-config` for the cross leg). For aarch64-cross `[env]` such as `CC`/`CXX`/`BINDGEN_EXTRA_CLANG_ARGS` for a bindgen/C++ consumer (RocksDB). The rb-sys aarch64 image **already** sets `LIBCLANG_PATH`, `BINDGEN_EXTRA_CLANG_ARGS_aarch64_unknown_linux_gnu=--sysroot=…`, and the cross toolchain — so this **augments**, it does not bootstrap. Overwrites `linux-cargo-config` on the aarch64 leg. Same no-apt limitation. |
| `darwin-pre-build-command` | string | no | `""` (skip) | **Optional** shell run on each `darwin-build` leg **before** `compile-command`. `macos-26` ships Homebrew, so a prost/protoc (`brew install protobuf`) or bundled-C consumer installs its native system dep here. Empty default skips it. **Darwin only** — the linux analogue is `linux-cross-image-repo` (below): the rb-sys-dock image has no in-container install in cross-gem@v1.4.4, so a linux system package the image lacks is added by pre-seeding a deps-enriched cross image, not by an in-leg command. |
| `linux-cross-image-repo` | string | no | `""` (no-op) | **Optional** GHCR repo of a **deps-enriched** rb-sys cross image to pre-seed onto the runner before cross-gem. cross-gem→`rb-sys-dock` computes the image name `rbsys/<platform>:<tag>` and only pulls it when absent locally. When this is set, each enabled linux leg runs `docker pull <repo>:<platform>` then `docker tag <repo>:<platform> rbsys/<platform>:<linux-cross-image-tag>`, so rb-sys-dock finds the image **locally** and **skips its pull**, building against our image (which carries protoc / OpenBLAS+LAPACK+gfortran / leptonica+tesseract). The image MUST be `FROM rbsys/<platform>:<tag>` (preserves the whole cross toolchain) and **PUBLIC**. The `:<tag>` is load-bearing — it must equal the `rb_sys` version each gem resolves (pin its `Cargo.lock`), or rb-sys-dock derives a different name and the pre-seed is bypassed. Build the image with this repo's [`build-cross-images.yml`](.github/workflows/build-cross-images.yml); pass `ghcr.io/scientist-labs/rust-gem-cross`. The **only** mechanism that adds an absent linux system package without forking cross-gem. Empty default ⇒ **no-op** (every existing caller unchanged). |
| `linux-cross-image-tag` | string | no | `0.9.128` | The `rb_sys` version `:tag` the pre-seeded image is re-tagged to as `rbsys/<platform>:<this>`. MUST equal the `rb_sys` gem version each consumer gem resolves (the value rb-sys-dock derives the image name from). Default `0.9.128` matches `build-cross-images.yml` and the current crates.io `rb-sys` max_stable. Only consulted when `linux-cross-image-repo` is non-empty. |
| `job-timeout-minutes` | number | no | `360` | **Optional** per-job timeout for the heavy build legs (`linux-gems`, `darwin-build`, `darwin-package`). Default `360` = GitHub's own default, so existing runs are unchanged. Raise it for a heavy-C++ consumer (RocksDB, MuPDF/Tesseract, from-source OpenBLAS) whose per-ABI compile can approach the limit; lower it to fail a hung build faster. `prepare`/`source-gem`/`collect` are light and not gated. |
| `darwin-verify-cmd` | string | no | `""` (skip) | **Optional** extra shell run on each `darwin-build` leg after compile+relocate, with `$BUNDLE` exported as the freshly-relocated `lib/<ext-name>/<minor>/<ext-name>.bundle`. Empty default skips it (correct for plain CPU-only gems). red-candle asserts framework linkage via `otool -L "$BUNDLE" \| grep -Eiq '(Metal\|Accelerate)\.framework'`. The arm64 architecture check (`file "$BUNDLE" \| grep -q arm64`) **always runs and is hardcoded** — only the framework grep is gem-specific. |
| `publish` | boolean | no | `false` | **Intent gate** for `gem push`. Lives in `inputs` (not `secrets`) so it is legal in step `if:` and bash guards. **Default `false`** per the safety mandate: a tag still builds all gems, runs `gem build`, and creates/attaches the GitHub Release, but **skips the RubyGems push**, emitting a loud `::notice::`. Set `true` **and** supply the secret to actually publish. A caller threads this from its own `workflow_dispatch` boolean to drive an on-demand **dry-run** (every leg builds; the Release + attach steps are skipped on any non-`push` event) — see [Publishing](#publishing). |
| `build-darwin` | boolean | no | `true` | Toggle the native `arm64-darwin` legs (`darwin-build` + `darwin-package`) together, via job-level `if: inputs.build-darwin` on both. Off ⇒ a token-less or Linux-only gem still ships source + the two linux platforms. The native `macos-26` path is hardcoded (the Docker/osxcross cross path yields CPU-only darwin). |
| `build-x86_64-linux` | boolean | no | `true` | Toggle the `x86_64-linux` precompiled leg, via a per-step guard inside the `linux-gems` matrix (you cannot job-`if` a single matrix include). Off ⇒ amd64 users fall through to the source gem. `fail-fast: false` keeps one leg's flake from cancelling the other. |
| `build-aarch64-linux` | boolean | no | `true` | Toggle the `aarch64-linux` (cross-compiled on the `x86_64` host) leg, same per-step-guard mechanism. Off ⇒ arm64-linux users fall through to the source gem. The cross-on-x86_64 topology is **hardcoded** (a native arm runner trips cross-gem v1.4.4's `cargo-binstall` `KeyError aarch64-unknown-linux-musl`). |

## Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `rubygems-api-key` | no | RubyGems API key, mapped by every push step into `GEM_HOST_API_KEY` via `env:`. `required: false` is **load-bearing**: an omitted optional secret resolves to the empty string (not a hard "secret not provided" error), which the in-step `[ -z "$GEM_HOST_API_KEY" ]` guard treats as *skip* — so OSS forks, dry-runs, and `publish: false` get a clean no-op. Map your own secret by name: `rubygems-api-key: ${{ secrets.RUBYGEMS_API_KEY }}`. **Never referenced in any `if:`** (the `secrets` context is absent from both job-if and step-if). The GitHub Release uses `github.token`, not this key, so a tag yields a source gem + Release even with no key. Prefer this explicit named secret over `secrets: inherit`. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | Resolved gem version (pushed tag, leading `v` stripped). |
| `prerelease` | `'true'` if `Gem::Version#prerelease?` (drives the Release `--prerelease` flag). |
| `ext-name` | The resolved extension/lib-dir name (empty input resolved to `gem-name`); exposed for reuse in chained jobs. |
| `gemspec` | The resolved gemspec filename actually built. |
| `release-url` | URL of the GitHub Release created/updated (emitted from the non-matrix `source-gem` job, so it surfaces reliably). |
| `published` | `'true'` only if the source gem was actually pushed (`publish: true` **and** token present **and** not an idempotent repush-skip). Kept on the non-matrix `source-gem` job deliberately — a matrix job surfaces only its last entry's output. Lets a caller chain notify/smoke-install on a real publish. |
| `platforms-built` | Comma-list of platform gems that built green (`source,x86_64-linux,aarch64-linux,arm64-darwin`), aggregated in a tiny post-join `collect` job. Optional; the only GHA-correct place to assemble a per-platform list, since matrix jobs collapse to one output. |

---

## Consumer prerequisites

The shared workflow builds and packs the binaries, but **two small changes must live in
the consumer repo** before a precompiled gem will `require` at install time. Until both
exist, the source gem works everywhere and the precompiled gems publish but won't load.
The greenfield gems
(`parsekit`/`tokenkit`/`spellkit`/`clusterkit`/`lancelot`/`phrasekit`/`fastsheet`/`indradb-ruby`)
currently ship neither — **this is the real adoption blocker, not the caller YAML.**

### 1. ABI require-shim in `lib/<gem>.rb`

Try the Ruby-ABI-versioned native path first, fall back to the flat one. Resolution
**must** go through `$LOAD_PATH` (`require`, never `require_relative`) because RubyGems
installs native extensions outside the gem's `lib/` dir. Exact red-candle form
([`lib/candle.rb` lines 10-15](https://github.com/scientist-labs/red-candle/blob/main/lib/candle.rb#L10-L15)):

```ruby
begin
  RUBY_VERSION =~ /(\d+\.\d+)/
  require "candle/#{Regexp.last_match(1)}/candle"
rescue LoadError
  require "candle/candle"
end
```

The require path uses **`<ext-name>`**, not `<gem-name>` (e.g. `indradb/3.4/indradb`
then `indradb/indradb`).

### 2. Gemspec env-gated platform branch

The gemspec must read the env var named by `platform-gem-env` and, when set, set
`spec.platform`, **clear `spec.extensions = []`** (so RubyGems does not recompile from
Rust on install), and add the per-ABI binaries. Exact red-candle form
([`candle.gemspec` lines 32-38](https://github.com/scientist-labs/red-candle/blob/main/candle.gemspec#L32-L38)):

```ruby
if (platform_gem = ENV["RED_CANDLE_PLATFORM_GEM"])
  spec.platform   = platform_gem
  spec.extensions = []
  spec.files     += Dir["lib/candle/*/candle.bundle"] + Dir["lib/candle/*/candle.so"]
else
  spec.extensions = ["ext/candle/extconf.rb"]
end
```

**New adopters should read the generic default `RUST_GEM_PLATFORM`** here (and pass no
`platform-gem-env` input). red-candle reads `RED_CANDLE_PLATFORM_GEM`, so it overrides
`platform-gem-env` to match.

### 3. Caller workflow owns the tag trigger and the token grant

A `workflow_call` workflow has no `on: push: tags`. The **caller** supplies
`on: push: tags: [ … ]` and a job with `permissions: contents: write`. A reusable
workflow can only *reduce*, never elevate, the inherited token scope, so if the calling
job omits `contents: write` (or the repo's default token is read-only),
`gh release create` 403s. See [`examples/red-candle-release.yml`](examples/red-candle-release.yml).

---

## Gotchas this workflow handles for you

These are hard-won from a multi-round incident. The workflow bakes each one in so you
don't rediscover them:

1. **aarch64-on-x86_64-host cross-compile.** `aarch64-linux` is cross-compiled on the
   `x86_64` `ubuntu-24.04` host, **not** on a native arm runner — the pinned cross-gem
   v1.4.4's `cargo-binstall` host-triple lookup throws `KeyError aarch64-unknown-linux-musl`
   on an arm host. The topology is hardcoded; runner labels are never exposed as inputs.
2. **Optional `aws-lc-sys` cargo-config.** A gem with a TLS dep can hit `aws-lc-sys`'s
   `cc`-builder panic under rb-sys-dock's `x86_64` gcc (gcc bug 95189). Set
   `x86_64-cargo-config` to `[env]\nAWS_LC_SYS_CMAKE_BUILDER = "1"`; the workflow writes
   it to `.cargo/config.toml` **only on the x86_64 leg, before cross-gem** (the panic is
   host==target specific, so aarch64/darwin/source must never get it).
3. **Native darwin Metal.** `arm64-darwin` is built natively on `macos-26`; the
   oxidize-rb/rb-sys cross path is Linux/Docker-only and yields a CPU-only darwin binary
   (frameworks can't link under osxcross). Use `darwin-verify-cmd` to assert your
   framework linkage (red-candle greps `otool -L` for `Metal`/`Accelerate`).
4. **Generic `arm64-darwin` platform.** The fat gem publishes the *generic*
   `arm64-darwin` platform, because RubyGems does not fall back across darwin majors — a
   generic gem is what serves every `arm64-darwin-NN` user. Intel darwin is never built.
5. **Idempotent push.** A `gem push` that prints `Repushing of gem versions is not
   allowed` is treated as success (skip), so a partial re-run of a tag never fails.
6. **Source-gem-first.** The source (`ruby`) gem is built, pushed, and **creates the
   GitHub Release first**, so a tag always yields ≥1 installable gem and a Release for
   the native legs to attach to, even if a native leg flakes.

> **Darwin fat gem is all-or-nothing per run.** The arm64-darwin gem packs *every*
> Ruby ABI in `ruby-versions`, and `darwin-package` fails the run if any ABI bundle is
> missing. So a single-ABI compile failure on `darwin-build` (matrix `fail-fast: false`)
> drops the *entire* arm64-darwin gem for that run — the source + linux gems may already
> be published, but the darwin gem won't assemble until you re-run. The whole release is
> idempotent (idempotent push + `--clobber` attach), so a re-run after fixing the ABI
> simply fills in the darwin gem.

---

## Gems with system build dependencies

The precompiled **linux** legs build inside a **fixed `rb-sys-dock` image** via the
SHA-pinned `oxidize-rb/cross-gem@v1.4.4`, which exposes **no in-container install hook**
(no `pre-script`, no `apt-packages`). That image **already ships** `clang`/`llvm-12` +
`LIBCLANG_PATH`, `cmake` (`CMAKE_<triple>`), and the cross sysroot
(`BINDGEN_EXTRA_CLANG_ARGS_<triple>=--sysroot=…`). So:

- **bindgen / C++-from-clang gems (e.g. RocksDB / `indradb-ruby`)** work on both legs;
  augment the cross env with `aarch64-cargo-config` / `linux-cargo-config` (`[env]` for
  `CC`/`CXX`/extra `BINDGEN_EXTRA_CLANG_ARGS`) since the underlying tools are present.
- **gems needing a package the image lacks** — `protoc` (`lancelot`/`lance`), `gfortran`
  (from-source OpenBLAS / `clusterkit`), Tesseract/Leptonica/MuPDF dev (`parsekit`) —
  **cannot** be served by the precompiled linux legs through this workflow: a
  `.cargo/config.toml` `[env]` can only point at a binary that exists, and there is no
  apt step to add one. For these, set `build-x86_64-linux: false` and
  `build-aarch64-linux: false` and ship **source + (where it builds) darwin**; linux
  users fall through to the source gem (which compiles the dep on install).
- On **darwin**, `darwin-pre-build-command` (`brew install …`) **does** let a native leg
  install a system dep before compile (macos-26 has Homebrew). That is the one
  per-platform install escape hatch; it has no linux equivalent under the pinned
  cross-gem.

Lifting the linux limitation needs a **cross-gem bump to a version with a pre-build/
in-container step** (or a custom rb-sys-dock image with the deps baked in) — a pin change,
deliberately **not** done here.

---

## Publishing

Publishing is gated by a **triad** — an intent flag, an optional secret, and an in-step
bash guard — so the `secrets` context is never touched in a conditional (where
`if: secrets.X != ''` silently evaluates empty and is always false):

1. **`publish` (boolean input, default `false`)** gates *intent* and is legal in `if:`/bash.
2. **`rubygems-api-key` (secret, `required: false`)** — an omitted secret resolves to `''`
   with no hard error.
3. Every push step (source-gem, linux-gems, darwin-package) maps the secret into
   `GEM_HOST_API_KEY` and branches in bash:
   - `publish != true` ⇒ `exit 0` with a loud `::notice::` (built + attached to the
     Release, **not** pushed).
   - token empty ⇒ `exit 0` with a `::warning::`.
   - otherwise ⇒ idempotent `gem push` (the "Repushing…" skip from gotcha #5).

So publishing is a **clean no-op when *either* the flag is off *or* the token is absent**.
`gem build` **always** runs, so a dry-run still validates packaging across all four
targets — ideal for a greenfield gem (`tokenkit`/`spellkit`/`fastsheet`) that has no
RubyGems key yet. The GitHub Release creation is **never** gated on the rubygems key (it
uses `github.token`), so a tokenless tag still yields the source gem + a Release.

To actually publish (reproducing red-candle's behavior):

```yaml
with:
  publish: true
secrets:
  rubygems-api-key: ${{ secrets.RUBYGEMS_API_KEY }}
```

### Two event paths: tag push (real release) vs `workflow_dispatch` (dry-run)

The workflow branches on `github.event_name`, so the same reusable workflow serves both
the real release and a maintainer-triggered dry-run:

| | **`push` (a version tag)** | **`workflow_dispatch` (a branch)** |
| --- | --- | --- |
| `prepare` version | `GITHUB_REF_NAME` (v-stripped), **guarded** `== version-command` | taken **straight from `version-command`** (no tag guard — `GITHUB_REF_NAME` is the branch, not a version) |
| All four build legs | build | **build** (validate `gem build` + cross-gem + fat-gem assembly) |
| GitHub Release + every "Attach to Release" step | created/updated | **skipped** (gated `github.event_name == 'push'`) — no junk Release named after the branch |
| RubyGems push | per the `publish` triad above | per the `publish` triad (callers default `publish: false`, so **none**) |

A `workflow_dispatch` run is therefore a **full-matrix DRY-RUN**: every leg builds, but
it touches **neither RubyGems nor a GitHub Release**. To wire it, the caller adds the
trigger and a `publish` boolean (default `false`) it threads into `with.publish`:

```yaml
on:
  push: { tags: [ "[0-9]+.[0-9]+.[0-9]+", "v[0-9]+.[0-9]+.[0-9]+" ] }
  workflow_dispatch:
    inputs:
      publish:
        description: "Push to RubyGems (leave false for a dry-run)"
        type: boolean
        default: false

jobs:
  release:
    permissions: { contents: write }
    uses: scientist-labs/rust-gem-release/.github/workflows/release.yml@0.10.0
    with:
      # tag push -> the input is unset -> the workflow default (false) applies;
      # dispatch -> the maintainer's choice flows through.
      publish: ${{ github.event_name == 'push' || inputs.publish }}
      # ...gem-name, version-command, etc.
```

The tag-push path is **unchanged** — byte-identical to the always-publish release of
prior versions.

---

## Repo name (reusable workflow, not an action)

The repo is **`scientist-labs/rust-gem-release`**. This is unambiguously a **reusable
workflow** (`on: workflow_call`), **not** a composite/JS/Docker action — its four legs are
*separate jobs* across two runner OSes (`ubuntu-24.04` + `macos-26`) with `strategy.matrix`,
`needs` edges, and per-job `permissions`. A composite action runs as steps inside one
already-allocated job on one runner and cannot declare `runs-on`, fan out a matrix, span
macOS+Linux, or model `needs`/per-job permissions. The entry file ships as
`.github/workflows/release.yml`, and the name leaves room for a sibling PR-CI reusable
workflow (`.github/workflows/build.yml`) in the same repo.

`0.1.0` is the **first release**. Callers pin **`@0.10.0`** — a **moving major tag**, advanced
as the org rollout hardens the workflow, so an interface-widening reaches all callers at
once. For reproducibility you may instead pin the **immutable `@0.1.0`** point release (or,
for supply-chain parity with the SHA-pinned actions inside this workflow, SHA-pin
`@<sha>`).

---

## Example

A complete, real caller — red-candle's migrated release workflow, with the Metal/Accelerate
`darwin-verify-cmd`, the `aws-lc-sys` `x86_64-cargo-config`, and `publish: true` — lives at
[`examples/red-candle-release.yml`](examples/red-candle-release.yml).
