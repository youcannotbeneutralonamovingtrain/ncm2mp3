# NCM 转 MP3/FLAC 转换器 · 完整安装使用指南

> 完全离线运行，支持图形菜单界面，支持导出到USB存储，适合无信号环境使用。


---

## 第一部分：安装 Termux

1. 用手机浏览器打开：`https://f-droid.org/packages/com.termux/`
2. 下载并安装 Termux APK
3. 安装完成后打开 Termux

---

## 第二部分：环境配置

在 Termux 中逐条执行以下命令：

### 1. 更新包管理器

```bash
pkg update -y && pkg upgrade -y
```

### 2. 安装 Python、dialog 和 Termux-API 工具

```bash
pkg install python dialog termux-api -y
```

> `termux-api` 是息屏保持运行的关键组件，务必安装！
> 同时还需要在手机上手动安装 Termux:API 配套 App（APK 无法通过脚本自动安装）：
> `https://f-droid.org/packages/com.termux.api/`

### 3. 安装 Python 依赖库

```bash
pip install mutagen pycryptodome
```

### 4. 授予存储权限（只需执行一次）

```bash
termux-setup-storage
```

> 执行后手机会弹出权限请求，点击允许

### 5. 关闭电池优化（重要）

进入手机 **设置 → 应用 → Termux → 电池** → 选择不限制或无限制
（小米/华为需额外关闭"后台限制"）

---

## 第三部分：安装脚本

### 方法一：下载文件（推荐）

在电脑或手机浏览器下载脚本文件，复制到手机 `/sdcard/`，然后在 Termux 执行：

```bash
cp /storage/emulated/0/ncm2mp3.py ~/ncm2mp3.py
```

### 方法二：手动创建

```bash
# 先删除旧版本（如有）
rm -f ~/ncm2mp3.py

# 新建文件
nano ~/ncm2mp3.py
```

粘贴以下完整脚本，然后按 `Ctrl+X` → `Y` → 回车保存：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
NCM to MP3/FLAC Converter for Termux (Android)
v2.3.1 - 半图形化界面版，使用 dialog
完全离线运行，无需网络
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
    print("缺少依赖，请先运行：pip install pycryptodome")
    sys.exit(1)

try:
    from mutagen.id3 import ID3, TIT2, TPE1, TALB, APIC
    from mutagen.flac import FLAC, Picture
except ImportError:
    subprocess.run(["clear"])
    print("缺少依赖，请先运行：pip install mutagen")
    sys.exit(1)


# ==== 息屏保持运行 ====

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


# ==== dialog 封装 ====

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
        "\n  已转换 {} / {} 首\n\n"
        "  成功：{} 个\n"
        "  失败：{} 个\n\n"
        "  是否继续转换剩余 {} 首？\n"
    ).format(done, total, success, fail, total - done)
    return d_yesno("暂停确认", msg, width=55)


# ==== NCM 解密核心 ====

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


# ==== 工具函数 ====

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


# ==== 主界面流程 ====

def screen_search():
    DEFAULT_DIRS = [
        "/storage/emulated/0/netease/cloudmusic/Music",
        "/storage/emulated/0/Android/data/com.netease.cloudmusic/files/Music",
        "/storage/emulated/0/Music",
        "/storage/emulated/0/Download",
    ]
    choice = d_menu(
        "NCM Converter - 选择搜索目录",
        "请选择 .ncm 文件的搜索范围：",
        [
            ("1", "自动搜索（网易云默认路径）"),
            ("2", "搜索整个内部存储（较慢）"),
            ("3", "手动输入路径"),
        ]
    )
    if choice is None:
        return []
    if choice == "1":
        search_dirs = DEFAULT_DIRS
    elif choice == "2":
        search_dirs = ["/storage/emulated/0"]
    else:
        path = d_inputbox("手动输入路径",
                          "请输入 .ncm 文件所在目录：",
                          "/storage/emulated/0/")
        if not path:
            return []
        search_dirs = [path]

    d_infobox("搜索中", "\n  正在搜索 .ncm 文件，请稍候...\n", 50)
    return list_ncm_files(search_dirs)


def screen_select(ncm_files, output_dir):
    if not ncm_files:
        d_msgbox("未找到文件",
                 "\n未找到任何 .ncm 文件。\n\n"
                 "请确认网易云音乐已下载歌曲，\n或尝试手动指定路径。\n")
        return []

    d_infobox("检查已转换文件",
              "\n  正在检查已存在的 MP3/FLAC 文件...\n", 50)
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

    title = "选择要转换的文件  (共 {} 个".format(len(ncm_files))
    if skip_count > 0:
        title += "，{} 个已存在将跳过".format(skip_count)
    title += ")"

    selected_indices = d_checklist(
        title,
        "空格键选择/取消，方向键移动，回车确认：",
        items
    )
    if selected_indices is None:
        return []
    return [ncm_files[int(i)] for i in selected_indices if i.isdigit()]


