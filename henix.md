# henix.md — Operit: MCP / `sudo` shell tool calls intermittently hang

Investigation and fixes for a bug where MCP tool calls that run shell commands — especially anything
involving `sudo` — intermittently freeze the agent for minutes at a time.

Baseline: upstream `AAswordman/Operit` @ `dbf7191` (`chore(release): sync dev to main for 1.12.2`).

---

## TL;DR

There is no single bug. There are **six independent ways a command can wedge**, and the one that bites
`sudo` specifically is the least obvious: **command completion is decided by regex-matching the terminal
prompt text.** `sudo`'s password prompt matches nothing, so no completion event is ever emitted and the
caller waits out its full timeout.

| # | Where | What goes wrong |
|---|-------|-----------------|
| 0 | `terminal/` submodule — `OutputProcessor.isPrompt()` | **The real root cause.** Completion is a prompt-text guess applied to every output line. `sudo`'s prompt matches nothing → no completion → hang. Also false-positives: any output line ending in `$`/`#` ends the command early. |
| 1 | `MCPBridge.kt` | One **static** mutex + one reused socket with a blanket 180 s `soTimeout`. A single hung `toolcall` froze **every** MCP command in the app for 3 minutes. |
| 2 | `MCPToolExecutor.invoke()` | Non-suspend `runBlocking` boundary doing **3 serialized round trips** per call, on `Dispatchers.Default`. No `withTimeout` anywhere on the MCP path. |
| 3 | `Terminal.executeCommand` | `deferred.await()` with **no timeout**. A dead PTY never emits the completion event → permanent hang. |
| 4 | `MCPSharedSession` | Caches `sharedSessionId` forever, never revalidates it against the live session list. |
| 5 | `RootShellExecutor` | Real stdout-then-stderr **pipe-buffer deadlock**, and libsu's `Job.exec()` has **no timeout** (verified in libsu 6.0.0 `Shell.java`: `setTimeout` only bounds *shell verification*). |

The 30-minute waits came from `execute_in_terminal_session`'s `timeout_ms` default of `1800000` — set in
**both** the tool schema (`SystemToolPromptsInternal.kt`, English *and* Chinese) and
`StandardTerminalCommandExecutor.kt`.

---

## 0. Root cause — completion is a prompt-text guess

`terminal/src/main/java/com/ai/assistance/operit/terminal/view/domain/OutputProcessor.kt`

`isPrompt()` is applied to **every line of output** via `handleReadyState()`. There is no sentinel and no
exit code — completion is inferred purely from whether the text looks like a shell prompt.

The original predicate accepted a bare `$` or `#` suffix:

```kotlin
return trimmed.endsWith("$") ||
        trimmed.endsWith("#") ||
        trimmed.endsWith("$ ") ||
        trimmed.endsWith("# ") ||
        Regex(".*@[a-zA-Z0-9.\\-]+\\s?:\\s?~?/?.*[#$]\\s*$").matches(trimmed) ||
        ...
```

Two failure modes fall straight out of that:

* **False negative → the `sudo` hang.** `sudo` prints `[sudo] password for user:` and blocks. That matches
  no pattern, so no completion event fires. The caller sits there until its timeout expires — 30 minutes for
  the terminal tool.
* **False positive → premature completion.** Any output line ending in `$` or `#` (e.g. `total: $`, a `#`
  comment, a hashed string) is taken for a prompt, so the command is marked finished while it is still
  running.

### Ground truth for the prompt

