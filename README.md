# 🎤 Realtek ALC256 Mic Fix on CachyOS (ASUS TUF A15) (Don't forget to star the repo ⭐!)

A step-by-step guide to fix the Realtek internal microphone not working on CachyOS (and likely other Arch-based distros) on ASUS TUF laptops (any laptop with ALC256).

---

## 💻 My Device Info

| Component | Details |
|-----------|---------|
| **Laptop** | ASUS TUF A15 |
| **Audio Codec** | Realtek ALC256 |
| **OS** | CachyOS (Arch-based) |
| **Audio Server** | PipeWire + WirePlumber |

---

## ❌ The Problem

The internal microphone worked fine on Windows but was not being picked up on CachyOS. Instead, PipeWire was defaulting to the **Ryzen HD Audio Controller** (AMD's APU audio, used for HDMI/DP output) as the audio input source, completely ignoring the Realtek mic.

Apps like Discord showed **"Ryzen HD Audio Stereo"** as the only mic — and even when selected, it produced extremely noisy audio due to the Internal Mic Boost being set to 100% by default.

---

## 🔍 Diagnosis

### Step 1 — Check detected audio hardware

```bash
aplay -l && arecord -l
```

Expected output showing Realtek is detected:
```
**** List of CAPTURE Hardware Devices ****
card 1: Generic [HD-Audio Generic], device 0: ALC256 Analog [ALC256 Analog]
```

The Realtek ALC256 is on **card 1**. Good — the hardware is detected.

---

### Step 2 — Check PipeWire sources

```bash
pactl list sources | grep -E "Name|Description|Active Port"
```

Output showed only the Ryzen controller as an input source — the Realtek mic was **not listed**, meaning PipeWire wasn't exposing it.

---

### Step 3 — List all audio cards

```bash
pactl list cards short
```

Output:
```
42    alsa_card.pci-0000_01_00.1    alsa   ← NVIDIA GPU (HDMI audio)
43    alsa_card.pci-0000_05_00.6    alsa   ← Realtek ALC256 (the one we want)
325   bluez_card.28_FA_19_FD_72_AB  module-bluez5-device.c
```

> ⚠️ **Confusing naming**: Card 43 shows as "Ryzen HD Audio Controller" in PipeWire's description, but it is actually the **Realtek ALC256**. This is because AMD's HDA controller acts as a bus/host for the Realtek codec on the motherboard. The name is misleading but the hardware is correct.

---

### Step 4 — Inspect card 43 profiles

```bash
pactl list cards | grep -A 80 "alsa_card.pci-0000_05_00.6"
```

Key findings:
- `alsa.mixer_name = "Realtek ALC256"` ✅ confirmed Realtek
- Active Profile was already `output:analog-stereo+input:analog-stereo` (Duplex) ✅
- `analog-input-internal-mic` port was available ✅

So the hardware and profile were fine — the issue was just that PipeWire wasn't defaulting to it, and the mic boost was set too high.

---

## ✅ The Fix

### Fix 1 — Set Realtek as default audio source

```bash
pactl set-default-source alsa_input.pci-0000_05_00.6.analog-stereo
```

Verify it's set:
```bash
pactl get-default-source
# Should return: alsa_input.pci-0000_05_00.6.analog-stereo
```

---

### Fix 2 — Lower the Internal Mic Boost (THE MAIN FIX)

The mic boost was set to **100% by default**, causing extremely heavy noise/static.

Check the current boost level:
```bash
amixer -c 1 get 'Internal Mic Boost'
# Front Left: 3 [100%] [30.00dB]  ← way too high!
```

Set it to 0:
```bash
amixer -c 1 set 'Internal Mic Boost' 0
```

Verify it's applied:
```bash
amixer -c 1 get 'Internal Mic Boost'
# Front Left: 0 [0%] [0.00dB]  ← 
```

> 💡 If the mic sounds too quiet after setting to 0, try `1` (25%) instead.

#### Make it permanent across reboots

This is the tricky part. Here is what happens at every boot and why naive fixes don't work:

1. **Kernel** loads ALC256 driver → boost set to 100% by default
2. **alsa-restore** (udev rule) → restores `asound.state` → still 100%
3. Any system-level fix (udev rule, systemd system service) → sets to 0% ✅
4. **WirePlumber user session** starts → restores its own saved mixer state → resets back to 100% ❌

The only fix that works is a **user systemd service** that runs *after* WirePlumber finishes, so it gets the last word.

```bash
mkdir -p ~/.config/systemd/user/
nano ~/.config/systemd/user/mic-boost-fix.service
```

Paste:
```ini
[Unit]
Description=Fix Internal Mic Boost after WirePlumber
After=wireplumber.service
Wants=wireplumber.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 5
ExecStart=/usr/bin/amixer -c 1 set 'Internal Mic Boost' 0
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

Enable it:
```bash
systemctl --user enable --now mic-boost-fix.service
```

> 💡 The `sleep 5` gives WirePlumber enough time to fully finish restoring its state before we override it. Last one to write wins!

---

### Fix 3 — Make default source permanent across reboots

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/
nano ~/.config/wireplumber/wireplumber.conf.d/51-default-source.conf
```

Paste the following:
```
wireplumber.settings = {
  default.audio.source = "alsa_input.pci-0000_05_00.6.analog-stereo"
}
```

Restart WirePlumber:
```bash
systemctl --user restart wireplumber
```

---

## ✔️ Verification

Confirm the default source is correct:
```bash
pactl info | grep "Default Source"
# Default Source: alsa_input.pci-0000_05_00.6.analog-stereo
```

Confirm it's the Realtek codec:
```bash
pactl list cards | grep -A 2 "alsa.mixer_name"
# alsa.mixer_name = "Realtek ALC256"   ← same PCI address as default source
```

Record a test clip:
```bash
arecord -D plughw:1,0 -f cd -d 5 test.wav && aplay test.wav
```

---

## ❓ FAQ

**Q: Discord/apps still show "Ryzen HD Audio" — is that wrong?**  
A: No! That's just a cosmetic label from PipeWire. The underlying hardware at `pci-0000_05_00.6` is confirmed to be the Realtek ALC256. The Ryzen HD Audio Controller has no physical mic — it only handles HDMI/DP audio output.

**Q: Why was the mic boost at 100%?**  
A: Linux ALSA sets default gain values that don't always match what the hardware needs. ASUS laptops with ALC256 are known to have this issue.

**Q: Will this survive a reboot?**  
A: Yes — but only with the user systemd service in Fix 2. `alsactl store`, udev rules, and system-level services all get overwritten by WirePlumber's user session at login. The user service runs last and wins.

**Q: Does this apply to other distros?**  
A: Yes — any Arch-based distro using PipeWire + WirePlumber should work the same way. May also apply to other ASUS laptops with Realtek ALC256/ALC255.

---

## 🧰 Tools Used

- `aplay` / `arecord` — ALSA playback/record utils
- `pactl` — PipeWire/PulseAudio control
- `alsamixer` — ALSA mixer TUI
- `alsactl store` — save ALSA state
- `wireplumber` — PipeWire session manager

---

## 📄 License

MIT — feel free to use, share, and improve!
