# Cosmic Byte Blitz Controller Integration (Linux)

> **⚠️ DISCLAIMER: Tested Configuration**  
> This guide is specifically tested on the **Cosmic Byte Blitz** controller. While the strategies described here are generally applicable to other generic "Shanwan" chipset controllers that fail to enumerate correctly, your mileage may vary. Proceed with caution when applying system-level changes.

## Introduction
This repository documents the primary strategy for resolving the common issue where the Cosmic Byte Blitz controller connects as a generic device without vibration support on Linux. The goal is to correct the USB enumeration process to ensure the device is detected in a mode that supports vibration.

---

## Strategy 1: The "Kernel Param" Fix (Recommended)
**Effect / Outcome:** The controller will be detected as a **Nintendo Switch Pro Controller**. While this enables vibration, the button layout (A/B and X/Y) will be **inverted** relative to the Xbox physical layout. This guide includes manual steps to fix this mapping.

This method modifies the kernel boot arguments to use an older, more lenient USB enumeration scheme (`usbcore.old_scheme_first=1`). This allows the controller's firmware enough time to handshake correctly without timing out and defaulting to a generic Android mode.

### 1. Apply the Kernel Parameter
Choose the instructions below that match your Linux distribution.

#### Option A: Fedora / RHEL / CentOS (Using `grubby`)
Fedora uses `grubby` to easily manage kernel arguments.

```bash
# 1. Apply the fix to all installed kernels
sudo grubby --update-kernel=ALL --args="usbcore.old_scheme_first=1"

# 2. Verify the argument was added
sudo grubby --info=DEFAULT | grep args
```

#### Option B: Ubuntu / Debian / Arch / Generic (Using GRUB)
Most distributions use the standard GRUB configuration file.

1.  Open the GRUB configuration file:
    ```bash
    sudo nano /etc/default/grub
    ```
2.  Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` or `GRUB_CMDLINE_LINUX`.
3.  Add `usbcore.old_scheme_first=1` to the existing parameters inside the quotes.
    *   **Example Before:** `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`
    *   **Example After:** `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash usbcore.old_scheme_first=1"`
4.  Save and exit (Ctrl+O, Enter, Ctrl+X).
5.  Update GRUB:
    *   **Ubuntu/Debian/Mint:**
        ```bash
        sudo update-grub
        ```
    *   **Arch Linux:**
        ```bash
        sudo grub-mkconfig -o /boot/grub/grub.cfg
        ```
    *   **Fedora (Alternative):**
        ```bash
        sudo grub2-mkconfig -o /boot/grub2/grub.cfg
        ```

**❗ REBOOT REQUIRED:** You must reboot your system for this change to take effect.

---

### 2. Fix Button Mapping (Inversion)
Because the controller now identifies as a Nintendo Switch Pro Controller, the physical layout is swapped:
- Physical **A** acts as **B**
- Physical **X** acts as **Y**

To fix this, we use `input-remapper`.

#### Step 2.1: Install Input Remapper

*   **Fedora:** `sudo dnf install input-remapper`
*   **Ubuntu/Debian:** `sudo apt install input-remapper` (or via PPA)
*   **Arch Linux:** `yay -S input-remapper-git`
*   **Generic:** `pip install input-remapper`

Enable the service:
```bash
sudo systemctl enable --now input-remapper
```

#### Step 2.2: Configure the Mapping
1.  Open **Input Remapper** (`input-remapper-gtk`).
2.  Select the device labeled **Nintendo Switch Pro Controller**.
3.  Click **New Preset** and name it `Xbox Emulation`.
4.  Create the following mappings:
    *   **Recommended:** Simply bind the physical buttons to their Xbox equivalents.
    *   Map input `BTN_EAST` (Physical A) -> output `BTN_SOUTH` (Xbox A).
    *   Map input `BTN_SOUTH` (Physical B) -> output `BTN_EAST` (Xbox B).
    *   Map input `BTN_NORTH` (Physical X) -> output `BTN_WEST` (Xbox X).
    *   Map input `BTN_WEST` (Physical Y) -> output `BTN_NORTH` (Xbox Y).
5.  Click **Apply** and ensure "Autoload" is checked.

Alternatively, if you are only using Steam, you can enable "Nintendo Switch Support" in **Steam Settings > Controller** and check "Use Nintendo Button Layout" to have Steam handle this automatically.

---

## Additional Fixes: Browser Compatibility (Chrome/Edge)
**Symptom:** The controller works in Steam/Games but is **not detected** by Google Chrome, Microsoft Edge, or other web-based gamepad testers.

**Cause:** When using **Strategy 1** (Kernel Param), the controller identifies as a Nintendo Switch Pro Controller. Chrome attempts to initialize this specific device using raw HID commands. On Fedora and many other distros, standard users are blocked from accessing `/dev/hidraw*` devices for security reasons. Chrome fails the handshake and ignores the controller.

### The Fix: Create a Udev Rule
We need to grant your user account permission to access the Switch Pro Controller's raw interface.

1.  Create the rule file:
    ```bash
    sudo nano /etc/udev/rules.d/50-nintendo-switch.rules
    ```

2.  Paste the following content:
    ```udev
    # Access for Nintendo Switch Pro Controller (Cosmic Byte Blitz in Switch Mode)
    KERNEL=="hidraw*", ATTRS{idVendor}=="057e", ATTRS{idProduct}=="2009", MODE="0666"
    ```

3.  Save and exit (Ctrl+O, Enter, Ctrl+X).

4.  Apply the rule and reconnect:
    ```bash
    sudo udevadm control --reload-rules && sudo udevadm trigger
    ```
    Now unplug and replug your controller dongle. It should be instantly detected by gamepad-tester.com and other browser games.
