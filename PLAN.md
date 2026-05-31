# Implementation plan for the Logos LSP + VS Code extension

This document briefs the agent that will build this project. Read [init.md](init.md) first — that is the feature spec. This file is the *how*.

## Architecture decision (already made)

**Option A: piggyback on clangd.** Our LSP server is a thin Logos-aware layer that delegates all C/ObjC understanding to clangd. We are *not* writing our own ObjC parser or indexer.

Data flow per `.x`/`.xm` file:

```
file.x ──logos.pl──▶ file.x.m  (with #line directives)
                          │
                          ▼
                       clangd  ──diagnostics/hover/defs──▶ our LSP ──▶ VS Code
                                                              │
                                                              └── translates positions
                                                                  back to .x via #line
```

Our server owns:
- Logos directive parsing (`%hook`, `%end`, `%new`, `%orig`, `%group`, `%c`, `%init`, `%ctor`, `%dtor`, `%subclass`, `%property`, `%config`, `%log`)
- Logos-only diagnostics (unclosed blocks, `%orig` in `%new`, etc.)
- Running `logos.pl` to produce the `.m` that clangd consumes
- Proxying `textDocument/*` requests to clangd, with position translation

Clangd owns:
- ObjC parsing, type resolution, completion, hover, go-to-def, find-refs
- SDK / framework header indexing
- Real compile errors

## Standalone (no-project) behavior

