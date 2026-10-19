+++
title = "Fixing yabai's scripting addition on macOS 27 (Golden Gate)"
date = 2026-10-18
tags = ['yabai', 'macos', 'scripting-addition', 'dotfiles', 'skhd']
description = 'Applying the unmerged macOS 27 patch to a Homebrew --HEAD yabai install, and why the scripting addition broke in the first place.'
+++

*Applying the unmerged patch from [asmvik/yabai#2832](https://github.com/asmvik/yabai/issues/2832) to a Homebrew `--HEAD` install, and why it was needed in the first place.*

My dotfiles, including yabai and skhd config, live at [github.com/sventurbo/.dotfiles](https://github.com/sventurbo/.dotfiles).

## TL;DR

- On macOS 27, yabai's **scripting addition** (the part that reaches into the Dock process) silently did nothing. yabai itself ran fine, but `yabai -m space --focus 2` exited with status 0 and did not switch spaces.
- The cause is not one bug but several things changing at once: a version allow-list that rejects macOS 27, Dock byte patterns that no longer match, a changed function-prologue instruction in a second Dock slice, a renamed Dock method (in the 27.2 beta), and a removed class that moving spaces depended on.
- The fix is a two-commit third-party patch on top of yabai `dd84572`. For a Homebrew `--HEAD` install you build it from source, swap the binary into the Cellar, and then redo the three things tied to the binary's identity: the **sudoers hash**, the **Accessibility grant**, and the **scripting-addition load**.
- Result on macOS 27.0.1 (Apple M1): space focus works, and the patterns for creating and destroying spaces resolve. **Moving spaces is still unsupported** on macOS 27.

## Environment

| | |
|---|---|
| macOS | 27.0.1 (build 26A434) |
| Hardware | Apple M1 (uses the `arm64e` Dock slice) |
| yabai | `yabai-v7.1.25`, Homebrew `--HEAD` build `HEAD-dd84572` from the `asmvik/formulae` tap |
| Dock binary SHA-256 | `b704affba65f732ffd6676c3bb22c94abc737ee73af8bdf29a5ef02650cde33e` |
| Shell | fish (snippets below are POSIX shell for zsh/bash; see the note in step 7) |

The scripting addition requires System Integrity Protection to be partially disabled, as described in the [yabai wiki](https://github.com/asmvik/yabai/wiki/Installing-yabai-(latest-release)#configure-scripting-addition). This article assumes your scripting addition already worked on an earlier macOS release, so that part is already set up.

## Background: what the scripting addition is

yabai is a tiling window manager. Most of what it does uses public or semi-public APIs. A few things have no such API: switching spaces without the slide animation, creating and destroying spaces, moving a space to another display. For these, yabai ships a **scripting addition (SA)**:

1. A small *loader* injects a *payload* into the running **Dock** process. The Dock is the process that owns Mission Control and spaces.
2. Inside the Dock, the payload needs the addresses of a few *private, undocumented* functions and globals, such as the "add space" function and the object that holds all spaces.
3. Apple gives no stable symbols for these, so yabai **finds them by byte pattern**. For each target there is an offset to start at and a pattern of machine-code bytes, with `??` as a wildcard. The scan starts at `<Dock __TEXT base> + offset` and walks forward (at most `0x1286A0` bytes) until the pattern matches. Source: `hex_find_seq()` in `src/osax/payload.m`. If nothing matches, the feature is switched off.
4. The payload opens a Unix socket (`/tmp/yabai-sa_<user>.socket`). `yabai -m space --focus …` sends a message to it, and the payload calls the Dock internals.

The payload and loader are compiled for `x86_64` and `arm64e`, turned into C arrays with `xxd -i`, and **embedded in the yabai binary** itself. That is why replacing the `yabai` binary is enough to ship a new payload.

Every table of offsets and patterns is keyed by macOS major version (the `get_*_offset()` and `get_*_pattern()` functions in `src/osax/arm64_payload.m`). When Apple recompiles the Dock for a new major release, these tables go stale. Someone has to disassemble the new Dock and write new ones.

## What it looked like when broken

Per the upstream issue (observed on the macOS 27.2 beta; see the author note at the top):

- The payload loads into the Dock and opens its socket.
- It reports the capability bitmask (`attrib`) `0x00`, meaning no features resolved.
- `yabai -m space --focus` **exits 0 but nothing happens**. This is the reason it is hard to notice: the client only sends a message to the socket and has no way of knowing that the payload ignored it.

## Why: what changed in macOS 27

All of this is from the issue and the patch, which I read in full. Where I checked something myself on 27.0.1, I say so.

### 1. The version allow-list rejected macOS 27

`verify_os_version()` in the payload accepts only known major versions. It did not know 27, so it refused to resolve anything.

### 2. Every pattern and offset was stale

None of the Tahoe (26) patterns matched in the new Dock, and the 27.0 values collected in [#2802](https://github.com/asmvik/yabai/issues/2802) did not match the 27.2 beta either (per the issue).

### 3. The Dock is a fat binary with two `arm64e` slices

```text
$ lipo -detailed_info /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock
architecture arm64e     (cpusubtype CPU_SUBTYPE_ARM64E)
architecture arm64e.x1  (cpusubtype 0x8000000c)
```

I confirmed both slices exist on 27.0.1. Per the issue, the `x1` slice is selected on newer chips, and its functions start with the pointer-authentication instruction `pacibsppc` (`FE A7 C1 DA`) instead of the usual `pacibsp` (`7F 23 03 D5`). The old patterns were anchored on `7F 23 03 D5`, so they could never match in that slice. The new patterns wildcard the first instruction (`?? ?? ?? ??`), so one pattern covers both slices.

### 4. A Dock method was renamed (27.2 beta)

Per the issue, `-[Spaces currentSpaceForDisplayUUID:]` was replaced by `currentSpaceForDisplay:`, which takes a `CGDirectDisplayID` instead of the display UUID string. The patch uses the old selector if it exists and only falls back to `displayForSPID:` plus `currentSpaceForDisplay:` if it does not. I did not check which path my 27.0.1 Dock takes. Focus works either way.

### 5. `DPDesktopPictureManager` is gone

The wallpaper handling that yabai used to move spaces between displays moved to a different class that has no equivalent method. The patch leaves `dppm` and `move_space` unresolved on 27, so **moving spaces stays disabled**. The log shows this clearly (see "Verifying").

### 6. The client side needed to know about macOS 27 too

The `yabai` binary itself has many `workspace_is_macos_tahoe()` checks that pick OS-specific behavior (Mission Control observers, window notifications, the `SLSGetWindowSubLevel` Mach message id). Without a macOS 27 entry, those code paths do not run. Per the issue, the Tahoe message id `0x76E3` is also correct on 27.

### A separate, version-independent crash

While testing, the patch author found that destroying an unknown space id crashed the Dock (`CFEqual() called with NULL first argument`), because `SLSCopyManagedDisplayForSpace` can return NULL and several handlers did not check. The second commit adds those guards. It is independent of macOS 27 but worth having, because the code runs inside the Dock.

## The patch, file by file

Two commits by the issue's author, based on `asmvik/yabai@dd84572`:

- `9da3923176d88e6c44a7802fa7d6374ae40414f4`: `feat(osax): Support the scripting addition on macOS 27`
- `8135286ece81646f12e6afaf59667e29ae1e1ae9`: `fix(osax): Guard space handlers against unknown space ids`

| File | Change |
|---|---|
| `src/osax/arm64_payload.m` | Adds a `majorVersion == 27` branch to all 14 lookup functions. Real offsets and patterns for `dock.spaces`, `add_space`, `remove_space` and `animation_time`. `dppm`, `move_space` and `set_front_window` return `0`/`NULL`. |
| `src/osax/payload.m` | `verify_os_version()` accepts 27. New `current_space_for_display()` helper with the selector fallback. Second commit: NULL guards in the focus, create, destroy and move handlers. |
| `src/osax/common.h` | `OSAX_VERSION` goes from `2.1.30` to `2.1.31`. |
| `src/workspace.h` | Adds `goldengate` (major 27) to the supported-version list, which generates `workspace_is_macos_goldengate()`. |
| `src/yabai.c`, `event_loop.c`, `view.c`, `window.c`, `window_manager.c` | Every `… \|\| workspace_is_macos_tahoe()` gains `\|\| workspace_is_macos_goldengate()`. The `SLSGetWindowSubLevel` message id `0x76E3` is used for 27. |
| `CHANGELOG.md` | One line. |

Apart from the selector fallback and the NULL guards, the patch adds no new logic. It fills in the per-version tables and extends the existing Sequoia/Tahoe branches to cover 27.

## Fixing it, step by step

### 0. Be realistic about the risk

This code is injected into the Dock. The patch comes from a contributor's fork, **not from the maintainer**, and is not merged upstream at the time of writing. Read it before building it. Step 3 shows how.

### 1. Check that you are on the patch's base commit

```sh
yabai --version
brew list --versions yabai
grep '"tap"' /opt/homebrew/Cellar/yabai/*/INSTALL_RECEIPT.json
```

I had `yabai-v7.1.25`, `yabai HEAD-dd84572` and `"tap": "asmvik/formulae"`. The patch applies cleanly to `dd845723416f5fe92af49fad5ebab00369e07edd`. On another commit it may need `git apply --3way` or manual edits. (On an Intel Mac, replace `/opt/homebrew` with `/usr/local` everywhere.)

### 2. Download the patch, pinned by commit

The issue suggests fetching the branch's `compare/….patch` and piping it into `git am`. Don't. The branch can move, and you should not apply something you haven't read. Download it to a file, and pin the commits:

```sh
mkdir -p ~/src/yabai-macos27 && cd ~/src/yabai-macos27
curl -sSL -O https://github.com/LCS-Dev-Ergos/yabai/commit/9da3923176d88e6c44a7802fa7d6374ae40414f4.patch
curl -sSL -O https://github.com/LCS-Dev-Ergos/yabai/commit/8135286ece81646f12e6afaf59667e29ae1e1ae9.patch
cat 9da3923176d88e6c44a7802fa7d6374ae40414f4.patch 8135286ece81646f12e6afaf59667e29ae1e1ae9.patch > fix.patch
wc -c fix.patch    # about 25 KB
```

Keep a copy of `fix.patch` with your notes. If the fork disappears, the commits go with it.

### 3. Read it, and grep for anything that does not belong

The patch is about 25 KB. I read all of it. It is offset and pattern tables, version gates and NULL checks. Then a mechanical check on the added lines for process, network and file calls:

```sh
grep -nE '^\+[^+].*(system\(|popen|execv|execl|posix_spawn|NSTask|NSURL|curl|CFNetwork|connect\(|fopen|fwrite|unlink|remove\()' fix.patch
```

No output (grep exits 1) means none of those appear in the lines the patch adds. That is a sanity check, not a proof: it does not replace reading the diff.

### 4. Build it

This mirrors what the Homebrew formula does for `--HEAD` (`make -j1 install`, then an ad-hoc `codesign`). The `install` target only builds into `./bin`; it does not copy anything to your system.

```sh
git clone https://github.com/asmvik/yabai.git
cd yabai
git checkout dd845723416f5fe92af49fad5ebab00369e07edd
git apply --check ../fix.patch && git apply ../fix.patch
make -j1 install
codesign -fs - bin/yabai
./bin/yabai --version      # still yabai-v7.1.25; the patch does not change the version string
```

You need the Xcode command line tools, including an `arm64e`-capable `clang`.

### 5. Check the patterns against your own Dock *before* installing anything

The author tested on macOS 27.2 beta and says he did not test 27.0 or 27.1. So before swapping binaries, I checked whether the patch's patterns match the Dock on 27.0.1. The script in the appendix replays `hex_find_seq()` on the Dock file for both slices:

```sh
python3 -I scan_dock.py
```

My output on 27.0.1:

```text
== slice arm64e (file offset 0x4000, size 0x49e880)
  dock.spaces     FOUND at 0x30a98   (matches in whole slice: 2)
  add_space       FOUND at 0x228cbc   (matches in whole slice: 1)
  remove_space    FOUND at 0x18b9a0   (matches in whole slice: 1)
  animation_time  FOUND at 0x227508   (matches in whole slice: 1)
== slice arm64e.x1 (file offset 0x4a4000, size 0x48a700)
  dock.spaces     FOUND at 0x2fdc0   (matches in whole slice: 2)
  add_space       FOUND at 0x21a1dc   (matches in whole slice: 1)
  remove_space    FOUND at 0x181838   (matches in whole slice: 1)
  animation_time  FOUND at 0x218a70   (matches in whole slice: 1)
```

Everything is found inside the scan window in both slices. One caveat: `dock.spaces` matches twice across the whole slice. The payload takes the **first** match at or after its start offset, so what matters is which one is first. Here that is `0x30a98`, close to the `0x2f5ac` the issue reports for 27.2, while the second match is far away at `0x6df78`. That is consistent, but the runtime test below is the real proof. If your output shows `NOT FOUND`, stop: the patch will not work on your build as is.

### 6. Back up the original binary and swap in the new one

`/opt/homebrew/bin/yabai` is a symlink into the Cellar, and the Cellar file is read-only. `install` replaces it cleanly:

```sh
cp -p /opt/homebrew/Cellar/yabai/HEAD-dd84572/bin/yabai ~/yabai.orig
install -m 555 bin/yabai /opt/homebrew/Cellar/yabai/HEAD-dd84572/bin/yabai
shasum -a 256 /opt/homebrew/bin/yabai bin/yabai    # the two hashes must match
codesign --verify --verbose /opt/homebrew/bin/yabai
```

Your Cellar directory name depends on your commit (`readlink /opt/homebrew/bin/yabai` shows it).

### 7. Update the sudoers rule

If your `~/.yabairc` loads the scripting addition (mine has `sudo yabai --load-sa` and a `dock_did_restart` signal that runs the same), you have a passwordless rule like this:

```text
<user> ALL=(root) NOPASSWD: sha256:<hash> /opt/homebrew/bin/yabai --load-sa
```

The rule pins the binary's SHA-256, so **any rebuilt binary invalidates it**. A stale rule does not fail loudly. `sudo` simply asks for a password, which a launchd-started `yabairc` cannot answer, so the scripting addition silently does not load after a reboot or Dock restart.

Replace the hash in place and let `visudo` validate the result (POSIX shell; in fish use `set OLD (…)`):

```sh
OLD=$(sudo grep -o 'sha256:[0-9a-f]*' /private/etc/sudoers.d/yabai | cut -d: -f2)
NEW=$(shasum -a 256 /opt/homebrew/bin/yabai | cut -d' ' -f1)
sudo sed -i '' "s/$OLD/$NEW/" /private/etc/sudoers.d/yabai
sudo visudo -c
sudo grep -o 'sha256:[0-9a-f]*' /private/etc/sudoers.d/yabai    # should now show $NEW
```

I ran the same thing with both hashes pasted in literally. `visudo -c` must say `parsed OK` for `/private/etc/sudoers.d/yabai`. Using `sed` on the single hash is less error-prone than editing the file in `visudo`'s default editor, and it leaves every other line alone.

### 8. Load the new payload

Restart the Dock so the old payload is gone, then load the new one:

```sh
killall Dock; sleep 2; sudo yabai --load-sa
```

Note: in yabai 7.x there is **no `--install-sa`** (older guides still mention it). `yabai --help` says `--load-sa` does "Install and load the scripting-addition", and it is the only command you need.

### 9. Re-grant Accessibility, or yabai will refuse to start

After the swap, `yabai --restart-service` did not bring yabai back. Symptoms:

```text
$ cat /tmp/yabai_$USER.err.log
yabai: could not access accessibility features! abort..

$ yabai -m query --spaces
yabai-msg: failed to connect to socket..
```

macOS ties the Accessibility permission to the binary's code identity. A Homebrew `--HEAD` build is signed ad hoc (`codesign -fs -`), so a different build counts as a different program, and an old "yabai" row in the list can look enabled while matching nothing. The fix:

```sh
tccutil reset Accessibility com.asmvik.yabai
```

Then, in **System Settings → Privacy & Security → Accessibility**:

1. Delete any remaining `yabai` rows with **−**.
2. Click **+**, press **⌘⇧G**, and add the real file `/opt/homebrew/Cellar/yabai/HEAD-dd84572/bin/yabai` (the Cellar path, not the symlink).
3. Make sure its toggle is on.

I did both the reset and the re-add, so I cannot say which one alone would have been enough.

```sh
yabai --restart-service
sleep 3; pgrep -lf yabai
```

### 10. Confirm the scripting addition resolved its targets

The payload logs to the unified log. In zsh, `log` is a shell builtin, so call the binary by path:

```sh
/usr/bin/log show --last 10m --predicate 'eventMessage CONTAINS "yabai-sa"' --style compact
```

What I got on 27.0.1 (timestamps and process ids trimmed):

```text
[yabai-sa] loaded payload..
[yabai-sa] checking for macOS 27.0.1 compatibility!
[yabai-sa] (0x10412c000) dock.spaces found at address 0x409BB0 (0xfffffffefc2ddbb0)
[yabai-sa] could not locate pointer to dppm! moving spaces will not work!
[yabai-sa] (0x10412c000) addSpace found at address 0x104354CBC (0x228cbc)
[yabai-sa] (0x10412c000) removeSpace found at address 0x1042B79A0 (0x18b9a0)
[yabai-sa] failed to get pointer to moveSpace function..
[yabai-sa] failed to get pointer to setFrontWindow function..
[yabai-sa] (0x10412c000) animation_time_addr found at address 0x104353508 (0x227508)
[yabai-sa] now listening..
```

The addresses in parentheses match what the static scan predicted for the `arm64e` slice (`0x228cbc`, `0x18b9a0`, `0x227508`). The `dppm`, `moveSpace` and `setFrontWindow` failures are expected on macOS 27 and mean "not supported", not "broken".

### 11. The actual test

An exit status of 0 proves nothing here, because the *unpatched* command also exits 0. Check that the current space really changes:

```sh
yabai -m query --spaces --space | grep '"index"'
yabai -m space --focus 2
sleep 1.5
yabai -m query --spaces --space | grep '"index"'
yabai -m space --focus 1       # go back
```

On my machine the index went from 1 to 2 and back to 1.

I also use skhd for the hotkeys, and after the fix my daily skhd-driven space switching — focusing, creating, and deleting spaces — all worked right away. No reboot wait needed; it just worked.

### 12. Does it survive a reboot?

My Mac rebooted a few days later. The scripting-addition socket in `/tmp` (which is wiped at boot) was recreated at the boot timestamp and yabai was running. That points to the `yabairc` and sudoers path working without a password prompt.

## What does not work

- **Moving spaces** (`yabai -m space --move`, display moves): `dppm` and `move_space` are unresolved on macOS 27.
- **`set_front_window`** is unresolved too. Per the issue, it is only reached from code that is compiled out in this build, so nothing is lost.
- Anything the patch's author did not try, I have not tried either. See below.

## What I did not verify

- The broken, pre-patch behavior on 27.0.1. The symptom description comes from the upstream issue.

- The `arm64e.x1` slice at runtime. On my M1 the payload uses the `arm64e` slice; the `x1` slice I only checked statically with the scan script.
- Intel Macs, and which chips select the `x1` slice.
- Whether my Dock's `currentSpaceForDisplayUUID:` still exists (the patch handles both cases).

## Housekeeping and rollback

- **`brew upgrade` / `brew reinstall` overwrite the patched binary.** So would a release that includes the fix. When that happens, redo steps 7 to 9 (sudoers hash, scripting-addition reload, Accessibility grant), or rebuild.
- **Every rebuild changes the binary hash and signature identity.** Any rebuild means a new sudoers hash and a new Accessibility grant.
- **Patterns are fragile.** A later macOS update (27.1, 27.2, a beta) can move the targets again. If `dock.spaces` and friends stop being "found" in the log after an update, re-run the scan script on the new Dock to see which pattern missed.
- **Rollback:** copy `~/yabai.orig` back over the Cellar binary (`install -m 555 ~/yabai.orig /opt/homebrew/Cellar/yabai/HEAD-dd84572/bin/yabai`), or `brew reinstall --HEAD asmvik/formulae/yabai`. Either way, redo the sudoers hash and the Accessibility grant.
- Install the fix from a **pinned commit you have read**, never from a moving branch.

## Troubleshooting cheat sheet

| Symptom | Cause | Fix |
|---|---|---|
| `yabai -m space --focus` exits 0 and nothing happens | Scripting addition not resolving any target (unsupported OS, stale patterns) | Check the `[yabai-sa]` log lines; apply the patch |
| `yabai: '--install-sa' is not a valid option!` | Option was removed in yabai 7.x | Use `sudo yabai --load-sa` |
| `yabai: could not access accessibility features! abort..` and `failed to connect to socket..` | New binary, new code identity, so macOS dropped the Accessibility grant | `tccutil reset Accessibility com.asmvik.yabai`, re-add the binary, restart the service |
| `sudo yabai --load-sa` asks for a password / SA missing after reboot | Sudoers rule still has the old binary's SHA-256 | Update the hash (step 7) and run `sudo visudo -c` |
| `NOT FOUND` in the scan script | Patterns do not match your Dock build | Do not install; ask upstream or re-derive the pattern for your build |

## Credits and links

- Patch and analysis: the author of [asmvik/yabai#2832](https://github.com/asmvik/yabai/issues/2832) (GitHub: `LCS-Dev-Ergos`). Branch: [`fix/macos-27-scripting-addition`](https://github.com/LCS-Dev-Ergos/yabai/compare/master...fix/macos-27-scripting-addition).
- Earlier macOS 27.0 findings: [asmvik/yabai#2802](https://github.com/asmvik/yabai/issues/2802).
- yabai by [asmvik](https://github.com/asmvik/yabai). Scripting-addition setup: the [wiki section](https://github.com/asmvik/yabai/wiki/Installing-yabai-(latest-release)#configure-scripting-addition).
- My dotfiles, including yabai and skhd config: [github.com/sventurbo/.dotfiles](https://github.com/sventurbo/.dotfiles).

## Appendix: `scan_dock.py`

Replays the payload's pattern search on the Dock binary. Reads only; changes nothing. Run it with `python3 -I scan_dock.py`.

```python
#!/usr/bin/env python3
"""Check whether the macOS 27 yabai scripting-addition patterns match this Mac's Dock.

Mirrors the payload's hex_find_seq(): starting at <__TEXT base + offset>, scan
forward for at most 0x1286A0 bytes for a byte pattern in which "??" (or any
token starting with "?") is a wildcard. The payload uses the FIRST match.
"""
import re
import struct

DOCK = "/System/Library/CoreServices/Dock.app/Contents/MacOS/Dock"
SCAN_LIMIT = 0x1286A0

# (start offset, pattern) per target, copied from the macOS 27 branches of
# src/osax/arm64_payload.m in the patch.
TARGETS = {
    "dock.spaces":    (0x20000,  "?8 ?? ?? ?? 08 ?? ?? 91 00 01 40 F9 E2 03 13 AA ?? ?? ?? 94 ?? ?? ?? ?? 08"),
    "add_space":      (0x160000, "?? ?? ?? ?? E1 03 1E AA ?? ?? ?? 97 FE 03 01 AA FD 7B 05 A9 FD 43 01 91 F3 03 14 AA F5 03 00 AA 96 8E 43 F8"),
    "remove_space":   (0x120000, "?? ?? ?? ?? FF ?? ?? D1 FC ?? ?? A9 FA ?? ?? A9 F8 ?? ?? A9 F6 ?? ?? A9 F4 ?? ?? A9 FD ?? ?? A9 FD ?? ?? 91 ?? 03 03 AA F5 03 02 AA F4 03 01 AA"),
    "animation_time": (0x160000, "00 10 6A 1E A8 ?? ?? D1 ?? 01 ?? F8"),
}

SUBTYPES = {0x80000002: "arm64e", 0x8000000C: "arm64e.x1"}


def to_regex(pattern):
    out = b""
    for token in pattern.split():
        out += b"." if token.startswith("?") else re.escape(bytes([int(token, 16)]))
    return re.compile(out, re.S)


def slices(data):
    """Yield (label, file offset, size) for each slice of a (fat) Mach-O file."""
    if struct.unpack(">I", data[:4])[0] != 0xCAFEBABE:
        yield "thin", 0, len(data)
        return
    count = struct.unpack(">I", data[4:8])[0]
    for i in range(count):
        _cpu, sub, off, size, _align = struct.unpack(">5I", data[8 + 20 * i:28 + 20 * i])
        yield SUBTYPES.get(sub, f"subtype 0x{sub:x}"), off, size


def main():
    with open(DOCK, "rb") as f:
        data = f.read()
    for label, off, size in slices(data):
        print(f"== slice {label} (file offset 0x{off:x}, size 0x{size:x})")
        sl = data[off:off + size]
        for name, (start, pattern) in TARGETS.items():
            rx = to_regex(pattern)
            window = sl[start:start + SCAN_LIMIT + len(pattern)]
            hit = rx.search(window)
            total = sum(1 for _ in rx.finditer(sl))
            if hit:
                print(f"  {name:15s} FOUND at 0x{start + hit.start():x}   (matches in whole slice: {total})")
            else:
                print(f"  {name:15s} NOT FOUND in scan window")


if __name__ == "__main__":
    main()
```

The patterns above are for macOS 27. If a later macOS release moves the targets again, the new offsets and patterns will be in the corresponding `majorVersion == 27` (or newer) branch of `src/osax/arm64_payload.m`; copy them into `TARGETS`.
