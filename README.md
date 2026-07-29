# Onkyo HDC-1.0 Linux audio

**GitHub repository blurb (About description):**

> Make an Onkyo HDC-1 work on Linux/Ubuntu: ALSA playback, line-in recording via WM8776 (`model=se200pci`), and notes on optical S/PDIF.

**Topics / tags:** `onkyo` `hdc-1` `alsa` `ice1724` `envy24` `se-90pci` `wm8776` `ubuntu` `mpd`

---

Guide for running an **Onkyo HDC-1** (Windows HTPC reinstalled with Linux) as a dedicated audio machine.

| Status | Feature |
|---|---|
| Works | Analog playback (`hw:0,0`) |
| Works | Analog line-in recording (RCA IN → WM8776) |
| Works | Driver auto-load after disabling OSS4 blacklists |
| Help wanted | Optical TOSLINK → external DAC lock under Linux |

**Tested on:** Ubuntu 22.04 LTS, in-tree `snd-ice1724`.

---

## Hardware overview

| Rear panel | Function | Connected to |
|---|---|---|
| AUDIO ANALOG **OUT** L/R (RCA) | Playback | Onkyo PCI audio card |
| **DIGITAL OPTICAL** (TOSLINK) | S/PDIF out | Same PCI card (soldered) |
| AUDIO ANALOG **IN** L/R (RCA) | Line-in / recording | Same PCI card via internal header **P203** |

### Important: this is not a stock SE-90PCI

PCI ID reports:

- Vendor/device: `1412:1724` (VIA VT1720/24 Envy24PT/HT)
- Subsystem: `160b:0011` → identified as **Onkyo SE-90PCI**

Retail SE-90PCI is **playback-only** (no ADC). The HDC-1 board is a **custom variant**:

- Same subsystem ID / EEPROM identity as SE-90PCI
- Extra **WM8776S** codec (ADC + DAC)
- Line-in wired to header **P203** (4-wire from rear RCA IN)

Linux’s `model=se90pci` path sets `num_total_adcs = 0` and never initializes WM8776 — capture opens but records silence. Forcing `model=se200pci` reuses the SE-200PCI WM8776 init and enables recording.

Controller chip (example): **VIA VT1720** (Envy24MT).

---

## 1. Make the driver load at boot

If `snd-ice1724` does not bind automatically, check for leftover **OSS4** blacklists:

```bash
ls /etc/modprobe.d/*oss4*
```

If present, disable them:

```bash
sudo mv /etc/modprobe.d/oss4-base_noALSA.conf /etc/modprobe.d/oss4-base_noALSA.conf.disabled
sudo mv /etc/modprobe.d/oss4-base.conf /etc/modprobe.d/oss4-base.conf.disabled 2>/dev/null
sudo mv /etc/modprobe.d/oss4-base_noOSS3.conf /etc/modprobe.d/oss4-base_noOSS3.conf.disabled 2>/dev/null
```

Use the SE-200PCI model so WM8776 (line-in) is initialized:

```bash
echo 'options snd-ice1724 model=se200pci' | sudo tee /etc/modprobe.d/snd-ice1724.conf
sudo reboot
```

After reboot:

```bash
lspci -nnk -d 1412:1724
# Kernel driver in use: snd_ice1724

cat /proc/asound/cards
# Expect: ONKYO SE200PCI (name comes from the forced model)

aplay -l
arecord -l
```

Reload without reboot (stop anything using the card first):

```bash
sudo systemctl stop mpd
pulseaudio -k 2>/dev/null
systemctl --user stop pulseaudio.socket pulseaudio.service 2>/dev/null
sudo modprobe -r snd_ice1724
sudo modprobe snd-ice1724
```

---

## 2. Analog playback (recommended daily path)

```text
hw:0,0
```

Example **MPD** output (`/etc/mpd.conf`):

```conf
audio_output {
    type            "alsa"
    name            "HDC-1 analog"
    device          "hw:0,0"
    format          "44100:32:2"
    mixer_type      "software"
    buffer_time     "100000"
}
```

Under `model=se200pci`, raise Front playback:

```bash
amixer -c 0 sset Front 100%
```

Verify while playing:

```bash
cat /proc/asound/card0/pcm0p/sub0/status
# expect: state RUNNING, appl_ptr advancing
```