Extracted from the bundled rootfs `terminal/src/main/assets/ubuntu-noble-aarch64-pd-v4.18.0.tar.xz`
(using Python's `lzma` + `tarfile`, since no `xz` binary was available):

| File | `PS1` |
|------|-------|
| `etc/profile` | `# ` (root) / `$ ` (user) |
| `etc/bash.bashrc`, `root/.bashrc` | `\u@\h:\w\$ ` → e.g. `root@host:/root# ` |

So **both** the bare form and the `user@host:path#` form are real. The bare heuristic is not dead code —
but it must not accept arbitrary text.

### The fix

Bare `$`/`#` is only accepted when the line has **no internal whitespace**; the structured form keeps a
regex that tolerates spaces in the expanded `\w` path:

```kotlin
if (trimmed.isNotEmpty() &&
    !trimmed.contains(' ') &&
    (trimmed.endsWith("$") || trimmed.endsWith("#"))
) {
    return true
}
return Regex(".*@[a-zA-Z0-9.\\-]+\\s?:\\s?~?/?.*[#$]\\s*$").matches(trimmed) ||
        Regex("root@[a-zA-Z0-9.\\-]+:\\s?~?/?.*#\\s*$").matches(trimmed)
```

> **This file lives in the `terminal/` submodule (`OperitTerminalCore`), which is a separate repository.**
> It is **not** part of the app commit — see [Submodule](#submodule-the-terminal-fix) below.

---

## App-side fixes (7 files, +465 / −72)

These break every *other* hang path and, crucially, make the `sudo` case **fail fast with an explanation**
instead of hanging for half an hour — even without the submodule fix.

### `Terminal.kt`

* `executeCommandFlow()` / `executeCommand()` now run a **watchdog** over `terminalState`. If a session sits
  on an interactive prompt for more than a 2 s grace period (double-checked, so a fast prompt does not trip
  it), the command is ended with an explanatory message instead of blocking:

```kotlin
private const val INTERACTIVE_PROMPT_GRACE_MS = 2_000L
private const val INTERACTIVE_PROMPT_POLL_MS  = 500L
```

* New `interactivePromptMessage(prompt)` tells the caller what happened and how to avoid it
  (`sudo -n`, or configure `NOPASSWD`).
* The collector scope moved from `Dispatchers.Main` to `Dispatchers.IO` — a `Main`-dispatched scope inside
  a `runBlocking` boundary is a latent deadlock.

### `MCPBridge.kt`

* Dedicated socket for `toolcall` instead of sharing one with control traffic.
* Split timeouts: control 15 s / `toolcall` 60 s / `spawn` 180 s.
* Distinct `SocketTimeoutException` logging so the failing phase is identifiable.

### `MCPToolExecutor.kt`

* Static per-server `toolSchemaCache`, invalidated on register / unregister / shutdown.
* Dropped the `isActive()` round trip — 3 round trips per call became 1.
* `withTimeout` backstops on the remaining calls.

### `MCPSharedSession.kt`

* Revalidates the cached `sharedSessionId` against `terminal.terminalState.value.sessions` before use.

### `RootShellExecutor.kt`

* Concurrent stdout/stderr drain via `CompletableFuture` — fixes the pipe-buffer deadlock.
* `execWithTimeout` wrapper, since libsu's `exec()` is unbounded.
* Wedged-shell fast-fail flag + `waitAndClose` recovery probe.

### `SystemToolPromptsInternal.kt` + `StandardTerminalCommandExecutor.kt`

* `execute_in_terminal_session` `timeout_ms` default lowered from `1800000` (30 min) to `600000` (10 min),
  in the English schema, the Chinese schema, and the two executor defaults.

---

## Submodule: the terminal fix

`OutputProcessor.kt` is the actual root cause fix, but it lives in
`https://github.com/AAswordman/OperitTerminalCore.git` — a **separate repository** that is not forked here.

To land it you need to:

1. Fork `OperitTerminalCore`.
2. Apply the `isPrompt()` change above to `src/main/java/com/ai/assistance/operit/terminal/view/domain/OutputProcessor.kt`.
3. Point `.gitmodules` at your fork, then bump the `terminal` gitlink in the parent repo.

Without it the app still stops hanging (the `Terminal.kt` watchdog handles that), but the underlying
completion detection stays a prompt-text guess.

---

## Verification

Both modified modules **compile clean**:

```
./gradlew :app:compileDebugKotlin :terminal:compileDebugKotlin
→ BUILD SUCCESSFUL in 4m 12s
  131 actionable tasks: 5 executed, 126 up-to-date
```

Emitting `Terminal.class` (19,396 B) and `OutputProcessor.class` (28,133 B).

### Building on Windows

Upstream is **Linux-first** — every workflow in `.github/workflows/` runs `ubuntu-24.04`, and
`llm/mnn/CMakeLists.txt` only looks for a host C/C++ compiler under
`if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux")`. There is no Windows branch, so a Windows host with no
MSVC/MinGW fails at `:mnn:configureCMakeDebug` when it tries to build FlatBuffers' `flatc`.

That can be worked around without installing a compiler: MNN checks its generated
`schema/current/*_generated.h` into the tree at every commit, and all nine `.fbs` schemas were verified
consistent with the matching headers at the pinned revision `d407447…`. A small CMakeLists change that
falls back to the checked-in headers when no host compiler is present takes
`:mnn:configureCMakeDebug[arm64-v8a]` from **FAILED** to **BUILD SUCCESSFUL in 22s**, leaving Linux CI
behaviour untouched. *(Kept out of this commit — say the word and it goes in as its own change.)*

---

## Open items

* `RootShellExecutor.isAvailable()` still calls the blocking `Shell.getShell()` on every `executeCommand`.
  With `setTimeout(10)` a cold shell costs a 10 s stall. Preheating at startup would fix it.
* None of this makes `sudo` work non-interactively. If the environment's `sudo` is not `NOPASSWD`, a
  `sudo` command still cannot complete — it now fails fast instead of hanging. Configure `NOPASSWD`, or
  prefix `sudo -n` (the bundled `examples/linux_ssh` package already does exactly this).
* `TerminalManager` (PTY layer) was only partially inspected; the `MCPSharedSession` fix assumes
  `terminal.terminalState.value.sessions` is authoritative — the same assumption
  `StandardTerminalCommandExecutor.createOrGetSession()` already makes.

---

## Confirming on-device

```bash
adb logcat -s MCPBridge MCPToolExecutor MCPBridgeClient RootShellExecutor Terminal
```

| Log line | Confirms |
|---|---|
| `命令[...: toolcall]读取超时` | #1 — bridge never answered |
| Another server's `list`/`listtools` stalling behind a `toolcall` | the global-mutex serialization (pre-fix) |
| `会话 ... 停留在交互式提示上` | #0 — the `sudo` prompt path, now fast-failing |
| `缓存的共享会话 ... 已不存在，将重新创建` | #4 |
| `Root 命令在 ...ms 内没有返回，判定共享 su shell 已挂起` | #5 |
