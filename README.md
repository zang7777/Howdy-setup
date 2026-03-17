# 👁️ Howdy — IR Facial Recognition on Ubuntu 24.04
> Windows Hello for Linux — Complete Setup Guide

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?style=flat&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Howdy](https://img.shields.io/badge/Howdy-2.6.1-4A90D9?style=flat)](https://github.com/boltgolt/howdy)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![Solved with Claude](https://img.shields.io/badge/Solved%20with-Claude%20(Anthropic)-D4A843?style=flat)](https://claude.ai)

---

## 📸 Demo

## 📸 Demo

### 📷 Capture
<p align="center">
  <img src="https://github.com/zang7777/Howdy-setup/blob/main/demo/capture.jpeg" width="700"/>
</p>

---

### ⚙️ 🚀Processing / Output
<p align="center">
  <img src="https://github.com/zang7777/Howdy-setup/blob/main/demo/output.gif" width="700"/>
</p>

---

### ⚙️ setup
<p align="center">
  <img src="https://github.com/zang7777/Howdy-setup/blob/main/demo/working.gif" width="700"/>
</p>
---

## 🧠 What Is This?

**Howdy** brings Windows Hello-style facial recognition to Linux. It uses your laptop's built-in **IR (Infrared) camera** — the small dark oval on the webcam strip — to authenticate you at login, sudo prompts, and more. No password needed, just look at the camera.

> ⚠️ **Disclaimer:** This is not created by me — just a setup guide by a clueless kid learning how to set up Howdy on Ubuntu. Use at your own risk.
---

## 📷 Camera Options & Trade-offs

### ✅ **Regular RGB Webcam** (`/dev/video1`)
- ✔️ Works with Howdy
- ✔️ Easier to set up
- ❌ Can be fooled by a photo of your face
- ❌ Affected by lighting changes (too dark = fails)
- ❌ Less accurate overall

### ✅ **IR Camera** (`/dev/video3`) — **RECOMMENDED**
- ✔️ Can't be fooled by photos
- ✔️ Works in complete darkness (IR emitter lights your face invisibly)
- ✔️ More accurate and faster recognition
- ✔️ **What Windows Hello uses**

> 💡 **Recommendation:** Use the IR camera for best results. It's more secure and reliable.

---

## ⚙️ System Requirements

| Item | Details |
|------|---------|
| OS | Ubuntu 24.04 LTS |
| IR Camera | Any IR webcam (e.g. `Integrated_Webcam_FHD`) |
| Python | 3.12+ |
| Howdy | 2.6.1 |

---

## 🚀 Setup Guide

### Step 1 — Install Python Dependencies

Ubuntu 24.04 uses PEP 668 which blocks system-wide pip installs by default.  
Always use `--break-system-packages` flag:

```bash
sudo pip install --break-system-packages \
  dlib \
  face_recognition \
  opencv-python \
  ffmpeg-python \
  numpy --ignore-installed
```

> ⚠️ If numpy throws a conflict error, add `--ignore-installed` — it's already installed via apt and works fine.

---

### Step 2 — Add Howdy PPA & Install

```bash
sudo add-apt-repository ppa:boltgolt/howdy
sudo apt update
sudo apt install howdy
```

> 💡 If install gets interrupted, fix with:
> ```bash
> sudo dpkg -i --force-all /var/cache/apt/archives/howdy_2.6.1_all.deb
> sudo dpkg --configure -a
> ```

---

### Step 3 — Create Howdy Alias

Howdy 2.6.1 on Ubuntu installs as a PAM module only — no binary in PATH.  
Create an alias to use it like a normal command:

```bash
echo 'alias howdy="sudo python3 /usr/lib/security/howdy/cli.py"' >> ~/.bashrc
source ~/.bashrc
```

Now `howdy add`, `howdy config`, `howdy list` all work.

---

### Step 4 — Fix Python 3 Compatibility Bug

Howdy's `pam.py` uses the Python 2 module name `ConfigParser` which doesn't exist in Python 3:

```bash
sudo sed -i 's/import ConfigParser/import configparser as ConfigParser/g' \
  /usr/lib/security/howdy/pam.py
```

---

### Step 5 — Find Your IR Camera

```bash
v4l2-ctl --list-devices
```

Test each device visually — IR camera will look grey/monochrome:

```bash
ffplay /dev/video0    # regular webcam — shows color
ffplay /dev/video2    # IR camera — shows grey/infrared
```

Your IR camera is usually `/dev/video2`.

---

### Step 6 — Configure Howdy

```bash
howdy config
```

Change these settings:

```ini
[video]
device_path = /dev/video2      # your IR camera device
certainty = 4.5                # lower = stricter (1-10)
timeout = 10                   # seconds before giving up
recording_plugin = opencv      # more stable than ffmpeg for IR
force_mjpeg = true             # fixes IR frame decoding issues
dark_threshold = 50            # handles IR lighting

[snapshots]
capture_failed = false         # disable to avoid permission errors
capture_successful = false
```

---

### Step 7 — Add Your Face

```bash
howdy add
```

- Look directly at the **IR camera** (dark oval — not the regular webcam)
- Enter a label like `zang` when prompted
- Stay still and well-lit while frames are captured

---

### Step 8 — Set Up PAM Integration

Check if Howdy is already in PAM:

```bash
cat /etc/pam.d/common-auth | grep howdy
```

If empty (nothing returned), add it manually:

```bash
sudo nano /etc/pam.d/common-auth
```

Add this as the **very first line**:

```
auth sufficient /usr/lib/security/pam_python.so /usr/lib/security/howdy/pam.py
```

> 💡 Find the correct `pam_python.so` path with:
> ```bash
> find / -name "pam_python.so" 2>/dev/null
> ```

---

### Step 9 — Fix GStreamer & DeprecationWarning Noise

Suppress OpenCV GStreamer warnings:

```bash
echo 'export OPENCV_LOG_LEVEL=ERROR' >> ~/.bashrc
source ~/.bashrc
```

Fix Python deprecation warning in compare.py:

```bash
sudo sed -i 's/datetime.datetime.utcnow()/datetime.datetime.now(datetime.timezone.utc)/g' \
  /usr/lib/security/howdy/compare.py
```

Fix snapshot permission error:

```bash
sudo chmod 777 /usr/lib/security/howdy/snapshots
```

---

### Step 10 — Test It

```bash
sudo -k
sudo ls
```

Look at the IR camera. You should see:

```
Identified face as zang
```

---

## 🐛 Problems & Solutions

| Problem | Cause | Fix |
|---------|-------|-----|
| `Nala already running instance` | Ctrl+Z suspended a previous install | Kill PIDs + clear lock files |
| `howdy: command not found` | No binary installed — PAM module only | Create alias pointing to `cli.py` |
| `half-installed (iHR)` | Install was interrupted | `sudo dpkg -i --force-all howdy_*.deb` |
| `externally-managed-environment` | Ubuntu 24.04 PEP 668 | Add `--break-system-packages` to pip |
| `ConfigParser` module not found | Python 2 vs 3 naming | `sed` patch on `pam.py` |
| PAM not triggering face scan | Howdy never added to PAM | Manually add to `common-auth` |
| `pam_python.so` not found | Wrong path in PAM config | `find / -name pam_python.so` |
| `AttributeError: 'int' has no isdigit` | Bug in ffmpeg recorder | Switch to `recording_plugin = opencv` |
| GStreamer warnings in terminal | OpenCV verbose logging | `export OPENCV_LOG_LEVEL=ERROR` |
| `permission denied` on snapshots | Wrong folder permissions | `sudo chmod 777 howdy/snapshots` |
| Face not recognized | Low certainty or bad lighting | Set `certainty = 4.5`, re-add model |

---

## 📋 Useful Commands

```bash
howdy add        # add a new face model
howdy list       # list saved models
howdy remove     # remove a model
howdy enable     # enable Howdy
howdy disable    # disable Howdy temporarily
howdy config     # edit configuration
```

---

## 🤖 Configured with Claude

This entire setup — every error, every fix, every config tweak — was debugged live with **Claude by Anthropic**.

```
  ╔══════════════════════════════════════════╗
  ║                                          ║
  ║        ✦  Claude by Anthropic  ✦        ║
  ║                                          ║
  ║     AI that guided this setup from       ║
  ║     first error to final working         ║
  ║     facial authentication — live.        ║
  ║                                          ║
  ║          https://claude.ai               ║
  ║                                          ║
  ╚══════════════════════════════════════════╝
```

---

<p align="center">
  Made with ❤️ on Ubuntu 24.04 &nbsp;•&nbsp; Debugged with <strong>Claude (Anthropic)</strong>
</p>

---

