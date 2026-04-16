# Forcing the Lenovo ThinkPad Copilot Key into Submission with Linux

If you recently got a new ThinkPad (like the **P16s** or similar modern models) and use Linux, you’ve probably discovered that the new dedicated **Copilot** key is practically useless out of the box. 

If you try to map it using standard desktop environment settings (like Wayland on Fedora Cosmic or GNOME), it registers as an unpredictable mess. 

This guide explains **why** the key behaves this way and provides a rock-solid, tested solution using `keyd` to turn it back into a standard, holdable Meta (Super/Windows) key.

## The Problem: It's Not a Key, It's a Macro

The hardware manufacturers didn't map the Copilot button to a single, unassigned scancode. Instead, they hardcoded a rapid-fire macro directly into the firmware. 

If you run `libinput debug-events` and press the Copilot key, you'll see something like this:

```text
-event2    KEYBOARD_KEY      +0.000s	KEY_LEFTMETA (125) pressed
 event2    KEYBOARD_KEY      +0.000s	*** (-1) pressed
 event2    KEYBOARD_KEY      +0.000s	KEY_F23 (193) pressed
 event2    KEYBOARD_KEY      +0.043s	KEY_F23 (193) released
 event2    KEYBOARD_KEY      +0.043s	*** (-1) released
 event2    KEYBOARD_KEY      +0.043s	KEY_LEFTMETA (125) released
```

### What is actually happening?
1. **The Chord:** The hardware fires `Left Meta` + `Left Shift` + `F23` simultaneously. (The `*** (-1)` is the Left Shift key being swallowed by the input handler).
2. **The Timer:** Exactly **~43 milliseconds** later, the hardware forcefully releases all three keys.

Because it's a 3-key chord that auto-releases in 43ms, standard solutions like `hwdb` or `input-remapper` fail. `hwdb` can only remap single scancodes, which leaves you leaking Meta + Shift. Most mapping tools cannot "hold" a modifier open when the firmware explicitly sends a "key up" signal.

---

## The Solution: Layer Interception with `keyd`

To fix this, we use `keyd`, a low-level key remapping daemon. Instead of trying to intercept the entire 3-key chord at once, we use a **layer trigger**. 

We tell `keyd` to catch the very first key of the macro (`Left Meta`), drop into a custom layer, and the millisecond it sees `Shift` fire, output a clean, holdable **Right Meta**. It completely ignores the trailing `F23`.

### 1. Clean Up Old Attempts
If you previously tried modifying `/etc/udev/hwdb.d/` to disable F23, delete those rules first. If F23 is blocked at the kernel level, it can cause unpredictable behavior. Don't ask how I know this. 

```bash
sudo rm /etc/udev/hwdb.d/90-custom-keyboard.hwdb
sudo systemd-hwdb update
sudo udevadm trigger
```
> **Note:** If you had active hwdb rules, you may need to fully reboot to clear them from the kernel's memory.

### 2. Install `keyd`
On Fedora, `keyd` is available via COPR. For other distros, check your package manager or build from source.

```bash
sudo dnf copr enable alternateved/keyd
sudo dnf install keyd
```

### 3. Apply the Configuration
Create and edit the default configuration file:

```bash
sudo mkdir -p /etc/keyd
sudo nano /etc/keyd/default.conf
```

Paste the following configuration exactly:

```ini
[ids]
*

[main]
# Intercept the Copilot macro's first key and treat it as a layer trigger
leftmeta = layer(meta_layer)

[meta_layer]
# When shift is also pressed (completing the Copilot macro), 
# intercept it and output a clean Right Meta hold
shift = rightmeta
```

### 4. Enable and Reload
Enable the service to run on boot and reload the daemon to apply your changes:

```bash
sudo systemctl enable keyd --now
sudo keyd reload
```

---

## Why this works so well
By turning `leftmeta` into a layer trigger, you completely bypass the 43ms timing limitation and the recursive loops of standard chords. `keyd` catches the macro before it can even finish firing, stripping out the "garbage" and handing your Wayland compositor a pristine, sustained Meta state for all your custom shortcuts.
