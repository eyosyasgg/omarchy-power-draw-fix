[README.md](https://github.com/user-attachments/files/33494740/README.md)
# 0.0W NVIDIA dGPU (`D3cold`) on Omarchy 4 & Hyprland (Hibernation-Safe)

Drop your hybrid gaming laptop's idle battery discharge from **13W–27W down to 6.5W–8.5W** on **Omarchy 4 (Arch Linux + Hyprland Lua config)** by putting the NVIDIA discrete GPU into deep hardware power-off (**ACPI `D3cold` / `0.0W`**)—**without** blacklisting kernel modules, breaking `systemctl hibernate`, or risking a login loop.

Tested on an **HP Victus 15 (`Intel Core i5-12450H` + `NVIDIA GeForce GTX 1650 Mobile TU117M`)** running **Omarchy 4.0.4 (`Hyprland v0.56.2` / `Aquamarine 0.15.0` / `NVIDIA Open Kernel Module`)**.

---

## Why Naive "Turn Off NVIDIA" Guides Fail on Omarchy & Hyprland

Most Linux guides tell you to either blacklist `nvidia` modules or delete the PCIe device via `udev` (`ATTR{remove}="1"`). On modern hybrid laptops running Omarchy 4 and `systemd` hibernation, those approaches cause three major problems:

1. **The Unbound Driver Trap (`8W–12W` Leak):** Blacklisting `nvidia` and `nouveau` without transitioning the PCIe root port to `D3cold` leaves the GPU sitting uninitialized in **ACPI `D0`**, silently burning ~10W of battery with no way to enter sleep.
2. **Broken Hibernation (`S4` Resume Failure):** When waking from hibernation (`suspend-to-disk`), the laptop performs a full cold BIOS POST, which powers the NVIDIA GPU back **ON into `D0`**. If `nvidia-hibernate.service`, `nvidia-resume.service`, and `NVreg_PreserveVideoMemoryAllocations=1` are bypassed or the PCIe node was removed before sleep, the GPU either fails to freeze (`-EIO`) or wakes up stuck in `D0` after resume.
3. **The 4 Hidden `D0` Blockers on Omarchy 4:**
   Even when NVIDIA's native Fine-Grained Runtime D3 (`NVreg_DynamicPowerManagement=0x02`) is enabled in the kernel, four user-space and `sysfs` blockers prevent the GPU from ever entering `D3cold`:
   * **Blocker 1 (`sysfs power/control = on`):** By default, `/sys/bus/pci/devices/0000:01:00.0/power/control` is set to `on`, forbidding the kernel from runtime-suspending the PCIe function.
   * **Blocker 2 (Omarchy's `default/hypr/nvidia.lua`):** When Omarchy detects vendor `0x10de` in `/sys/bus/pci/devices`, it automatically exports `LIBVA_DRIVER_NAME=nvidia`, `__GLX_VENDOR_LIBRARY_NAME=nvidia`, and `NVD_BACKEND=direct`. This forces Chrome, Electron (Obsidian, Discord), VS Code, and Xwayland to hold open file descriptors on `/dev/nvidia0` and `/dev/nvidia-uvm`.
   * **Blocker 3 (DRM Card Enumeration):** Hyprland's Aquamarine backend enumerates both `/dev/dri/card0` (NVIDIA) and `/dev/dri/card1` (Intel), holding open `/dev/dri/card0` and `/dev/dri/renderD129` unless restricted via `AQ_DRM_DEVICES`.
   * **Blocker 4 (The `nvidia-smi` / Daemon Wake-Loop):** Status bar plugins (such as `nenadjokic.nvidia-hybrid`) or hardware daemons (`omen-space-daemon`) that run `nvidia-smi` or query NVML every 2 seconds force the motherboard to cold-boot the sleeping GPU from `D3cold -> D0`, causing **25W–30W power spikes** every few seconds.

---

## Results (HP Victus 15 — i5-12450H + GTX 1650)

| Configuration State | PCIe Root Port (`00:01.0`) | GTX 1650 (`01:00.0`) | Open `/dev/nvidia*` Handles | Total Laptop Battery Draw |
| :--- | :--- | :--- | :--- | :--- |
| **Default Omarchy 4 Boot** | `D0` (`active`) | `D0` (`active`) | `Hyprland`, `chrome`, `electron`, `code`, `omen-space-daemon` | **12.5W – 14.3W** |
| **RTD3 Active + Bar Plugin Polling `nvidia-smi`** | Bounces `D3cold <-> D0` every 2s | Bounces `D3cold <-> D0` every 2s | `nvidia-smi` spawned every 2s | **Spikes to 27.1W** |
| **This Setup (Locked `D3cold`)** | **`D3cold`** | **`D3cold` (`suspended`)** | **`0` (None)** | **6.5W – 8.5W** |

---

## Quick Start (Automated Setup Script)

Clone this repository and run the setup script as your normal user:

```bash
git clone [https://github.com/](https://github.com/)<your-username>/<your-repo>.git
cd <your-repo>
chmod +x setup-nvidia-0w.sh
./setup-nvidia-0w.sh apply
```

Then reboot (`sudo reboot`) and verify your GPU is in `D3cold`:

```bash
./setup-nvidia-0w.sh watch
```

### Script Commands
* `./setup-nvidia-0w.sh apply` — Installs the `udev` rule, verifies `RTD3` kernel options, appends the fail-safe Lua block to `~/.config/hypr/hyprland.lua`, and checks for wake pollers.
* `./setup-nvidia-0w.sh status` — Displays current `D0`/`D3cold` states, open file handles on `/dev/nvidia*`, guard status, and live battery draw.
* `./setup-nvidia-0w.sh watch` — Samples PCIe power states and battery wattage every 2 seconds for 12 seconds without waking the GPU.
* `./setup-nvidia-0w.sh revert` — Removes all rules and Lua overrides and restores default hybrid behavior.

---

## Manual Step-by-Step Guide

If you prefer to configure every file manually instead of running the script, follow the steps below.

### Step 1: Verify Kernel Parameters for Fine-Grained RTD3

Ensure your kernel command line (`/proc/cmdline`) or `/etc/modprobe.d/nvidia.conf` includes NVIDIA's Dynamic Power Management (`0x02` = fine-grained) and video memory preservation for hibernation:

```bash
cat /proc/driver/nvidia/params | grep -E "DynamicPowerManagement|PreserveVideoMemory"
```

Expected output:
```text
PreserveVideoMemoryAllocations: 1
DynamicPowerManagement: 2
```

*(If `DynamicPowerManagement` is not `2`, create `/etc/modprobe.d/nvidia-rtd3.conf` with `options nvidia NVreg_DynamicPowerManagement=0x02 NVreg_PreserveVideoMemoryAllocations=1`.)*

---

### Step 2: Enable PCIe Runtime Power Management via `udev`

Create `/etc/udev/rules.d/80-nvidia-pm.rules` so both the NVIDIA VGA/3D controller (`0x03xxxx`) and NVIDIA HDMI Audio controller (`0x040300`) have `power/control` set to `auto`.

> **Important Pitfall:** Always include `change` in `ACTION=="add|bind|change"`. If you only match `add|bind`, running `sudo udevadm trigger` (which defaults to `--action=change`) or unplugging AC power will fail to apply the rule.

```bash
sudo tee /etc/udev/rules.d/80-nvidia-pm.rules << 'EOF'
# Enable Runtime D3 (D3cold) power management for NVIDIA VGA/3D and Audio devices
ACTION=="add|bind|change", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x03[0-9]*", ATTR{power/control}="auto"
ACTION=="add|bind|change", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x040300", ATTR{power/control}="auto"
EOF

sudo udevadm control --reload
sudo udevadm trigger --action=change --subsystem-match=pci --attr-match=vendor=0x10de
```

Verify `control` switched from `on` to `auto`:
```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/control
# Expected output: auto
```

---

### Step 3: Add the Fail-Safe iGPU Block to `~/.config/hypr/hyprland.lua`

#### Critical Technical Lessons for Omarchy 4 + Hyprland (`Aquamarine`)
1. **Never pass unescaped `/dev/dri/by-path/pci-0000:00:02.0-card` to `AQ_DRM_DEVICES`:**
   Aquamarine uses the colon (`:`) as a delimiter between multiple DRM devices. Because `pci-0000:00:02.0-card` contains two colons inside the PCI address, Aquamarine splits the path into `/dev/dri/by-path/pci-0000`, `00`, and `02.0-card`, fails to find any GPU, and crashes Hyprland on login with `what(): CBackend::create() failed!`.
   **Solution:** Have Lua inspect `/sys/bus/pci/devices/0000:00:02.0/drm/` at startup and pass the colon-free `/dev/dri/cardX` path (`/dev/dri/card1` or `/dev/dri/card0`) to `AQ_DRM_DEVICES`.
2. **Use a Self-Disarming Boot Guard (`~/.cache/igpu-mode-armed`):**
   Because Btrfs root snapshots (`@`) do not roll back `/home` (`@home`), a bad edit in `~/.config/hypr/hyprland.lua` persists across snapshots. Wrapping the override in a guard file that is immediately deleted via `os.remove()` *before* `CBackend::create()` and only re-created via `o.exec_on_start()` *after* Hyprland finishes launching guarantees you can never be stuck in a login loop.

Append the following block to the end of `~/.config/hypr/hyprland.lua`:

```lua
-- >>> OMARCHY-IGPU-0W-BEGIN <<<
-- Fail-safe Intel iGPU-only mode (auto-disarms before CBackend::create if startup ever fails)
local igpu_guard = (os.getenv("HOME") or "/home/eyos") .. "/.cache/igpu-mode-armed"
local gf = io.open(igpu_guard, "r")
if gf then
  gf:close()
  os.remove(igpu_guard)
  o.exec_on_start("touch " .. o.shell_quote(igpu_guard))

  for _, card in ipairs({ "card0", "card1", "card2" }) do
    local cf = io.open("/sys/bus/pci/devices/0000:00:02.0/drm/" .. card .. "/dev", "r")
    if cf then
      cf:close()
      hl.env("AQ_DRM_DEVICES", "/dev/dri/" .. card)
      break
    end
  end

  hl.env("LIBVA_DRIVER_NAME", "iHD")
  hl.env("__GLX_VENDOR_LIBRARY_NAME", "mesa")
  hl.env("__EGL_VENDOR_LIBRARY_FILENAMES", "/usr/share/glvnd/egl_vendor.d/50_mesa.json")
  hl.env("VK_DRIVER_FILES", "/usr/share/vulkan/icd.d/intel_icd.json")
  hl.env("NVD_BACKEND", "")
end
-- >>> OMARCHY-IGPU-0W-END <<<
```

Arm the guard before rebooting:
```bash
touch ~/.cache/igpu-mode-armed
```

---

### Step 4: Remove or Stop `nvidia-smi` Wake Pollers

Any status bar widget or background daemon that runs `nvidia-smi` or opens `/dev/nvidia0` on a timer will wake the GPU out of `D3cold` every few seconds, causing 25W–30W battery spikes while the GSP firmware cold-boots.

1. **Omarchy `nenadjokic.nvidia-hybrid` Quickshell Plugin:**
   This plugin polls `nvidia-smi --query-gpu=...` every 2000ms. Remove or disable it:
   ```bash
   rm -rf ~/.config/omarchy/plugins/nenadjokic.nvidia-hybrid
   ```
2. **HP `omen-space-daemon` (If Installed):**
   If `sudo fuser -v /dev/nvidia*` shows `omen-space-daem` holding `/dev/nvidia0` open, stop or disable it when on battery:
   ```bash
   sudo systemctl stop omen-space-daemon.service
   ```

---

### Step 5: Reboot and Verify `D3cold` (`0.0W`)

> **Note:** Do **not** run `nvidia-smi` or `lspci` to check if the GPU is asleep! Reading PCI configuration space or opening `/dev/nvidiactl` physically wakes the GPU into `D0` for several seconds. Always read cached kernel `sysfs` files instead.

Run this verification loop after rebooting and unplugging the charger:

```bash
echo "=== 1. Open Handles on NVIDIA (Should be empty) ==="
sudo fuser -v /dev/nvidia* /dev/dri/by-path/*01:00.0* 2>&1

echo -e "\n=== 2. 12-Second D3cold & Battery Monitor ==="
for i in {1..6}; do
  sleep 2
  echo "t+$((i*2))s -> 01:00.0 state: $(cat /sys/bus/pci/devices/0000:01:00.0/power_state) | status: $(cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status) | 00:01.0 state: $(cat /sys/bus/pci/devices/0000:00:01.0/power_state) | Bat: $(awk '{print $1*1e-6 " W"}' /sys/class/power_supply/BAT*/power_now)"
done
```

Expected output:
```text
t+2s  -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 6.872 W
t+4s  -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 6.999 W
t+6s  -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 7.565 W
t+8s  -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 7.253 W
t+10s -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 7.253 W
t+12s -> 01:00.0 state: D3cold | status: suspended | 00:01.0 state: D3cold | Bat: 6.953 W
```

---

## Running Specific Apps on the NVIDIA dGPU On-Demand (PRIME Offload)

Because the `nvidia`, `nvidia_uvm`, `nvidia_modeset`, and `nvidia_drm` kernel modules remain loaded, you do **not** need to reboot to use the GTX 1650 for CUDA, local LLMs (`llama.cpp`), or games:

* **For CUDA / Compute (`llama.cpp`, PyTorch):** Simply launch your compute job. Opening `/dev/nvidia-uvm` wakes the GPU into `D0` automatically, and the card returns to `D3cold` (`0.0W`) a few seconds after the process exits.
* **For OpenGL / Vulkan Apps:** Launch the application with PRIME render offload environment variables:
  ```bash
  __NV_PRIME_RENDER_OFFLOAD=1 \
  __GLX_VENDOR_LIBRARY_NAME=nvidia \
  __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json \
  VK_DRIVER_FILES=/usr/share/vulkan/icd.d/nvidia_icd.json \
  <command>
  ```
* **To Switch the Whole Desktop Session Back to Hybrid/External HDMI Mode:**
  Remove the guard file (`rm ~/.cache/igpu-mode-armed`) and log out/reboot.

---

## Emergency Recovery Reference (Limine Bootloader Rescue Shell)

If a manual edit to `~/.config/hypr/hyprland.lua` ever prevents Hyprland from starting and `Ctrl + Alt + F3` is unavailable due to `plymouth` (`quiet splash`), you can access a root maintenance shell directly from the **Omarchy (Limine) Bootloader** without a live USB:

1. At the **Omarchy Bootloader** menu, highlight `linux-omarchy` and press `e` to edit the entry.
2. Move the cursor to the end of the `cmdline:` line and backspace everything back to `initramfs_async=0` (removing `quiet splash loglevel=0 systemd.show_status=false ...`).
3. Append `systemd.unit=rescue.target` after `initramfs_async=0`:
   ```text
   ... resume=/dev/mapper/omarchy_root initramfs_async=0 systemd.unit=rescue.target
   ```
4. Press `F10` or `Ctrl + X` to boot, enter your LUKS disk encryption passphrase, and then enter your root password at the `Give root password for maintenance` prompt.
