# Cosmic Byte Blitz Controller Integration (Linux)

> **⚠️ DISCLAIMER: Tested Configuration**  
> This guide is specifically tested on the **Cosmic Byte Blitz** controller. While the strategies described here are generally applicable to other generic "Shanwan" chipset controllers that fail to enumerate correctly, your mileage may vary. Proceed with caution when applying system-level changes.

## Video Showcase of Cosmic Byte Blitz with Vibration on Fedora 43


https://github.com/user-attachments/assets/106812f4-dcfc-470d-be35-bec9d5bd4ec3



## Introduction
This repository documents the primary strategy for resolving the common issue where the Cosmic Byte Blitz controller connects as a generic device without vibration support on Linux. The goal is to correct the USB enumeration process to ensure the device is detected in a mode that supports vibration.

---

## Strategy : The "Kernel Param" Fix
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

**Cause:** Modern browsers (like Chrome) attempt to initialize gamepads using raw HID commands. On Fedora and many other distros, standard users are blocked from accessing `/dev/hidraw*` devices for security reasons.

*   **For Users using this method:** Your controller identifies as a **Nintendo Switch Pro Controller** (ID `057e:2009`). Chrome *requires* access to handshake with this specific device. Use the rule below as written.
*   **For Other Controllers:** If your controller is detected as something else but still doesn't work in the browser, you can adapt the rule below using your own device ID.

### The Fix: Create a Udev Rule
We need to grant your user account permission to access the controller's raw interface.

1.  **Find your Controller ID (Optional for Strategy 1):**
    If you are NOT using this method, run `lsusb` in a terminal and look for your controller (e.g., `ID 054c:0ce6`). The first part is `idVendor`, the second is `idProduct`.

2.  Create the rule file:
    ```bash
    sudo nano /etc/udev/rules.d/50-controller-perms.rules
    ```

3.  Paste the following content:
    ```udev
    # Access for Nintendo Switch Pro Controller (Strategy 1 / Cosmic Byte Blitz)
    KERNEL=="hidraw*", ATTRS{idVendor}=="057e", ATTRS{idProduct}=="2009", MODE="0666"

    # Access for Generic/Other Controllers (Uncomment and replace IDs if needed)
    # KERNEL=="hidraw*", ATTRS{idVendor}=="YOUR_VENDOR_ID", ATTRS{idProduct}=="YOUR_PRODUCT_ID", MODE="0666"
    ```

4.  Save and exit (Ctrl+O, Enter, Ctrl+X).

5.  Apply the rule and reconnect:
    ```bash
    sudo udevadm control --reload-rules && sudo udevadm trigger
    ```
    Now unplug and replug your controller dongle. It should be instantly detected by gamepad-tester.com and other browser games.