def screen_output():
    usb_list = get_usb_paths()
    menu_items = [("1", "手机内部存储  /sdcard/Music/NCM_Output/")]
    for i, u in enumerate(usb_list):
        menu_items.append((str(i + 2), "USB存储  {}".format(u)))
    menu_items.append((str(len(usb_list) + 2), "自定义路径"))

    choice = d_menu("选择输出目录", "转换后的文件保存到哪里？", menu_items)
    if choice is None:
        return None
    if choice == "1":
        return "/storage/emulated/0/Music/NCM_Output"
    elif choice.isdigit() and 2 <= int(choice) <= len(usb_list) + 1:
        return usb_list[int(choice) - 2]
    else:
        return d_inputbox("自定义路径", "请输入输出目录路径：",
                          "/storage/emulated/0/Music/")


def screen_pause_interval():
    choice = d_menu(
        "设置暂停间隔",
        "每转换多少首后暂停询问是否继续？",
        [
            ("0",  "不暂停，全部自动转换完"),
            ("5",  "每 5  首暂停一次"),
            ("10", "每 10 首暂停一次"),
            ("20", "每 20 首暂停一次"),
            ("50", "每 50 首暂停一次"),
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

    gauge = d_gauge("转换中（息屏保持运行）",
                    "正在转换 0 / {} ...".format(total), total)

    for i, ncm_path in enumerate(selected):
        name = os.path.basename(ncm_path)
        pct = int((i / total) * 100)
        msg = "正在转换 {} / {}\n\n  {}".format(i + 1, total, name[:50])
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

            gauge = d_gauge("转换中（息屏保持运行）",
                            "正在转换 {} / {} ...".format(done, total), total)

    try:
        gauge.stdin.write("XXX\n100\n\n  转换完成！\nXXX\n".encode("utf-8"))
        gauge.stdin.flush()
        gauge.stdin.close()
        gauge.wait()
    except Exception:
        pass

    release_wake_lock()

    try:
        status_str = "已中止" if aborted else "已完成"
        subprocess.run(
            ["termux-notification",
             "--title", "NCM 转换{}".format(status_str),
             "--content",
             "成功 {} 个 / 失败 {} 个，输出到 {}".format(
                 len(success_list), len(fail_list), output_dir),
             "--priority", "high"],
            timeout=5, capture_output=True
        )
    except (FileNotFoundError, subprocess.TimeoutExpired):
        pass

    return success_list, fail_list, aborted


def screen_result(success_list, fail_list, output_dir, aborted):
    total = len(success_list) + len(fail_list)
    status = "已中止" if aborted else "转换完成"
    msg = "\n  {}！\n\n".format(status)
    msg += "  成功：{} / {} 个\n".format(len(success_list), total)
    if fail_list:
        msg += "  失败：{} 个\n".format(len(fail_list))
    if aborted:
        msg += "  剩余未转换文件已跳过\n"
    msg += "\n  输出目录：\n  {}\n".format(output_dir)
    if fail_list:
        msg += "\n  失败文件：\n"
        for name, err in fail_list[:5]:
            msg += "  - {}\n".format(name[:40])
        if len(fail_list) > 5:
            msg += "  ... 共 {} 个失败\n".format(len(fail_list))
    d_msgbox("转换结果", msg, 65)


def main():
    if subprocess.run(["which", "dialog"], capture_output=True).returncode != 0:
        print("未安装 dialog，请先运行：pkg install dialog -y")
        sys.exit(1)

    while True:
        choice = d_menu(
            "NCM -> MP3 Converter v2.3.1",
            "\n  欢迎使用 NCM 音乐转换器\n  完全离线 · 支持USB导出\n",
            [
                ("1", "开始转换 NCM 文件"),
                ("2", "查看输出目录"),
                ("3", "关于本程序"),
                ("4", "退出"),
            ]
        )

        if choice is None or choice == "4":
            subprocess.run(["clear"])
            print("再见！")
            break

        elif choice == "1":
            output_dir = screen_output()
            if not output_dir:
                continue

            ncm_files = screen_search()
            if not ncm_files:
                d_msgbox("未找到文件",
                         "\n未找到任何 .ncm 文件。\n\n请确认路径是否正确。\n")
                continue

            selected = screen_select(ncm_files, output_dir)
            if not selected:
                continue

            pause_every = screen_pause_interval()

            pause_tip = ("每 {} 首暂停一次".format(pause_every)
                         if pause_every > 0 else "全程自动转换，不暂停")
            if not d_yesno(
                "确认转换",
                "\n  即将转换 {} 个文件\n\n"
                "  输出到：{}\n\n"
                "  暂停设置：{}\n\n"
                "  转换期间可以息屏，完成后会推送通知。\n\n"
                "  确认开始？\n".format(len(selected), output_dir, pause_tip)
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
                d_msgbox("输出目录内容",
                         "\n  路径：{}\n\n  共 {} 个已转换文件\n".format(
                             path, len(files)))
            else:
                d_msgbox("输出目录",
                         "\n  尚未进行过转换，目录不存在。\n")

        elif choice == "3":
            d_msgbox("关于",
                     "\n  NCM -> MP3 转换器\n"
                     "  半图形界面版 v2.3.1\n\n"
                     "  - 完全离线运行\n"
                     "  - 支持 MP3 / FLAC 输出\n"
                     "  - 自动写入歌曲标签和封面\n"
                     "  - 支持导出到 USB OTG 存储\n"
                     "  - 自动跳过已存在同名文件\n"
                     "  - 息屏/锁屏时保持运行\n"
                     "  - 转换完成推送系统通知\n"
                     "  - 支持中途暂停/终止转换\n\n"
                     "  By Monica AI\n")


if __name__ == "__main__":
    main()
```

---

## 第四部分：运行脚本

```bash
python ~/ncm2mp3.py
```

> 以后每次使用只需执行这一条命令即可！

---

## 第五部分：操作说明

### 界面操作按键

| 按键 | 功能 |
|------|------|
| 方向键 | 移动选项 |
| 空格键 | 勾选 / 取消勾选文件 |
| 回车键 | 确认 |
| Tab 键 | 在按钮间切换 |
| Esc 键 | 返回上一级 |
| Ctrl+C | 强制立即中断转换 |

### 完整使用流程

```
运行脚本
  ↓
主菜单 → 选择「开始转换」
  ↓
1. 选择输出目录（内部存储 或 USB OTG）
  ↓
2. 选择搜索方式（自动 / 全盘 / 手动路径）
  ↓
3. 自动扫描已存在的 MP3/FLAC 文件
   - 已存在同名文件 → 标注 [Skip]，默认不勾选
   - 未转换文件    → 正常显示，默认勾选
  ↓
4. 勾选要转换的文件
  ↓
5. 设置暂停间隔（每 N 首询问一次 / 不暂停）
  ↓
6. 确认 → 自动申请唤醒锁（可以息屏了！）
  ↓
7. 转换中，每 N 首弹出确认框：
   - 选「是」→ 继续转换
   - 选「否」→ 优雅停止，已转换文件全部保留
  ↓
8. 完成 → 自动释放唤醒锁 + 推送系统通知
```

### 中断转换方式对比

| 方式 | 时机 | 效果 |
|------|------|------|
| 暂停确认框选「否」| 每 N 首后 | 优雅停止，当前文件已完整保存 |
| Ctrl+C | 随时 | 立即中断，当前文件可能不完整 |
| 划掉 Termux | 随时 | 强制终止，唤醒锁自动释放 |

### 息屏保持运行说明

| 机制 | 说明 |
|------|------|
| termux-wake-lock | 阻止 CPU 休眠，转换开始时自动申请 |
| termux-wake-unlock | 转换完成/中止后自动释放，不影响正常省电 |
| atexit 退出钩子 | 即使脚本崩溃或被强制退出，也会自动释放唤醒锁 |
| termux-notification | 转换完成后推送系统通知，息屏也能收到提醒 |

### Termux:API 说明

| 组件 | 安装方式 | 作用 |
|------|---------|------|
| termux-api（包） | pkg install termux-api（脚本已包含） | 提供命令行工具 |
| Termux:API（App） | 需手动去 F-Droid 下载安装 | 桥接 Android 系统功能 |

> 两者缺一不可。不装 Termux:API App 不影响转换功能，但息屏保持运行和完成通知将不可用。

### 跳过逻辑说明

check_already_converted() 会同时扫描以下目录中的 .mp3 / .flac 文件：

| 扫描目录 | 说明 |
|---------|------|
| 你选择的输出目录 | 上次转换的结果 |
| /sdcard/Music | 手机通用音乐目录 |
| /sdcard/Download | 下载目录 |
| /sdcard/netease/cloudmusic/Music | 网易云目录 |

---

## 常见问题

| 问题 | 解决方法 |
|------|---------|
| 找不到 .ncm 文件 | 选择「手动输入路径」或「搜索整个内部存储」 |
| 提示缺少依赖 | 重新运行 pip install mutagen pycryptodome |
| USB存储未显示 | 确认U盘已插好且手机支持OTG，重新运行脚本 |
| dialog 未找到 | 重新运行 pkg install dialog -y |
| 权限被拒绝 | 重新运行 termux-setup-storage 并允许权限 |
| 明明有 MP3 却没被跳过 | 该 MP3 可能不在扫描目录内，手动取消勾选即可 |
| 息屏后脚本停止运行 | 确认已安装 Termux:API App 并关闭电池优化 |
| 转换完成没有收到通知 | 确认已安装 Termux:API App 并授予通知权限 |
| 中止后想继续未完成的 | 重新运行脚本，已转换文件会自动标记跳过 |
| 报错 NameError: os | 旧版脚本混入，执行 rm ~/ncm2mp3.py 后重新下载 |
| 报错 SyntaxError: bytes | 旧版脚本混入，执行 rm ~/ncm2mp3.py 后重新下载 |

