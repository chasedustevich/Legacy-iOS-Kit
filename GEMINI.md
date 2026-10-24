# Legacy iOS Kit Developer Guide (GEMINI.md)

## 1. Overview & Architecture

[Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit) (formerly `iOS-OTA-Downgrader`) is a comprehensive, shell-based toolkit designed for managing, downgrading, restoring, jailbreaking, and saving blobs for 32-bit and 64-bit iOS devices. It supports both **macOS** and **Linux** across x86_64 and arm64 architectures.

### Key Components

- **`restore.sh`**: The central entrypoint and orchestration script. Handles argument parsing, device detection, environment validation, recovery/DFU mode transitions, exploit delivery (checkm8, kloader, limera1n, etc.), firmware extraction, patching, and flashing via `idevicerestore` or `futurerestore`.
- **`bin/`**: Platform-specific prebuilt binaries and utilities:
  - `bin/macos`: Precompiled universal/x86_64/arm64 binaries (e.g., `idevicerestore`, `irecovery`, `ipwnder`, `gaster`, `futurerestore_old`, `hfsplus`, `img4tool`, `jq`, `aria2c`, `zenity`).
  - `bin/linux`: Architecture-specific binaries (`arm64`, `x86_64`).
  - `bin/deps-macos.sh` & `bin/deps-linux.sh`: Upstream dependency fetch and build scripts.
- **`resources/`**:
  - `firmware/`: Firmware bundles and patch configurations (`FirmwareBundles/`).
  - `jailbreak/`: Jailbreak packages, patches, untether archives (`evasi0n`, `daibutsu`, `p0sixspwn`, `g1lbertJB`, etc.).
  - `manifest/`: BuildManifest plist files for OTA downgrade targets (e.g., iOS 6.1.3, 8.4.1, 10.3.3).
  - `patch/`: Binary patches (ASR patches, kernelcache patches, Odysseus components).
  - `sshrd/`: SSH ramdisk configurations, iBSS/iBEC patches, and IM4M manifests.
  - `modules/`: Modular helper scripts (e.g., `fourthree.sh`, `touch4-ios7-repairs.sh`).
- **`saved/`**: Working directory for persisted artifacts (saved SHSH blobs, baseband backups, dumped activation records, Python virtual environments).
- **`tmp$$/`**: Temporary per-run directory created under the script path for inflight operations, cleaned automatically on exit.

---

## 2. Shell Scripting & Compatibility Rules

### Bash Version Target
- **macOS Compatibility**: Must maintain strict compatibility with macOS's default `/bin/bash` (**version 3.2.57**).
- **Linux Compatibility**: Supports modern Bash 4.x and 5.x.
- **Forbidden Constructs (unless guarded by bash version checks)**:
  - Associative arrays (`declare -A` does not exist in Bash 3).
  - Case modification parameter expansion (`${var,,}` or `${var^^}`).
  - `mapfile` / `readarray`.
  - Negative array indexing (`${arr[-1]}`).
  - Globstar (`shopt -s globstar`).

### Scripting Conventions
- **Indentation**: 4 spaces per indent level. No raw tabs.
- **Conditionals**:
  - Use `[[ ... ]]` for string and filesystem checks.
  - Use `(( ... ))` for integer comparisons and arithmetic evaluation.
- **Quoting**:
  - Always quote variable expansions for paths and command arguments: `"$var"`, `rm -rf "$target"`.
  - Handle space-separated paths gracefully where applicable (though script warns when cloned into path with spaces).
- **Standard Output Helpers**:
  - Never use raw `echo` for user notifications; use established helper functions:
    - `log "..."`: Informational status logs (green).
    - `print "..."`: General messages or instructions (blue/bold).
    - `warn "..."`: Warnings (yellow).
    - `error "..." [details]`: Errors; outputs red message and exits script (`exit 1`).
    - `input "..."`: Prompts for user input (yellow).
    - `pause`: Prompts "Press Enter/Return to continue (or press Ctrl+C to cancel)".
- **Process Management & Traps**:
  - Always maintain cleanup hooks (`trap "clean" EXIT`, `trap "exit 1" INT TERM`).
  - Ensure child processes (`iproxy`, `darkhttpd`, `sshfs`, etc.) and PID variables (`$httpserver_pid`, `$iproxy_pid`, `$anisette_pid`) are cleaned up properly in `clean()` / `clean_sudo()`.

---

## 3. Development, Testing, & Verification

### Syntax Validation
Always verify bash syntax after any modification to `restore.sh` or module scripts:
```bash
bash -n restore.sh
bash -n resources/modules/*.sh
```

### Dry Run & Parameter Testing
When testing changes to CLI flag handling, device detection logic, or menu paths without a physical iOS device connected:
- Check flag definitions and help text:
  ```bash
  ./restore.sh --help
  ```
- Run with debug tracing:
  ```bash
  ./restore.sh --debug --no-device
  ```
- Simulate specific device parameters:
  ```bash
  ./restore.sh --no-device --device=iPhone5,1 --ecid=0x123456789ABC
  ```

---

## 4. Hardware Safety & Operational Constraints

1. **Root Execution Check**: `restore.sh` explicitly refuses to run as `root` directly (`$EUID == 0`) to prevent file permission corruption and unsafe global modifications. Sudo is requested conditionally when configuring `usbmuxd` or flushing daemons.
2. **Preserve User Artifacts**: Never delete or overwrite files in `saved/` (such as user SHSH blobs, basebands, or activation records) without user confirmation.
3. **Daemon Interactions**:
   - macOS: Finder/MobileDevice services (`AMPDevicesAgent`, `AMPDeviceDiscoveryAgent`, `MobileDeviceUpdater`) are paused during device operations to prevent interference and resumed on clean exit unless `--no-finder` is specified.
   - Linux: Custom `usbmuxd` handling restarts or stops the system daemon where required to ensure reliable communication.
