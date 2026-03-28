# Remote OTA Updates with Raspberry Pi Connect
This guide explains how to create a custom "Over-the-Air" (OTA) update artifact and how to prepare additional Raspberry Pi devices to receive it. This process is ideal for managing a fleet of "headless" devices (Pis without a monitor) behind firewalls or in remote locations.
## Prerequisites

* **Public Link Creation:** You must have a way to upload your .tar.zst file and create a public link. Options include a personal web server or Amazon S3.
* **Admin Access:** You must know the admin username and password for your Raspberry Pi.
* **Raspberry Pi Connect Account:** All devices must be linked to the same account. See this video for setup: [https://youtu.be/rvCaN1PSKY0](https://youtu.be/rvCaN1PSKY0)

## Phase 1: Creating the Artifact from Scratch

### Step 1: Prepare the Files
1. **The Script (aptupgradescript):** Create a file with your update logic. Use absolute paths for logging to ensure you can find the results later. Optionally, you can download the aptupgradescript included in this repository.
```bash
#!/bin/sh
export DEBIAN_FRONTEND=noninteractive
LOG="/home/admin/ota_upgrade.log"
echo "Update started: \$(date)" > "$LOG"
apt-get update >> "$LOG" 2>&1
if apt-get -y -o DPKG::Options::="--force-confnew" upgrade >> "$LOG" 2>&1; then
    if [ -r /var/run/reboot-required ]; then
        echo "Rebooting..." >> "$LOG"
        exit 2
    fi
    exit 0
else
    echo "Failed." >> "$LOG"
    exit 1
fi
```

2. **The Control File (aptupgrade.yaml):** This tells the system what the script is. Note: In version 1.3.9, keep payloads at the same indentation level as artefact. Optionally, you can download the aptupgradescript.yaml included in this repository.
```yaml
artefact:
  name: aptupgrade
  version: 1.0
  device_type: rpi
payloads:
  - name: aptupgradescript
    type: script
```

### Step 2: Build the Package
Run the otamaker tool on the YAML file. This generates the .tar.zst file and its fingerprint.
```bash
otamaker aptupgrade.yaml
```

### Step 3: Host the Artifact
1. Upload aptupgrade.tar.zst to your web server (e.g., https://yourdomain.com/).
2. Avoid Redirects: Ensure the URL you use is the final destination (e.g., use https if your site forces it).
3. Verify Link: Run curl -I https://yourdomain.com/aptupgrade.tar.zst to ensure it returns an HTTP 200 OK.
4. Get your Hash:
```bash
curl -sL https://yourdomain.com/aptupgrade.tar.zst | sha256sum
```

## Phase 2: Setting Up the Pi
```bash
sudo apt update
sudo apt install rpi-connect rpi-connect-ota
rpi-connect ota on
```

## Phase 3: Deploying to the Fleet
1. Click **Deploy** next to your Pi in the dashboard.
2. Select **Existing** and choose your Deployment artefact.
3. Click **Deploy**.
4. On the Pi you just deployed to, watch the background process in the terminal: `journalctl -t rpi-ota-connector -f`
   The dashboard will show "In Progress" and eventually "Succeeded" for each unit.

**Repeat Phase 2 and 3 for each Pi in your fleet.**