The extension is **not just a project-aware LSP**. It must work usefully on *any* `.x`, `.xm`, `.m`, `.mm`, or `.h` file the user opens — including files outside any workspace, and including files transitively reached by cmd-clicking through `#import`s (e.g., the user is now browsing `Foundation.h` inside an SDK directory and wants to cmd-click `<CoreFoundation/CFBase.h>` from there). This section defines that contract. The seed of this requirement is [init.md](init.md#L11) ("even for headers outside the project…"), but it's broader than imports — it applies to type resolution too.

### Files in scope

- A loose `.x` / `.xm` / `.m` / `.mm` opened by itself with no workspace.
- A `.h` that is part of the workspace.
- A `.h` opened *transitively* from another file — typically read-only, often inside an SDK or framework bundle.
- A `.h` the user is browsing directly from inside an SDK.

In all four cases, the extension must provide at minimum: syntax highlighting, Logos keyword completion + snippets (for `.x`/`.xm`), `#import` resolution, and best-effort go-to-definition on types.

### `#import` resolution order

When the user cmd-clicks a `#import` (or `#include`), resolve in this order. Stop at the first hit. Surface the winning path somewhere accessible (status bar, output channel) so misroutes are debuggable.

1. **Quoted form `#import "Foo.h"`**: relative to the current file's directory, then walk up toward the project root (or filesystem root if standalone).
2. **Angle form `#import <Framework/Header.h>`**:
    1. Project-local search paths (`-I` / `-F` from `compile_commands.json` for this file), if available.
    2. **Theos SDK** at `$THEOS/sdks/iPhoneOS*.sdk` — public iOS framework headers, versioned to match the deployment target.
    3. **Theos vendor include** at `$THEOS/vendor/include` — community-maintained PrivateFrameworks headers, IOKit class-dumps, etc.
    4. **Xcode active SDKs**, queried via `xcrun --sdk iphoneos --show-sdk-path` and `xcrun --sdk macosx --show-sdk-path`. Never hardcode `/Applications/Xcode.app/...`.
    5. **System headers** (`/usr/include`, `/usr/local/include`, Homebrew prefixes).

**Why Theos SDK beats Xcode SDK**: for tweak development the Theos-managed iOS SDK is more correct — it matches the deployment target and often includes class-dumped headers Apple's SDK doesn't ship. For a vanilla macOS file in a non-Theos workspace, flip the priority: Xcode SDK should win. Detect this from *workspace shape* (presence of Theos `Makefile`, `control`, etc.), not per-file.

Order must be configurable via extension settings. The above is the default.

### Go-to-definition on arbitrary symbols

Cmd-click on `UIView`, `NSString`, `AVPlayer`, or any user-defined class should land on its `@interface` / `@protocol` / `struct` / `typedef` wherever it lives. This needs clangd to have *indexed* the headers, not just be able to resolve them on demand.

Strategy:

1. **Persistent SDK index**: at first activation, spawn clangd with a synthetic compile command that pulls in the SDK roots from the `#import` resolution order. Let clangd background-index. Persist the index across sessions at `~/.cache/logos-lsp/clangd-index/` (clangd's `--background-index-priority` and `--index-file` flags).
2. **Project index** layered on top when `compile_commands.json` exists.
3. **Per-file transitive augmentation** for standalone files: parse the file's `#import`s, resolve each per the order above, and add their containing directories as `-I` / `-F` on the synthesized compile command for that file. This makes navigation work even before the SDK index is fully built.
4. **Inside-SDK detection**: when the active file's path is inside an SDK (walk up looking for `SDKSettings.plist` or `*.sdk/`), set `-isysroot` to that SDK root for that file's compile command. Transitive navigation through Apple's headers then works without any project setup.

### SDK index bootstrap

**Indexing itself is clangd's job, not ours.** Clangd's `--background-index` writes binary symbol records (declarations, references, defs) under `.cache/clangd/index/` for every TU it sees. We get cross-file go-to-def and global symbol completion for free *for any file clangd has compiled*.

**The catch:** clangd only indexes what's in a TU it actually compiles. If the project's `.x` files never `#import <AVFoundation/...>`, clangd never sees AVFoundation, and `AVAudioSession` is not navigable. This breaks the "No project" column of the capability matrix.

**The fix:** induce clangd to index the SDKs by feeding it synthetic translation units.

#### How

1. **Enumerate frameworks.** Scan `$THEOS_SDK/System/Library/Frameworks/*.framework`, the Xcode SDKs (via `xcrun --show-sdk-path`), and `$THEOS/vendor/include` for private framework dumps. Collect a list of `(framework name, umbrella header path, sysroot)`.
2. **Generate stub TUs.** For each framework, write a one-line file to `~/.cache/logos-lsp/synthetic-tus/`:
    ```
    Foundation.m:     #import <Foundation/Foundation.h>
    UIKit.m:          #import <UIKit/UIKit.h>
    AVFoundation.m:   #import <AVFoundation/AVFoundation.h>
    …
    ```
   When a framework lacks an umbrella header, fall back to importing every `.h` in its `Headers/` directory in a single stub.
3. **Write a synthetic `compile_commands.json`** alongside the stubs, with `-isysroot`, `-F`, `-x objective-c`, and ARC enabled for each TU.
4. **Hand it to clangd.** Clangd indexes in the background, writes `.idx` files, persists.
5. **Cache the bootstrap output** by `(SDK identifier, SDK version hash, generator script version)`. Subsequent activations check the cache and skip if valid.

#### When

Bootstrap runs **on first activation after install** and **whenever the cache key changes** (Theos updated, Xcode upgraded, settings changed). Not on every activation, not lazily per-symbol. Foreground UI shows a status-bar item with progress; user can cancel and resume later.

For users editing a single `.x` outside any project, bootstrap happens the first time the extension activates for the `logos` language regardless of workspace presence. Standalone users get the same indexed coverage as project users.

#### Budgets

- Time: a few minutes for iPhoneOS + macOS SDKs on a modern Mac. Acceptable as one-time.
- Disk: ~100 MB total for the index (clangd's format is compact). Pruning policy: drop indices for SDK versions no longer installed.
- Memory: clangd's working set during indexing can spike to 1–2 GB. Surface this so users on small machines can throttle parallelism via settings.

#### Strategies we explicitly rejected

- **Pre-built index shipped with the extension.** Avoids the wait but couples extension releases to SDK versions. Painful to maintain.
- **Lazy per-framework indexing on first reference.** Breaks completion: you can't complete a symbol whose framework you haven't yet typed. Surprises the user with delays.
- **Relying on incidental project compilation.** Works for project mode but kills standalone go-to-def. Insufficient for our capability matrix.

#### What if Xcode/Theos isn't installed?

Bootstrap reports what it found and what it skipped. The extension still works in degraded mode — Logos directive features and per-file resolution still function; SDK-wide go-to-def just won't return anything outside what the current file's `#import`s reach. Surface this state clearly; don't silently degrade.

### Fallback compile command (no `compile_commands.json`)

When no entry exists for a file, synthesize one:

```
clang -x objective-c \
    -isysroot <Theos iPhoneOS SDK | Xcode active SDK | inside-SDK root> \
    -F <Theos vendor PrivateFrameworks> \
    -I <project root, if any> \
    -I <directories from resolved #imports in this file> \
    -fobjc-arc \
    -std=gnu11 \
    <file>
```

Use `-x objective-c++` for `.mm` / `.xm` (post-logos). ARC vs MRC: default to ARC; downgrade if the file uses `[obj retain]`/`release` patterns (heuristic — don't be clever, just a fallback).

### Capability matrix

| Feature | No project | Theos detected, no compile DB | Full project (Theos + compile_commands.json) |
|---|---|---|---|
| Syntax highlighting (`.x`/`.xm`/`.h`/`.m`) | yes | yes | yes |
| Logos keyword completion + snippets | yes | yes | yes |
| Block-pair validation (`%hook`/`%end`) | yes | yes | yes |
| Doc-hover for Logos keywords | yes | yes | yes |
| `#import` cmd-click (incl. transitively inside SDKs) | yes | yes | yes |
| Go-to-def on SDK types (NSString, UIView, …) | yes | yes | yes |
| Go-to-def on types reachable via `#import` graph | yes (best-effort) | yes | yes |
| Completion of project-defined types | — | partial (heuristic header scan) | yes |
| `%hook ClassName`-aware method completion | — | yes | yes |
| Logos semantic diagnostics (`%orig` in `%new`, etc.) | yes | yes | yes |
| Live ObjC compile errors | — | — | yes |
| `@interface` stub generation from runtime / class-dump | — | yes | yes |
| Auto-import missing headers | — | partial | yes |
| Rename refactor across `%hook` blocks | — | — | yes |

### Don'ts for standalone mode

- **Don't refuse** to load a file because no project is detected. Standalone *is* a supported mode.
- **Don't pop modal dialogs** asking the user to configure Theos. A single non-intrusive status item is fine.
- **Don't write files** (`compile_commands.json`, `theos/`, cache dirs) into a directory that isn't a recognized project. Per-workspace state goes in the extension's storage URI; global state in `~/.cache/logos-lsp/`.
- **Don't index the user's home directory** searching for headers. Only walk the explicit roots from settings.
- **Don't disable Logos features** just because the file is a `.h` or `.m` (someone might be editing a header that's part of a tweak).

## What Theos already provides — DO NOT REINVENT

The fresh clone lives at `~/Developer/theos` (cloned with `--recursive`). Treat it as read-only reference and reuse its tooling.

| Capability | Theos artifact | Where |
|---|---|---|
| `.x`/`.xm` → `.m`/`.mm` translation | `logos.pl` | `theos/vendor/logos/bin/logos.pl` |
| `#line` directive emission | inside `logos.pl` (`generateLineDirectiveForPhysicalLine`) | same |
| `compile_commands.json` generation | `make commands` target | `theos/makefiles/master/rules.mk:226` |
| Per-compile capture wrapper | `gen-commands.pl` | `theos/bin/gen-commands.pl` |
| New-project scaffolding | `nic.pl` + templates | `theos/bin/nic.pl`, `theos/vendor/templates/` |
| Syntax-highlight reference | vim syntax file | `theos/vendor/logos/extras/vim/` |
| Logos parser-of-record | the Perl modules | `theos/vendor/logos/bin/lib/Logos/*.pm` |

**Generator pluggability:** `logos.pl` has three back-ends (`Base`, `MobileSubstrate`, `libhooker`) selectable via flags. Default is `MobileSubstrate`. Our LSP should accept the same flag set so we generate the same `.m` the build does.

## Do's

- **Always invoke `logos.pl` for source translation.** Read its output. Never try to imitate what it does.
- **Honor `#line` directives** in the generated `.m` as the authoritative forward source map. They are emitted as either `#line N "file.x"` (raw `.x`/`.xm` mode) or `# N "file"` (preprocessed `.xi`/`.xmi` mode).
- **Trigger or read from `make commands`** for `compile_commands.json`. If the file is stale or missing, offer a command that runs `make commands` in the workspace; do not generate it ourselves.
- **Detect a Theos project** by, in order: `$THEOS` env set + `Makefile` referencing `$THEOS/makefiles/*`, then look for a project-local `theos` symlink, then look for `control` + `Makefile` shape. Don't require all signals.
- **Make the Theos path configurable** in extension settings. Default to `$THEOS` env, fall back to `/opt/theos` only as a last resort. For development of *this LSP*, point it at `~/Developer/theos`.
- **Keep the Logos parser separate** from the clangd proxy. A user editing a `.x` file should get Logos diagnostics even when clangd is unavailable (no compile_commands yet).
- **Use a TextMate grammar** for syntax highlighting (#7 in [init.md](init.md)). Reference the vim file in `extras/vim/` as a starting point.
- **Cache the generated `.m`** keyed by file content hash. `logos.pl` is fast but not free, and you'll re-run it on every keystroke if you're not careful.
- **Implement features in phases** (see below). Don't try to ship all 26 items at once.

## Don'ts

- **Don't reimplement `logos.pl` in TypeScript/Rust.** It is the ground truth for what valid Logos is. Wrap it.
- **Don't reimplement `compile_commands.json` generation.** `make commands` exists.
- **Don't write our own ObjC parser, type system, or header indexer.** That's clangd's job. If you find yourself wanting one, stop and reconsider.
- **Don't fork or modify the cloned Theos.** Treat `~/Developer/theos` as read-only. If logos.pl has a bug that blocks you, file an issue upstream and work around it.
- **Don't touch the user's installed Theos** (typically `/opt/theos` or `~/theos`). It has local modifications the user wants preserved. Use the clone for testing.
- **Don't assume the Theos location.** Always go through the configured path.
- **Don't write column-precise reverse mapping in v1.** `#line` is line-only and that's good enough for diagnostics/hover/most jumps. Defer column mapping until a feature actually needs it (likely rename / find-refs span highlighting).
- **Don't run `make` on every keystroke.** That rebuilds everything. For live diagnostics, invoke `logos.pl` directly and hand the result to clangd's `textDocument/didChange` with the recorded compile command from `compile_commands.json`.
- **Don't ship without a fallback** for non-Theos workspaces. The extension should at least give syntax highlighting and Logos keyword completion when no Theos is detected.

## Gotchas

1. **File-type variants compile differently:**
   - `.x` / `.xm` → straight to `logos.pl` → `.m` / `.mm`
   - `.xi` / `.xmi` → `cc -E` first, **then** `logos.pl` → `.mi` / `.mii`

   The preprocessed path emits `# N "file"` markers (gcc-style); the raw path emits `#line N "file.x"`. Parse both.

2. **`logos.pl` requires a filename argument.** It does not read stdin. For live editing you'll need to write the buffer to a temp file (or a pipe-fed temp path) before calling it. Check if upstream will accept a stdin patch later.

3. **Generated symbol names are mangled.** `%hook Foo` `- (void)bar` becomes something like `_logos_method$_ungrouped$Foo$bar` in the `.m`. If clangd reports a reference at that symbol, your proxy must recognize the pattern and present it to the user as `-[Foo bar]` in the original `.x`. Build a name-demangling table from logos's generator output, or scrape it post-hoc with a regex.

4. **`#line` directives only map lines, not columns.** Inside passthrough regions (regular ObjC), column == column. Inside transformed regions (a line containing `%orig` or a method header rewritten with the mangled name), the column in the `.m` does not match the column in the `.x`. For v1: snap to line, accept imperfect column.

5. **`compile_commands.json` records the `.x.m` path, not the `.x` path.** Clangd opens the `.x.m`. Your LSP must intercept and present `.x` to the user while feeding clangd `.x.m`. This is the central proxy responsibility.

6. **The generator back-end matters.** If a user builds with `THEOS_PACKAGE_SCHEME=rootless` or chooses libhooker, the `.m` output differs. Read `ALL_LOGOSFLAGS` from the build (or from the captured command in `compile_commands.json`) so our preview generation matches.

7. **Clangd is not Theos-aware.** It will follow `#line` directives but it has no idea what `%hook` is. All Logos diagnostics must come from our server. Don't expect clangd to catch e.g. an `%orig` inside a `%new`.

8. **Submodules.** Theos uses submodules heavily (`logos`, `orion`, `templates`, etc.). The clone at `~/Developer/theos` was made with `--recursive` — keep it that way. If you need to update it, `git submodule update --recursive`.

9. **macOS quarantine / Gatekeeper** can interfere with launching `logos.pl` from a path the user didn't explicitly approve. Surface a clear error if `logos.pl` fails to execute.

10. **VS Code's existing clangd extension** (`llvm-vs-code-extensions.vscode-clangd`) already wires clangd into VS Code. Decision: do we spawn our own clangd, or ride on top of theirs? Recommendation: **spawn our own** so we can intercept `compile_commands.json` lookups and translate paths. Document the conflict if both extensions are active.

11. **Theos isn't always in `$THEOS`.** Some users put it in a project-local `./theos` symlink. The Makefile resolution is what matters; mirror its logic rather than reading `$THEOS` directly.

12. **`nic.pl` is interactive.** For the "new project" command (#23 in init.md), drive it via stdin or rewrite the prompting in the extension and call `nic.pl` non-interactively. Look at how others (e.g. theos-jailed, dragoma) script it.

13. **Xcode SDK paths are not stable.** Always resolve via `xcrun --sdk <name> --show-sdk-path`. Hardcoded `/Applications/Xcode.app/...` breaks for users on Xcode-beta or non-default install locations. Cache the result for the session.

14. **`.tbd` stubs vs full headers.** Apple's SDKs are full of `.tbd` (text-based dylib) files. These are *not* headers — they are link-time stubs. Don't try to read symbols from them; only consume `.h` files. Some private frameworks ship only `.tbd` with no headers — those classes will appear unresolvable, which is correct behavior.

15. **Inside-SDK detection is by path walk.** A file at `/Applications/Xcode.app/.../iPhoneOS.sdk/System/Library/Frameworks/Foundation.framework/Headers/NSString.h` is "inside" the iPhoneOS SDK. Walk parents looking for `*.sdk/` or an adjacent `SDKSettings.plist`. This is how transitive navigation through Apple's headers works without configuration.

16. **VS Code marks files outside the workspace read-only by default.** Good — we want that. But clangd will still index them; make sure file watchers don't try to write back to them.

17. **Theos vendor PrivateFrameworks coverage is incomplete.** Many private classes have no header anywhere. Go-to-def will fail for them, which is expected. Don't show a scary error — just silently fail or show "no definition found." Optionally, fall back to a class-dump-on-demand for `%hook ClassName` cases (Phase 3 feature, item #15 in init.md).

18. **We do not write an indexer; clangd does.** What we *do* write is the **SDK index bootstrap** — the synthetic-TU machinery that induces clangd to index the SDK headers in the first place. See [SDK index bootstrap](#sdk-index-bootstrap) below. The first-run cost (minutes of CPU, ~100 MB of disk) lands on clangd, not on us. Our responsibility is correctness of the synthetic compile commands and good UX around progress and cancellation.

## Suggested phases

Each phase ends in a shippable extension. Don't move on until the current one is solid.

**Phase 1 — language basics (no clangd yet)**
- TextMate grammar for `.x` / `.xm` / `.xi` / `.xmi` (#7)
- Logos keyword completion (#3)
- Logos snippets (#11)
- Block-pair validation: unclosed `%hook` / `%group` / `%subclass` (#8)
- Doc-hover for Logos keywords linking to theos.dev (#26)
- Theos project detection (#16)
- Works on *any* opened file regardless of workspace (the standalone-mode floor — see capability matrix above for what's "yes" in the "No project" column)

**Phase 2 — clangd integration (standalone first, then project)**
- Spawn clangd as a child process. Single clangd instance, even in standalone mode.
- **2a — Standalone path resolution.** Synthesize fallback compile commands per the recipe in [Standalone behavior](#standalone-no-project-behavior). Implement `#import` cmd-click for all four file-in-scope cases (loose `.x`, workspace `.h`, transitive `.h`, inside-SDK `.h`). Implement inside-SDK detection. Persistent SDK index at `~/.cache/logos-lsp/clangd-index/`. Go-to-def on SDK types must work with no project. (#1, partial #2, #13)
- **2b — Project layer.** When `compile_commands.json` is present, prefer it over the synthesized fallback. Run `logos.pl` to produce `.m` for `.x`/`.xm` files; feed to clangd. Proxy `textDocument/publishDiagnostics` with `.m` → `.x` path swap (line numbers come from `#line` for free). Proxy hover, completion, go-to-def for project ObjC code. Project symbol completion (#2 full, #13).

Standalone behavior must work end-to-end at the end of 2a, before any project-mode features land. This is non-negotiable: a user should be able to install the extension, open a single `.x` file, and get useful import resolution and SDK type navigation without any further configuration.

**Phase 3 — Logos semantics**
- `%hook` aware completion: show methods of the hooked class only (#2)
- Hover on `%hook ClassName` shows the real `@interface` (#10)
- Semantic diagnostics: `%orig` in `%new`, unknown method on class, etc. (#9)
- Auto-import missing headers (#14)
- Generate `@interface` stub from class-dump (#15)

**Phase 4 — Makefile + build**
- Makefile variable completion (#4, #19)
- Validate `control` and `Filter.plist` (#18)
- VS Code tasks for `make`, `make do`, `make package`, `make install` (#17)
- "New Theos Project" command wrapping `nic.pl` (#23)

**Phase 5 — debug + device**
- SSH device manager (#20)
- LLDB-based debugger integration (#6) — probably via `vadimcn.vscode-lldb` as a backend
- Live tweak log viewer (#21)
- Crash log symbolication (#22)

**Phase 6 — refactor + polish**
- Inline `.m` expansion preview (#12)
- Rename refactor across `%hook` blocks (#24) — this is the first feature that needs column-precise reverse mapping
- Format on save (#25)

## Concrete first task for the next agent

1. Initialize a Node + TypeScript VS Code extension scaffold in this repo (`npm init`, `yo code`-style structure, `package.json` with `contributes.languages` for `logos`).
2. Build the TextMate grammar for `.x` files. Reference [theos/vendor/logos/extras/vim/](../theos/vendor/logos/extras/vim/).
3. Wire up a minimal LSP (language-server-protocol over stdio) that publishes the unclosed-`%hook` diagnostic. This proves the plumbing.
4. **Verify the standalone contract early.** Open a `.x` file in a folder with no Makefile and confirm Phase 1 features all work. If anything in the "No project" column of the capability matrix doesn't work, fix it before moving on.
5. Stop and check in with the user before starting Phase 2 — clangd integration is the architectural commitment and worth a checkpoint. Specifically flag the Phase 2a / 2b split for review.

## Reference paths

- Spec: [init.md](init.md)
- Theos clone (read-only): `~/Developer/theos`
- Logos sources: `~/Developer/theos/vendor/logos/bin/`
- Logos main script: [theos/vendor/logos/bin/logos.pl](../theos/vendor/logos/bin/logos.pl)
- Theos Makefile rules: [theos/makefiles/instance/rules.mk](../theos/makefiles/instance/rules.mk)
- compile_commands rule: [theos/makefiles/master/rules.mk:226](../theos/makefiles/master/rules.mk#L226)
- Templates: [theos/vendor/templates/](../theos/vendor/templates/)
