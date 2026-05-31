Logos is a regex-based preprocessor for objc method swizzling code.

https://theos.dev/docs/logos

I want a language server for logos, lets build one. I need your help making it as feature-complete as possible. It also needs to be wrapped up in a VS Code extension. They don't have to be the same repo in the end, but for now there is no need to separate them, and I will speak in terms of what I features I want the extension to enable in Logos development. I have some ideas I will present first as requirements:

The extension should enable the user to…

1. command-click any import statement to open the relevant header
    1. even for headers outside the project, such as if you click `#import <Foundation/Foundation.h>` and it opens `Foundation.h` and you want to click on another header from there
2. autocomplete class and method names, including `%new` methods, and jump to symbol/method/class
    1. includes right click → jump to definition as well as "go to symbol"
    2. this requires indexing types and methods within the project
3. autocomplete logos keywords
4. autocomplete rules and resources in the Makefile
5. show compile errors inline as-you-type like a real IDE would, assuming you have a Makefile setup that compiles the current file
    1. this may require an entire build server or something really complicated
6. first class support for using VS Code's debugger to debug any Theos project on a target device
    1. also support launching or attaching to arbitrary apps and processes

## Additional ideas

### Logos-specific language features

7. Syntax highlighting for all Logos directives (`%hook`, `%end`, `%orig`, `%new`, `%log`, `%ctor`, `%dtor`, `%init`, `%c`, `%group`, `%subclass`, `%property`, `%config`).
8. Block structure validation: detect unclosed `%hook`/`%group`/`%subclass`, auto-pair `%end`, and fold blocks in the gutter.
9. Semantic diagnostics unique to Logos:
    - `%orig` used inside a `%new` method (no original exists)
    - Hooking a method that doesn't exist on the target class (with a "did you mean…" quick fix, or a quick fix to convert to `%new`)
    - Selector signature mismatch vs. the real method (wrong arg count/types)
    - `%init` called outside `%ctor` / before `%group` is declared
10. Hover info: show the real Objective-C method signature being hooked, where it came from (which header/framework), and a preview of what the Logos block expands to.
11. Snippets for common scaffolds (`%hook ClassName … %end`, `%new` method with proper type encoding, `%ctor` blocks, `%subclass` with `%property`).
12. Inline expansion preview — show what code Logos generates for the current block (helps users learn what `%orig` and `%new` actually do).

### Header / symbol indexing

13. SDK + private framework indexing (iOS SDK, PrivateFrameworks dumps via `theos/sdks` or class-dump headers) so completion works for Apple's own classes, not just project-local ones.
14. Auto-import missing headers when you reference an unknown class.
15. Generate `@interface` stub for a hooked class — pull from class-dump output or runtime introspection so private methods become known to the indexer.

### Theos / Makefile integration

16. Detect Theos project root (presence of `theos`/`$THEOS` and a tweak `Makefile`) and surface it as a workspace concept.
17. Tasks/commands for `make`, `make do`, `make package`, `make install`, with device picker (pairs with #6).
18. Validate `control` file and `Filter.plist` — bundle ID format, required fields, architecture lists.
19. Variable completion for known Theos Makefile variables (`TARGET`, `ARCHS`, `INSTALL_TARGET_PROCESSES`, `_FILES`, `_FRAMEWORKS`, `_LIBRARIES`, `_CFLAGS`, etc.) with hover docs.

### Device / runtime tooling (extends #6)

20. SSH device manager: configure, test, and pick devices; reuse for install + debug.
21. Live tweak log viewer — tail `oslog`/`syslog` filtered to the tweak's process.
22. Crash log symbolication for installed tweaks.

### Project scaffolding

23. Wrap `nic.pl` in a "New Theos Project" command (tweak, app, tool, prefs, library templates).

### Editor niceties

24. Rename refactor for hooked methods/classes across all `%hook` blocks and `%c()` references.
25. Format on save (delegating ObjC bodies to clang-format, preserving Logos directives).
26. Doc hover for Logos keywords linking to the relevant section on theos.dev.