---

## 3. Analog recording (line-in / tape)

### Mixer

```bash
amixer -c 0 sset 'Capture Select' 'LINE-IN'
amixer -c 0 sset 'Capture' 88%              # tune to taste; 100% is often too hot (~+24 dB)
amixer -c 0 sset 'AGC Capture Mode' 'Off'
sudo alsactl store 0                        # persist across reboot
```

`Capture Select` options: `LINE-IN`, `CD-IN`, `MIC-IN`, `ALL-MIX`.

### Record / play back

```bash
# source playing into rear ANALOG IN
arecord -D hw:0,0 -f S32_LE -c 2 -r 44100 -d 10 /tmp/in.wav
aplay -D hw:0,0 /tmp/in.wav
```

Live trim:

```bash
alsamixer -c 0
# F4 = Capture view
```

### Why `se90pci` fails for input

| Model | WM8776 init | Capture controls | Line-in |
|---|---|---|---|
| `se90pci` (default for this SSID) | No | No | Silent WAV |
| `se200pci` (forced) | Yes | Capture / Capture Select | Works on HDC-1 |

This borrows SE-200PCI codec setup on hardware that still identifies as SE-90PCI.

---

## Help wanted: optical (S/PDIF) output

Analog path is solid. **Optical out under Linux is not reliably working** on at least one HDC-1 + simple TOSLINK PC-speaker DAC setup. If you get optical working, please open an issue/PR with your recipe.

### What already works on the host

- Digital PCM device opens and streams: `hw:0,1` or `iec958:CARD=SE200PCI,DEV=0`
- `/proc/asound/card0/pcm1p/sub0/status` shows `RUNNING` with advancing `appl_ptr`
- TOSLINK **red light** reaches the DAC
- Light often **stays on** after stop (idle carrier — common; not proof of audio lock)
- Same machine’s optical worked under **Windows**; jack is on the SE-90PCI / HDC-1 card

### What fails

- No audible output / no lock on a no-UI optical PC-speaker DAC
- Tried: `44100:32:2`, `44100:16:2`, `48000:16:2`, raw `hw:0,1`, `iec958:...`, and analog + `IEC958=PCM Out` mirror

### Things to try / report

```bash
# optical-only example
device "hw:0,1"
format "44100:16:2"

amixer -c 0 sget 'IEC958 Output'   # expect on
amixer -c 0 sget 'IEC958'          # expect PCM Out
```

- Dual stream: play `hw:0,0` and `hw:0,1` together
- Another DAC/AVR, another optical cable
- `iecset` / AES channel-status bits
- Kernel version and whether `model=se200pci` vs `se90pci` changes SPDIF behavior

### Temporary workaround

Use **analog RCA OUT** for listening.

---

## Optional: headless boot

```bash
sudo systemctl set-default multi-user.target
```

When a monitor is needed: `sudo systemctl start gdm`  
Restore GUI default: `sudo systemctl set-default graphical.target`

---

## Quick reference

| Goal | Setting |
|---|---|
| Module options | `options snd-ice1724 model=se200pci` |
| Analog play | `hw:0,0`, Front 100% |
| Analog record | Capture Select `LINE-IN`, tune Capture % |
| Persist mixer | `sudo alsactl store 0` |
| Optical play | `hw:0,1` / `iec958:...` — **help wanted** |

---

## Related upstream

- Kernel: `sound/pci/ice1712/` (`snd-ice1724`, `se.c`)
- Models: `se90pci`, `se200pci`
- Docs: [ALSA configuration — snd-ice1724](https://docs.kernel.org/sound/alsa-configuration.html)

A proper upstream quirk (“HDC-1: SE-90PCI SSID + WM8776 line-in”) would be cleaner than forcing `se200pci` forever.

---

## Disclaimer

Hardware varies. Forcing `model=se200pci` initializes SE-200PCI-oriented controls (extra surround mixers may appear). On HDC-1 it has enabled line-in **and** kept stereo analog playback working; verify on your unit.

Revert if needed:

```bash
echo 'options snd-ice1724 model=se90pci' | sudo tee /etc/modprobe.d/snd-ice1724.conf
sudo modprobe -r snd_ice1724 && sudo modprobe snd-ice1724
```
