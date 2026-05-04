# NCM to MP3/FLAC Converter · Complete Installation & Usage Guide

> Runs fully offline, features a graphical menu interface, and supports exporting to USB storage. Suitable for use in environments without network signal.


---

## Part 1: Installing Termux

1. Open the following URL in your phone's browser: `https://f-droid.org/packages/com.termux/`
2. Download and install the Termux APK
3. Once installed, open Termux

---

## Part 2: Environment Setup

In Termux, run the following commands one by one:

### 1. Update the package manager

```bash
pkg update -y && pkg upgrade -y
```

### 2. Install Python, dialog, and the Termux-API tools

```bash
pkg install python dialog termux-api -y
```

> `termux-api` is the key component that keeps the script running while the screen is off — make sure to install it!
> You also need to manually install the companion Termux:API app on your phone (the APK cannot be auto-installed via script):
> `https://f-droid.org/packages/com.termux.api/`

### 3. Install Python dependencies

```bash
pip install mutagen pycryptodome
```

### 4. Grant storage permission (only required once)

```bash
termux-setup-storage
```

> A permission prompt will appear on your phone — tap Allow.

### 5. Disable battery optimization (important)

Go to **Settings → Apps → Termux → Battery** on your phone, and select Unrestricted or No restrictions.
(On Xiaomi/Huawei devices, you may also need to disable "Background restrictions" separately.)

---

## Part 3: Installing the Script

### Method 1: Download the file (recommended)

Download the script file on your computer or phone browser, copy it to your phone's `/sdcard/`, then run the following in Termux:

```bash
cp /storage/emulated/0/ncm2mp3.py ~/ncm2mp3.py
```

### Method 2: Create the file manually

```bash
# First remove the old version (if any)
rm -f ~/ncm2mp3.py

# Create a new file
nano ~/ncm2mp3.py
```

