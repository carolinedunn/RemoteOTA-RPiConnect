# Remote OTA Updates with Raspberry Pi Connect
This guide walks you through the end-to-end process of creating an Over-the-Air (OTA) update artifact for Raspberry Pi Connect and applying that same artifact to additional devices in your fleet. This process can be used with other scripts to install / deploy software packages.

## Prerequisites

* **Public Link Creation:** You must have a way to upload your .tar.zst file and create a public link. Options include a personal web server or Amazon S3.
* **Admin Access:** You must know the admin username and password for your Raspberry Pi.
* **Raspberry Pi Connect Account:** All devices must be linked to the same account. See this video for setup: [https://youtu.be/rvCaN1PSKY0](https://youtu.be/rvCaN1PSKY0)

## Phase 1: Creating the Artifact from Scratch
This process turns your shell script into a compressed package that the Raspberry Pi Connect service can verify and deploy. Optional: You can skip steps 1 and 2 and download the pre-made file [aptupgrade.tar.zst](https://github.com/carolinedunn/RemoteOTA-RPiConnect/blob/main/aptupgrade.tar.zst)

### Step 1: Prepare the Files
On your primary computer (or the "admin" Pi), create a folder for this update. You need two files:
1. **The Script (aptupgradescript):** Create a file with your update logic. Use absolute paths for logging to ensure you can find the results later. Optionally, you can download the [aptupgradescript](https://github.com/carolinedunn/RemoteOTA-RPiConnect/blob/main/aptupgradescript) included in this repository.
```bash
#!/bin/sh
export DEBIAN_FRONTEND=noninteractive
apt update
if apt -y -o DPKG::Options::="--force-confnew" upgrade > output.txt; then
    if [ -r /var/run/reboot-required ]; then
        echo Rebooting to finish the upgrade
        exit 2 # EXIT_REBOOT
    fi
else
    echo Upgrade failed:
    echo
    cat output.txt
    exit 1 # EXIT_FAILURE
fi
echo Upgrade complete
exit 0 # EXIT_SUCCESS
```

2. **The Control File (aptupgrade.yaml):** This tells the system what the script is. Note: In version 1.3.9, keep payloads at the same indentation level as artefact. Optionally, you can download the [aptupgradescript.yaml](https://github.com/carolinedunn/RemoteOTA-RPiConnect/blob/main/aptupgrade.yaml) included in this repository.
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
3. Verify Link: Run the following command and it should return an HTTP 200 OK.
   ```bash
   curl -I https://yourdomain.com/aptupgrade.tar.zst
   ```
4. Get your Hash: Run the following command (using your own link) and copy the long string of characters it returns:
    ```bash
    curl -sL https://yourdomain.com/aptupgrade.tar.zst | sha256sum
    ```

### Step 4: Register the Artifact
1. Log in to the [Raspberry Pi Connect Dashboard](https://connect.raspberrypi.com/devices)
2. Go to **Remote Update -> New**
3. Enter the Name (your choice), HTTPS URL and the hash from the previous step in the SHA-256 Checksum field.
4. Click **Create artefact**

## Phase 2: Setting Up the Pi
1. Install the latest rpi-connect package and the new rpi-connect-ota package:

For a Raspberry Pi with desktop:
```bash
sudo apt update
sudo apt install rpi-connect rpi-connect-ota
rpi-connect ota on
```
For a Raspberry Pi without a desktop (aka Lite):
```bash
sudo apt update
sudo apt install rpi-connect-lite rpi-connect-ota
rpi-connect ota on
```
2. Enter your admin password when prompted. This is the password you set up when you flashed your microSD card in Raspberry Pi Imager.

## Phase 3: Deploying to the Fleet
1. Go to your [Raspberry Pi Connect Dashboard](https://connect.raspberrypi.com/devices)
2. Click **Deploy** next to your Pi in the dashboard.
3. Select **Existing** and choose your Deployment artefact.
4. Click **Deploy**.
5. On the Pi you just deployed to, watch the background process in the terminal: `journalctl -t rpi-ota-connector -f`
   The dashboard will show "In Progress" and eventually "Succeeded" for each unit.

**Repeat Phase 2 and 3 for each Pi in your fleet.**

This tutorial is based on this post from [Raspberry Pi](https://www.raspberrypi.com/news/new-remote-updates-on-raspberry-pi-connect/).
