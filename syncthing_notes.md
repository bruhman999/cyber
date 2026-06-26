# Syncthing Reference Notes

---

## What Is Syncthing?

Syncthing is an open-source, peer-to-peer file synchronization tool. It keeps folders in sync across two or more devices in real time. Unlike cloud services (Google Drive, OneDrive), your files are never stored on a third-party server — they go directly between your devices.

Relay servers are used only as a fallback if a direct connection between devices cannot be established. Even then, traffic is encrypted and the relay cannot read your files.

---

## How Safe Is It?

- All transfers are TLS encrypted in transit
- Each device generates a unique cryptographic certificate tied to its Device ID
- Device IDs function like public keys — knowing someone's ID gives them no access
- Every device that connects must be explicitly approved by you in the GUI
- Every folder shared must also be explicitly approved

To access your files, someone would need your Device ID AND you would need to accept their connection request AND share a folder with them. There is no way to access files passively.

---

## Installation

### Fedora
```bash
sudo dnf install syncthing
```
Enable autostart as your user (not root):
```bash
systemctl --user enable syncthing
systemctl --user start syncthing
```

### Debian / Ubuntu
```bash
sudo apt install syncthing
```
Enable autostart:
```bash
systemctl --user enable syncthing
systemctl --user start syncthing
```

### Windows
1. Download the installer from https://syncthing.net/downloads/
2. Run the installer
3. To autostart: open the GUI at http://127.0.0.1:8384/ → Actions → Settings → General → check "Start on Login"

---

## Accessing the GUI

On any device Syncthing is running on, open a browser and go to:
```
http://127.0.0.1:8384/
```

---

## How to Pair Two Devices

### Step 1 — Get both Device IDs
- On each device, open the GUI
- Go to Actions → Show ID
- Copy the Device ID (looks like: WE7CJPF-F7QMP6B-JIMCHF2-...)

### Step 2 — Add the remote device
- On Device A, click Add Remote Device
- Paste Device B's ID
- Give it a name and hit Save
- Device B will show a popup asking to accept — click Add Device

### Step 3 — Share a folder
- On Device A, go to Folders → Add Folder (or edit an existing folder)
- Set the folder path
- Under the Sharing tab, check Device B
- Hit Save
- Device B will prompt you to accept the folder — accept it and choose where to store it locally

Syncthing will begin syncing automatically once both devices are online at the same time.

---

## Key Behaviors to Know

- Both devices must be online simultaneously to sync
- Changes made while a scan is in progress are fine — Syncthing will pick them up
- If you edit the same file on both devices before they sync, a conflict copy is created with a filename like: document~DEVICEID-conflict-20260625.txt — just delete whichever copy you don't want
- Syncthing watches for changes continuously — on flash drives this means constant read/write activity which can cause wear over time
- If a folder path becomes unavailable (e.g. flash drive unplugged), Syncthing will show a folder error — you can pause the folder in the GUI to avoid this

---

## Folder Types

| Type | Behavior |
|------|----------|
| Send & Receive | Changes sync both ways (default) |
| Send Only | This device pushes changes, never receives |
| Receive Only | This device receives changes, never pushes |

---

## Ports Used

| Port | Protocol | Purpose |
|------|----------|---------|
| 8384 | TCP | Local GUI / API |
| 22000 | TCP + QUIC | Device sync traffic |
| 21027 | UDP | Local discovery (broadcast) |

---

## Tips

- Use Tailscale alongside Syncthing if you want sync to work away from home without opening ports
- Avoid syncing large binary files that change constantly (game saves, databases, recordings)
- Syncthing is best for documents, configs, notes, scripts, and similar files
- For files that need to be accessed from multiple machines without offline use, a shared TrueNAS SMB folder may be simpler