Paste the complete script below, then press `Ctrl+X` → `Y` → Enter to save:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
NCM to MP3/FLAC Converter for Termux (Android)
v2.3.1 - Semi-graphical interface using dialog
Runs fully offline, no network required
"""

import os
import sys
import json
import struct
import base64
import binascii
import subprocess
import tempfile
import atexit
from pathlib import Path

try:
    from Crypto.Cipher import AES
except ImportError:
    subprocess.run(["clear"])
    print("Missing dependency. Please run: pip install pycryptodome")
    sys.exit(1)

try:
    from mutagen.id3 import ID3, TIT2, TPE1, TALB, APIC
    from mutagen.flac import FLAC, Picture
except ImportError:
    subprocess.run(["clear"])
    print("Missing dependency. Please run: pip install mutagen")
    sys.exit(1)


# ==== Keep running while screen is off ====

_wake_lock_held = False

def acquire_wake_lock():
    global _wake_lock_held
    try:
        result = subprocess.run(
            ["termux-wake-lock"],
            timeout=5, capture_output=True
        )
        if result.returncode == 0:
            _wake_lock_held = True
    except (FileNotFoundError, subprocess.TimeoutExpired):
        pass

def release_wake_lock():
    global _wake_lock_held
    if _wake_lock_held:
        try:
            subprocess.run(
                ["termux-wake-unlock"],
                timeout=5, capture_output=True
            )
        except (FileNotFoundError, subprocess.TimeoutExpired):
            pass
        _wake_lock_held = False

atexit.register(release_wake_lock)


# ==== dialog wrappers ====

def d_msgbox(title, msg, width=60):
    subprocess.run([
        "dialog", "--title", title,
        "--msgbox", msg, "0", str(width)
    ])

def d_infobox(title, msg, width=60):
    subprocess.run([
        "dialog", "--title", title,
        "--infobox", msg, "0", str(width)
    ])

def d_yesno(title, msg, width=60):
    r = subprocess.run([
        "dialog", "--title", title,
        "--yesno", msg, "0", str(width)
    ])
    return r.returncode == 0

def d_inputbox(title, msg, default="", width=60):
    with tempfile.NamedTemporaryFile(delete=False, suffix=".tmp") as tf:
        tmp = tf.name
    r = subprocess.run(
        ["dialog", "--title", title,
         "--inputbox", msg, "0", str(width), default],
        stderr=open(tmp, "w")
    )
    if r.returncode != 0:
        os.unlink(tmp)
        return None
    with open(tmp) as f:
        val = f.read().strip()
    os.unlink(tmp)
    return val

def d_menu(title, msg, items):
    with tempfile.NamedTemporaryFile(delete=False, suffix=".tmp") as tf:
        tmp = tf.name
    args = ["dialog", "--title", title, "--menu", msg, "0", "70", "15"]
    for tag, desc in items:
        args += [str(tag), desc]
    r = subprocess.run(args, stderr=open(tmp, "w"))
    if r.returncode != 0:
        os.unlink(tmp)
        return None
    with open(tmp) as f:
        val = f.read().strip()
    os.unlink(tmp)
    return val

def d_checklist(title, msg, items):
    with tempfile.NamedTemporaryFile(delete=False, suffix=".tmp") as tf:
        tmp = tf.name
    args = ["dialog", "--title", title,
            "--checklist", msg, "0", "72", "18"]
    for tag, desc, status in items:
        args += [str(tag), desc, status]
    r = subprocess.run(args, stderr=open(tmp, "w"))
    if r.returncode != 0:
        os.unlink(tmp)
        return None
    with open(tmp) as f:
        raw = f.read().strip()
    os.unlink(tmp)
    if not raw:
        return []
    return [x.strip().strip('"') for x in raw.split()]

def d_gauge(title, msg, items_total):
    proc = subprocess.Popen(
        ["dialog", "--title", title, "--gauge", msg, "8", "60", "0"],
        stdin=subprocess.PIPE
    )
    return proc

def d_pause_confirm(done, total, success, fail):
    msg = (
        "\n  Converted {} / {} tracks\n\n"
        "  Success: {}\n"
        "  Failed:  {}\n\n"
        "  Continue with the remaining {} tracks?\n"
    ).format(done, total, success, fail, total - done)
    return d_yesno("Pause Confirmation", msg, width=55)


# ==== NCM decryption core ====

CORE_KEY = binascii.unhexlify("687A4852416D736F356B496E62617857")
META_KEY = binascii.unhexlify("2331346C6A6B5F215C5D2630553C2728")

def decrypt_ncm(ncm_path, output_dir):
    try:
        with open(ncm_path, "rb") as f:
            f.read(8)
            f.seek(2, 1)

            key_len = struct.unpack("<I", f.read(4))[0]
            key_data = bytearray([b ^ 0x64 for b in f.read(key_len)])
            aes = AES.new(CORE_KEY, AES.MODE_ECB)
            key_data = aes.decrypt(bytes(key_data))
            pad = key_data[-1]
            key_data = key_data[:-pad][17:]

            meta_len = struct.unpack("<I", f.read(4))[0]
            music_name = Path(ncm_path).stem
            artist = "Unknown"
            album = "Unknown"
            cover_data = None
            fmt = "mp3"

            if meta_len > 0:
                meta_raw = bytearray([b ^ 0x63 for b in f.read(meta_len)])
                meta_str = base64.b64decode(meta_raw[22:])
                aes2 = AES.new(META_KEY, AES.MODE_ECB)
                meta_str = aes2.decrypt(meta_str)
                pad = meta_str[-1]
                meta_str = meta_str[:-pad][6:]
                try:
                    meta_json = json.loads(meta_str)
                    music_name = meta_json.get("musicName", music_name)
                    fmt = meta_json.get("format", "mp3").lower()
                    artists = meta_json.get("artist", [])
                    if artists:
                        artist = "/".join([a[0] for a in artists if a])
                    album = meta_json.get("album", "Unknown")
                except Exception:
                    pass
            else:
                f.seek(meta_len, 1)

            f.seek(4, 1)
            f.seek(5, 1)
            cover_len = struct.unpack("<I", f.read(4))[0]
            if cover_len > 0:
                cover_data = f.read(cover_len)

            key_len2 = len(key_data)
            S = list(range(256))
            j = 0
            for i in range(256):
                j = (j + S[i] + key_data[i % key_len2]) & 0xFF
                S[i], S[j] = S[j], S[i]

            safe_name = "".join(
                c for c in music_name if c not in r'\/:*?"<>|'
            ).strip() or Path(ncm_path).stem

            out_path = os.path.join(output_dir, "{}.{}".format(safe_name, fmt))
            counter = 1
            while os.path.exists(out_path):
                out_path = os.path.join(
                    output_dir, "{}_{}.{}".format(safe_name, counter, fmt)
                )
                counter += 1

            with open(out_path, "wb") as out_f:
                i2 = j2 = 0
                while True:
                    chunk = f.read(0x8000)
                    if not chunk:
                        break
                    result = bytearray(len(chunk))
                    for k, byte in enumerate(chunk):
                        i2 = (i2 + 1) & 0xFF
                        j2 = (j2 + S[i2]) & 0xFF
                        S[i2], S[j2] = S[j2], S[i2]
                        result[k] = byte ^ S[(S[i2] + S[j2]) & 0xFF]
                    out_f.write(result)

            try:
                if fmt == "mp3":
                    try:
                        tags = ID3(out_path)
                    except Exception:
                        tags = ID3()
                    tags.add(TIT2(encoding=3, text=music_name))
                    tags.add(TPE1(encoding=3, text=artist))
                    tags.add(TALB(encoding=3, text=album))
                    if cover_data:
                        tags.add(APIC(encoding=3, mime="image/jpeg",
                                      type=3, desc="Cover", data=cover_data))
                    tags.save(out_path)
                elif fmt == "flac":
                    audio = FLAC(out_path)
                    audio["title"] = music_name
                    audio["artist"] = artist
                    audio["album"] = album
                    if cover_data:
                        pic = Picture()
                        pic.data = cover_data
                        pic.type = 3
                        pic.mime = "image/jpeg"
                        audio.add_picture(pic)
                    audio.save()
            except Exception:
                pass

            return True, out_path, fmt

    except Exception as e:
        return False, str(e), ""


# ==== Utility functions ====

def list_ncm_files(search_dirs):
    ncm_files = []
    for d in search_dirs:
        if os.path.exists(d):
            for root, dirs, files in os.walk(d):
                for f in files:
                    if f.lower().endswith(".ncm"):
                        ncm_files.append(os.path.join(root, f))
    return sorted(ncm_files)

def get_usb_paths():
    candidates = ["/storage", "/mnt/media_rw", "/mnt/usb", "/run/media"]
    usb_paths = []
    for c in candidates:
        if os.path.exists(c):
            try:
                for item in os.listdir(c):
                    full = os.path.join(c, item)
                    if os.path.isdir(full) and item not in ["emulated", "self"]:
                        usb_paths.append(full)
            except PermissionError:
                pass
    return usb_paths

def sizeof_fmt(num):
    for unit in ["B", "KB", "MB", "GB"]:
        if abs(num) < 1024.0:
            return "{:.1f} {}".format(num, unit)
        num /= 1024.0
    return "{:.1f} TB".format(num)

def check_already_converted(output_dir):
    existing = set()
    scan_dirs = [
        output_dir,
        "/storage/emulated/0/Music",
        "/storage/emulated/0/Download",
        "/storage/emulated/0/netease/cloudmusic/Music",
    ]
    for d in scan_dirs:
        if os.path.exists(d):
            try:
                for root, dirs, files in os.walk(d):
                    for f in files:
                        if f.lower().endswith((".mp3", ".flac")):
                            existing.add(Path(f).stem.lower().strip())
            except PermissionError:
                pass
    return existing


# ==== Main UI flow ====

def screen_search():
    DEFAULT_DIRS = [
        "/storage/emulated/0/netease/cloudmusic/Music",
        "/storage/emulated/0/Android/data/com.netease.cloudmusic/files/Music",
        "/storage/emulated/0/Music",
        "/storage/emulated/0/Download",
    ]
    choice = d_menu(
        "NCM Converter - Select Search Directory",
        "Choose where to search for .ncm files:",
        [
            ("1", "Auto search (NetEase Cloud Music default paths)"),
            ("2", "Search the entire internal storage (slower)"),
            ("3", "Manually enter a path"),
        ]
    )
    if choice is None:
        return []
    if choice == "1":
        search_dirs = DEFAULT_DIRS
    elif choice == "2":
        search_dirs = ["/storage/emulated/0"]
    else:
        path = d_inputbox("Manual Path",
                          "Enter the directory containing .ncm files:",
                          "/storage/emulated/0/")
        if not path:
            return []
        search_dirs = [path]

    d_infobox("Searching", "\n  Searching for .ncm files, please wait...\n", 50)
    return list_ncm_files(search_dirs)


def screen_select(ncm_files, output_dir):
    if not ncm_files:
        d_msgbox("No Files Found",
                 "\nNo .ncm files were found.\n\n"
                 "Make sure NetEase Cloud Music has downloaded songs,\n"
                 "or try specifying a path manually.\n")
        return []

    d_infobox("Checking Converted Files",
              "\n  Checking for existing MP3/FLAC files...\n", 50)
    existing = check_already_converted(output_dir)

    items = []
    skip_count = 0
    for i, fpath in enumerate(ncm_files):
        raw_name = os.path.basename(fpath).replace(".ncm", "")
        size = sizeof_fmt(os.path.getsize(fpath))
        already = raw_name.lower().strip() in existing
        if already:
            display = "[Skip] {}  [{}]".format(raw_name[:35], size)
            status = "off"
            skip_count += 1
        else:
            display = "{}  [{}]".format(raw_name[:43], size)
            status = "on"
        items.append((str(i), display, status))

    title = "Select Files to Convert  ({} total".format(len(ncm_files))
    if skip_count > 0:
        title += ", {} already exist and will be skipped".format(skip_count)
    title += ")"

    selected_indices = d_checklist(
        title,
        "Space to toggle, arrow keys to move, Enter to confirm:",
        items
    )
    if selected_indices is None:
        return []
    return [ncm_files[int(i)] for i in selected_indices if i.isdigit()]


def screen_output():
    usb_list = get_usb_paths()
    menu_items = [("1", "Phone internal storage  /sdcard/Music/NCM_Output/")]
    for i, u in enumerate(usb_list):
        menu_items.append((str(i + 2), "USB storage  {}".format(u)))
    menu_items.append((str(len(usb_list) + 2), "Custom path"))

    choice = d_menu("Select Output Directory", "Where should converted files be saved?", menu_items)
    if choice is None:
        return None
    if choice == "1":
        return "/storage/emulated/0/Music/NCM_Output"
    elif choice.isdigit() and 2 <= int(choice) <= len(usb_list) + 1:
        return usb_list[int(choice) - 2]
    else:
        return d_inputbox("Custom Path", "Enter the output directory path:",
                          "/storage/emulated/0/Music/")


def screen_pause_interval():
    choice = d_menu(
        "Set Pause Interval",
        "After how many tracks should the script pause and ask to continue?",
        [
            ("0",  "No pause, convert all automatically"),
            ("5",  "Pause every 5  tracks"),
            ("10", "Pause every 10 tracks"),
            ("20", "Pause every 20 tracks"),
            ("50", "Pause every 50 tracks"),
        ]
    )
    if choice is None:
        return 0
    return int(choice)


def screen_convert(selected, output_dir, pause_every):
    os.makedirs(output_dir, exist_ok=True)
    total = len(selected)
    success_list = []
    fail_list = []
    aborted = False

    acquire_wake_lock()

    gauge = d_gauge("Converting (stays running with screen off)",
                    "Converting 0 / {} ...".format(total), total)

    for i, ncm_path in enumerate(selected):
        name = os.path.basename(ncm_path)
        pct = int((i / total) * 100)
        msg = "Converting {} / {}\n\n  {}".format(i + 1, total, name[:50])
        try:
            gauge.stdin.write(
                "XXX\n{}\n{}\nXXX\n".format(pct, msg).encode("utf-8")
            )
            gauge.stdin.flush()
        except Exception:
            pass

        ok, result, fmt = decrypt_ncm(ncm_path, output_dir)
        if ok:
            success_list.append((name, result, fmt))
        else:
            fail_list.append((name, result))

        done = i + 1
        if pause_every > 0 and done % pause_every == 0 and done < total:
            try:
                gauge.stdin.close()
                gauge.wait()
            except Exception:
                pass

            if not d_pause_confirm(done, total,
                                   len(success_list), len(fail_list)):
                aborted = True
                break

            gauge = d_gauge("Converting (stays running with screen off)",
                            "Converting {} / {} ...".format(done, total), total)

    try:
        gauge.stdin.write("XXX\n100\n\n  Conversion complete!\nXXX\n".encode("utf-8"))
        gauge.stdin.flush()
        gauge.stdin.close()
        gauge.wait()
    except Exception:
        pass

    release_wake_lock()

    try:
        status_str = "Aborted" if aborted else "Complete"
        subprocess.run(
            ["termux-notification",
             "--title", "NCM Conversion {}".format(status_str),
             "--content",
             "{} succeeded / {} failed, output to {}".format(
                 len(success_list), len(fail_list), output_dir),
             "--priority", "high"],
            timeout=5, capture_output=True
        )
    except (FileNotFoundError, subprocess.TimeoutExpired):
        pass

    return success_list, fail_list, aborted


def screen_result(success_list, fail_list, output_dir, aborted):
    total = len(success_list) + len(fail_list)
    status = "Aborted" if aborted else "Conversion Complete"
    msg = "\n  {}!\n\n".format(status)
    msg += "  Success: {} / {}\n".format(len(success_list), total)
    if fail_list:
        msg += "  Failed:  {}\n".format(len(fail_list))
    if aborted:
        msg += "  Remaining unconverted files were skipped\n"
    msg += "\n  Output directory:\n  {}\n".format(output_dir)
    if fail_list:
        msg += "\n  Failed files:\n"
        for name, err in fail_list[:5]:
            msg += "  - {}\n".format(name[:40])
        if len(fail_list) > 5:
            msg += "  ... {} failures total\n".format(len(fail_list))
    d_msgbox("Conversion Result", msg, 65)


def main():
    if subprocess.run(["which", "dialog"], capture_output=True).returncode != 0:
        print("dialog is not installed. Please run: pkg install dialog -y")
        sys.exit(1)

    while True:
        choice = d_menu(
            "NCM -> MP3 Converter v2.3.1",
            "\n  Welcome to the NCM Music Converter\n  Fully offline  Supports USB export\n",
            [
                ("1", "Start converting NCM files"),
                ("2", "View output directory"),
                ("3", "About this program"),
                ("4", "Exit"),
            ]
        )

        if choice is None or choice == "4":
            subprocess.run(["clear"])
            print("Goodbye!")
            break

        elif choice == "1":
            output_dir = screen_output()
            if not output_dir:
                continue

            ncm_files = screen_search()
            if not ncm_files:
                d_msgbox("No Files Found",
                         "\nNo .ncm files were found.\n\nMake sure the path is correct.\n")
                continue

            selected = screen_select(ncm_files, output_dir)
            if not selected:
                continue

            pause_every = screen_pause_interval()

            pause_tip = ("Pause every {} tracks".format(pause_every)
                         if pause_every > 0 else "Auto-convert all without pausing")
            if not d_yesno(
                "Confirm Conversion",
                "\n  About to convert {} files\n\n"
                "  Output to: {}\n\n"
                "  Pause setting: {}\n\n"
                "  You can turn off the screen during conversion; you'll get a notification when done.\n\n"
                "  Confirm to start?\n".format(len(selected), output_dir, pause_tip)
            ):
                continue

            success_list, fail_list, aborted = screen_convert(
                selected, output_dir, pause_every
            )
            screen_result(success_list, fail_list, output_dir, aborted)

        elif choice == "2":
            path = "/storage/emulated/0/Music/NCM_Output"
            if os.path.exists(path):
                files = [f for f in os.listdir(path)
                         if f.endswith((".mp3", ".flac"))]
                d_msgbox("Output Directory Contents",
                         "\n  Path: {}\n\n  {} converted files in total\n".format(
                             path, len(files)))
            else:
                d_msgbox("Output Directory",
                         "\n  No conversion has been run yet; the directory does not exist.\n")

        elif choice == "3":
            d_msgbox("About",
                     "\n  NCM -> MP3 Converter\n"
                     "  Semi-graphical interface v2.3.1\n\n"
                     "  - Runs fully offline\n"
                     "  - Supports MP3 / FLAC output\n"
                     "  - Auto-writes song tags and cover art\n"
                     "  - Supports export to USB OTG storage\n"
                     "  - Auto-skips files with the same name\n"
                     "  - Stays running with screen off / locked\n"
                     "  - Pushes a system notification when done\n"
                     "  - Supports mid-run pause/abort\n\n"
                     "  By Monica AI\n")


if __name__ == "__main__":
    main()
```

---

## Part 4: Running the Script

```bash
python ~/ncm2mp3.py
```

> From now on, this single command is all you need to run it!

---

## Part 5: Usage Instructions

### Interface controls

| Key        | Function                               |
| ---------- | -------------------------------------- |
| Arrow keys | Move between options                   |
| Spacebar   | Check / uncheck a file                 |
| Enter      | Confirm                                |
| Tab        | Switch between buttons                 |
| Esc        | Return to the previous screen          |
| Ctrl+C     | Force-interrupt conversion immediately |

### Full usage flow

```
Run the script
  ↓
Main menu → Select "Start Conversion"
  ↓
1. Choose output directory (internal storage or USB OTG)
  ↓
2. Choose search method (auto / full disk / manual path)
  ↓
3. Auto-scan for existing MP3/FLAC files
   - Files with the same name already exist → tagged [Skip], unchecked by default
   - Unconverted files                       → shown normally, checked by default
  ↓
4. Check the files you want to convert
  ↓
5. Set the pause interval (ask every N tracks / no pause)
  ↓
6. Confirm → Wake-lock is acquired automatically (you can turn the screen off!)
  ↓
7. During conversion, a confirmation dialog appears every N tracks:
   - Choose "Yes" → continue converting
   - Choose "No"  → graceful stop, all converted files are kept
  ↓
8. Done → Wake-lock is released automatically + system notification is pushed
```

### Comparison of interruption methods

| Method                      | When                 | Effect                                              |
| --------------------------- | -------------------- | --------------------------------------------------- |
| Choose "No" in pause dialog | After every N tracks | Graceful stop; current file is fully saved          |
| Ctrl+C                      | Anytime              | Immediate interrupt; current file may be incomplete |
| Swipe Termux away           | Anytime              | Force kill; wake-lock is released automatically     |

### Notes on keeping the script running while screen is off

| Mechanism           | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| termux-wake-lock    | Prevents the CPU from sleeping; acquired automatically when conversion starts |
| termux-wake-unlock  | Released automatically when conversion completes/aborts; does not affect normal power saving |
| atexit exit hook    | Even if the script crashes or is force-quit, the wake-lock is still released automatically |
| termux-notification | Pushes a system notification when conversion is finished, so you can be notified even with the screen off |

### About Termux:API

| Component            | Installation                                            | Purpose                         |
| -------------------- | ------------------------------------------------------- | ------------------------------- |
| termux-api (package) | pkg install termux-api (already included in the script) | Provides the command-line tools |
| Termux:API (App)     | Must be downloaded and installed manually from F-Droid  | Bridges Android system features |

> Both are required. Without the Termux:API app, the conversion itself still works, but keeping the script alive while the screen is off and the completion notification will be unavailable.

### How the skip logic works

`check_already_converted()` scans for `.mp3` / `.flac` files in all of the following directories:

| Scanned directory                 | Description                          |
| --------------------------------- | ------------------------------------ |
| The output directory you selected | Results from the previous conversion |
| /sdcard/Music                     | The phone's general music directory  |
| /sdcard/Download                  | The downloads directory              |
| /sdcard/netease/cloudmusic/Music  | The NetEase Cloud Music directory    |

---

## FAQ

| Issue                                      | Solution                                                     |
| ------------------------------------------ | ------------------------------------------------------------ |
| Cannot find any .ncm files                 | Choose "Manually enter path" or "Search the entire internal storage" |
| Missing-dependency error                   | Re-run `pip install mutagen pycryptodome`                    |
| USB storage does not show up               | Make sure the USB drive is properly connected and your phone supports OTG; re-run the script |
| dialog not found                           | Re-run `pkg install dialog -y`                               |
| Permission denied                          | Re-run `termux-setup-storage` and grant permission           |
| MP3 exists but isn't being skipped         | The MP3 may not be in any scanned directory; just uncheck it manually |
| Script stops running when screen turns off | Make sure the Termux:API app is installed and battery optimization is disabled |
| No notification when conversion finishes   | Make sure the Termux:API app is installed and notification permission is granted |
| Want to resume an aborted run              | Just re-run the script — already-converted files will be marked Skip automatically |
| Error: NameError: os                       | An older script has been mixed in; run `rm ~/ncm2mp3.py` and re-download |
| Error: SyntaxError: bytes                  | An older script has been mixed in; run `rm ~/ncm2mp3.py` and re-download |

---
