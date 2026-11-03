# Voicemeeter Windows Volume

Tray app that allows you to sync windows volume and mute state to Voicemeeter volume controls. Made in web technologies. For kicks.

The advantage of this application is that you do not need to fiddle with custom macro keys and related workarounds. Any existing methods to change Windows volume (keyboard keys, software, hardware controls, headset controls, etc) will "just work" and retain your system's volume OSD functionality.

## How To Install

1. Head over to the [releases](https://github.com/Frosthaven/voicemeeter-windows-volume/releases/) page to download the latest version.
2. Download the installer and run it.

## Features

-   `Bind Windows Volume To`
    -   These are the inputs and outputs detected in Voicemeeter. When you change Windows volume, that volume will be synced to any and all checked entries in this list.
-   `Settings`
    -   `Automatically Start With Windows`
        -   Registers a scheduled task so that the tray applet launches on user login (with high enough privelages to operate) by default.
    -   `Limit Max Gain To 0dB`
        -   Windows volume will max out Voicemeeter to 0dB instead of +12dB
    -   `Use Linear Volume Scaling`
        -   Substitutes the logorithmic volume scaling out for simplified linear scaling
    -   `Sync Mute`
        -   If enabled, the mute state of Windows will be synced along with the volume
    -   `Restore Volume At Launch`
        -   This setting allows the app to remember your volume changes while it is open and restore the last known volume on launch. This is useful for users with drivers that reset the volume on startup/reboot.
    -   `Prevent 100% Volume Spikes`
        -   This will detect a driver/Windows issue where devices are reset to 100% audio on audio engine restarts or audio device changes. _Note: This comes with a side effect. Enabling this will prevent the system from ever instantly reaching 100% volume. You can still use volume sliders and keys to reach 100, however._
    -   `Apply Crackle Fix (USB Interfaces)`
        -   This will fix a common crackling and popping issue with software audio mixers and USB interfaces. It does this by limiting audiodg.exe to a single core and giving it high priority.

## Advanced Configuration

You can also edit some unlisted settings. After running the program at least once, the file `/required/settings.json` is created.

Close the program if it is running, and open that file in a text editor. Here are the options currently available:

-   `polling_rate` is how quickly (in milliseconds) the application will track changes in Windows volume.
-   `gain_min` is the level Voicemeeter will be if Windows volume is at 0
-   `gain_max` is the level Voicemeeter will be if Windows volume is at 100. Ignored if "Limit Max Gain To 0dB" is checked.
-   `disable_donate` if set to true, the donate link will be hidden from the tray app (donations are optional and nothing is locked behind them).
-   `audiodg` should be left alone unless you know what you are doing. It utilizes Windows priority and affinity codes to set both appropriately when "Apply Crackle Fix" is checked.
-   `toggles` this is an enumerated list of all checkable items in the tray applet.
-   `device_blacklist` this is an array of all devices to ignore when watching for device connections

## Debug / Verbose Output Mode

If you want to see nerdy console output, follow these steps:

- Go to the installation directory (Usually `C:\Program Files (x86)\Voicemeeter Windows Volume`)
- Make sure the program is not running (Exit it if it is)
- Double click to run `app-launcher-debug.vbs`

## Contributors

-   [metahexane](https://github.com/metahexane)
    -   Logarithmic volume calculation to replace flat scale conversion
-   [mkc1370](https://github.com/mkc1370)
    -   Added option to toggle whether the mute state is synced or not
-   [Mineral](https://github.com/Mineral)
    -   Inspiration for handling "Automatically Start With Windows" through powershell instead of .vbs files

# Bun Development Setup (Windows x64)

Dependency installation and native-module loading have been verified with Bun 1.4.4. Node.js is not needed for this workflow. Run these commands from the repository root:

```powershell
bun --bun install --frozen-lockfile
```

Keep `bun.lock`: it preserves the existing dependency versions and the pinned `voicemeeter-connector` commit. The old Git-package integrity values in `package-lock.json` were removed only for that connector because they do not match GitHub's source archive; Bun records its own archive integrity in `bun.lock`. Registry-package integrity checks remain enabled.

If an earlier installation left a partial dependency tree, reinstall the locked packages without deleting the repository or unrelated files:

```powershell
bun --bun install --frozen-lockfile --force
```

`--bun` forces lifecycle JavaScript to run with Bun. Do not use `--ignore-scripts` as an installation workaround. The Windows x64 prebuilds supplied by `ffi-napi` and `ref-napi` work with Bun; no Python/compiler installation or native source rebuild was needed. Bun may report two blocked `core-js` postinstalls; those print donation banners, not native build steps.

Verify real native calls and load the installed Voicemeeter DLL without logging in or changing volume:

```powershell
@'
const assert = require("node:assert/strict");
const ref = require("ref-napi");
const ffi = require("ffi-napi");
assert.equal(ref.alloc("int", 42).deref(), 42);
const kernel = ffi.Library("kernel32", { GetCurrentProcessId: ["uint32", []] });
assert.equal(kernel.GetCurrentProcessId(), process.pid);
console.log("Native PID:", process.pid, "Bun:", process.versions.bun);
require("voicemeeter-connector").Voicemeeter.init()
    .then(() => console.log("Voicemeeter DLL loaded"))
    .catch(error => { console.error(error); process.exitCode = 1; });
'@ | bun --no-install -
```

Voicemeeter must be installed for the DLL initialization check. This checks native loading, not an active mixer connection, tray interaction, or audio synchronization.

To launch the source directly with Bun, bypassing the legacy package scripts:

```powershell
bun --no-install ./src/index.js
```

The launch command runs the actual tray application and can change audio according to its settings. Full tray/audio behavior has not been verified by the dependency smoke checks.

# Legacy Release Packaging

The existing release-packaging scripts still target a bundled Node.js executable and invoke npm internally. They are not a Bun-only build workflow and were not exercised as part of Bun dependency setup.

## Legacy Requirements

-   [NodeJS](https://nodejs.org/) _(Currently on 16.19.0)_
-   [Python v3.6-v3.9](https://www.python.org/downloads/) _(Ensure the python binary and scripts folder are in the system PATH variable)_
-   [NASM](https://www.nasm.us/pub/nasm/releasebuilds/2.15.04/) _(Right click > Run as Administrator)_
-   [Windows Build Tools](https://visualstudio.microsoft.com/downloads/?q=build+tools) _(Scroll down until you reach the build tools download link)_
-   [NSIS v3.07+](https://nsis.sourceforge.io/Download) _(Manually add the NSIS folder that contains makensis.exe to the environmental PATH variable)_
-   [ImageMagick](https://imagemagick.org/script/download.php#windows)

Once you have the above tools installed as described, you will need to reboot your computer.

## Packaging Commands

See `package.json` for the existing `build` and `rebuild` scripts. They create the `_dist` release artifacts, but require the legacy toolchain above; invoking them through Bun does not remove their internal npm calls or the Node.js binary build.
