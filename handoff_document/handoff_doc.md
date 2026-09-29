# Operit — MCP / `sudo` shell hang: handoff

**Baseline:** upstream `AAswordman/Operit` @ `dbf7191` (`chore(release): sync dev to main for 1.12.2`)
**Fork:** `h3n1x0s/Operit`
**Landed:** commit `7a4064b` on `fork/main`
**Status:** the app-side fixes are pushed and compile clean. The **root-cause fix** (submodule) and the
**Windows build fix** are *not* landed — both are shipped here as patches.

---

## 1. What is already in the fork

```
7a4064b  fix(mcp): stop shell/tool calls from wedging, especially sudo
 8 files changed, 677 insertions(+), 72 deletions(-)
```

| File | Δ | Fix |
|---|---|---|
| `app/.../tools/system/shell/RootShellExecutor.kt` | +205 | concurrent stdout/stderr drain (pipe deadlock), `execWithTimeout`, wedged-shell recovery |
| `app/.../tools/system/Terminal.kt` | +128 | interactive-prompt watchdog (2 s grace), collector off `Dispatchers.Main` |
| `app/.../tools/mcp/MCPToolExecutor.kt` | +103 | per-server `toolSchemaCache`, dropped the `isActive()` round trip, `withTimeout` backstops |
| `app/.../data/mcp/plugins/MCPBridge.kt` | +63 | dedicated `toolcall` socket, split timeouts (15 / 60 / 180 s) |
| `app/.../data/mcp/plugins/MCPSharedSession.kt` | +30 | revalidate cached session against the live list |
| `app/.../core/config/SystemToolPromptsInternal.kt` | 4 | `timeout_ms` default 30 min → 10 min (EN + ZH) |
| `app/.../tools/defaultTool/standard/StandardTerminalCommandExecutor.kt` | 4 | same default lowered |
| `henix.md` | +212 | the fix write-up |

Deliberately **not** in that commit: build artifacts (`app/libs/.keep`, `app/objectbox-models/*`), the
`terminal` gitlink, and `llm/mnn/CMakeLists.txt`.

---

## 2. Patches in this folder

| Patch | Lines | Applies to | State |
|---|---|---|---|
| `operit-mcp-stuck-fix.patch` | 853 | Operit repo root (7 files) | already applied in `7a4064b` — kept as a record / for other forks |
| `operit-terminal-completion-fix.patch` | 34 | **inside the `terminal/` submodule** (1 file) | **not applied anywhere** — needs a fork |
| `operit-mnn-windows-build.patch` | 78 | Operit repo root (`llm/mnn/CMakeLists.txt`) | **not applied anywhere** — optional |

Apply the root-level patches from the repo root:

```bash
git apply handoff_document/operit-mcp-stuck-fix.patch
git apply handoff_document/operit-mnn-windows-build.patch
```

The submodule patch applies from **inside** `terminal/`:

```bash
git -C terminal apply ../handoff_document/operit-terminal-completion-fix.patch
```

> Gotcha: `git apply --check` in *forward* mode fails once a patch is already applied. Try `-R` first to
> tell "already applied" apart from "conflict".

---

## 3. Not landed

### 3.1 The actual root cause — `terminal/` submodule

`terminal/src/main/java/com/ai/assistance/operit/terminal/view/domain/OutputProcessor.kt`

`isPrompt()` decides command completion by **regex-matching the prompt text**, applied to every output
line. There is no sentinel and no exit code. `sudo`'s `[sudo] password for …:` prompt matches nothing, so
no completion event is ever emitted and the caller waits out its full timeout — **this is the bug.** The
same predicate also false-positives on any output line ending in `$` or `#`.

The patch requires a bare `$`/`#` to have no internal whitespace, and keeps the `user@host:path#` regex
for the structured form (ground truth: `etc/profile` sets `PS1='# '`/`'$ '`; `etc/bash.bashrc` and
`root/.bashrc` set `PS1='\u@\h:\w\$ '`).

**Why it isn't landed:** `terminal/` is `https://github.com/AAswordman/OperitTerminalCore.git` — a
separate repo, and `h3n1x0s/OperitTerminalCore` does not exist (404). To land it:

1. Fork `OperitTerminalCore`.
2. Apply `operit-terminal-completion-fix.patch` inside it, commit, push.
3. In the parent repo, point `.gitmodules` at the fork and bump the `terminal` gitlink.

**Not fatal if skipped:** the app still *stops hanging*, because the `Terminal.kt` watchdog fast-fails an
interactive prompt after 2 s with an explanatory message. But completion detection stays a text guess.

### 3.2 Windows build — `llm/mnn/CMakeLists.txt`

Upstream is Linux-first: every workflow in `.github/workflows/` runs `ubuntu-24.04`, and the CMakeLists
only looks for a host C/C++ compiler under `if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux")`. There is **no
Windows branch**. On a Windows host with no MSVC/MinGW, `:mnn:configureCMakeDebug` fails while trying to
build FlatBuffers' `flatc`.

The patch adds an `OPERIT_MNN_FORCE_SCHEMA_REGEN` option (default OFF) gating the `flatc` build, and falls
back to MNN's checked-in `schema/current/*_generated.h` when there is no host compiler. Linux behaviour is
unchanged — it still regenerates. Safety was established by verifying all 9 `.fbs` schemas against the
matching `_generated.h` at the pinned revision `d407447…`: every `table`/`struct`/`enum` present, zero
missing.

**Verified:** `:mnn:configureCMakeDebug[arm64-v8a]` went **FAILED → BUILD SUCCESSFUL in 22s**.

Not required for Linux CI, so it was left out of `7a4064b`.

---

## 4. Build & verification status

| Check | Result |
|---|---|
| `:app:compileDebugKotlin` | **BUILD SUCCESSFUL** (4 m 12 s) |
| `:terminal:compileDebugKotlin` | **BUILD SUCCESSFUL** — `OutputProcessor.class` emitted |
| `:mnn:configureCMakeDebug[arm64-v8a]` | **BUILD SUCCESSFUL** (22 s, with the CMakeLists patch) |
| Full `:app:assembleDebug` | **not completed** — see below |
| APK produced | **none** |

Full APK blockers, in order of removal:

1. ~~Host `flatc` for `:mnn`~~ — solved by `operit-mnn-windows-build.patch`.
2. **Open:** `CMAKE_OBJECT_PATH_MAX` warning — MNN's nested object dir is 155 chars and some kleidiai
   object paths may exceed the 250-char limit ("build may not work correctly"). Untested.
3. **Open:** `tools/native_ripgrep` needs Rust/cargo (not installed).

The `:mnn` native compile is long (MNN + LLM + kleidiai, plus `:llama`, `:quickjs`, `:mmd`, `:fbx`), so the
practical route to an APK is CI.

---

## 5. Environment facts

**Toolchain** — none of it pre-existed; all installed under `C:\Users\hen\.workbuddy-ai\toolchain\`:

| Component | Version |
|---|---|
| Temurin JDK | 21.0.12.1+1 |
| Gradle | 8.13 (wrapper dist, sha256 `20f1b117…0aed78`) |
| Android SDK platform | 36 |
| Android build-tools | 36.0.0 |
| NDK | 27.0.12077973 |
| CMake | 3.22.1 |

Note the CI pins **NDK 25.1.8937393**; there is no `ndkVersion` pin in the Gradle files.

`local.properties` → `sdk.dir=C:\Users\hen\.workbuddy-ai\toolchain\android-sdk`.

Two prebuilts the repo does not check in were fetched from the project's Google Drive archives per
`ci/script/download_android_dependencies.sh`: `app/libs/ffmpeg-kit-local.aar` (for `:app:compileDebugAidl`)
and `app/src/main/jniLibs/arm64-v8a/liboperit_ripgrep.so` + `libc++_shared.so` + `app/src/main/assets/subpack/`
(for `:app:verifyExternallyBuiltNativeLibraries`).

**Git:** `gh` is not installed, there is no credential helper, no `~/.git-credentials`, no `~/.netrc`, and
no token env var — yet `git push` to the fork succeeds unprompted (Windows Credential Manager holds it).
Identity is set **locally** only: `hen <h3n1x0s@users.noreply.github.com>`.

**Disk:** `C:` runs tight — 238 G, ~36 G free. The first clone attempt died with `No space left on device`.

---

## 6. Next steps

1. **Get an APK.** The push to `main` touched `app/**`, so `android-build.yml` should already have fired on
   `ubuntu-24.04`. Check Actions for the `assembleDebug` artifact.
2. **Fork `OperitTerminalCore`** and land `operit-terminal-completion-fix.patch` — without it the root
   cause remains.
3. **Optional:** apply `operit-mnn-windows-build.patch` to build on Windows hosts without a C++ compiler.
4. **Optional:** commit the CMakeLists patch to the fork as its own commit.

---

## 7. Open items

* `RootShellExecutor.isAvailable()` still calls the blocking `Shell.getShell()` on every `executeCommand`.
  With `setTimeout(10)` a cold shell costs a 10 s stall; preheating at startup would fix it.
* Nothing here makes `sudo` work *non-interactively*. If `sudo` is not `NOPASSWD`, a `sudo` command still
  cannot complete — it now fails fast instead of hanging. Configure `NOPASSWD`, or prefix `sudo -n` (the
  bundled `examples/linux_ssh` package already does this).
* `TerminalManager` (PTY layer) was only partly inspected. The `MCPSharedSession` fix assumes
  `terminal.terminalState.value.sessions` is authoritative — the same assumption
  `StandardTerminalCommandExecutor.createOrGetSession()` already makes.
