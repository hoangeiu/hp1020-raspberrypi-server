# HP LaserJet 1020 Print Server Installation Guide on Raspberry Pi

The HP LaserJet 1020 printer series requires a firmware upload every time it is powered on to function. This document guides you on how to set up a Print Server using CUPS and an automatic firmware loading script via `rc.local` to ensure maximum stability.

---

## 1. System & Driver Installation

Open Terminal on Raspberry Pi and run the following commands:

1. **Update system**
```
sudo apt update && sudo apt upgrade -y
```

2. **Install CUPS and HPLIP (basic support libraries)**
```
sudo apt install cups hplip -y
```

3. **Grant permissions to current user (default is pi)**
```bash
sudo usermod -a -G lpadmin $USER
```

4. **Allow remote CUPS access**
```
sudo cupsctl --remote-admin --remote-any --share-printers
```

5. **Install hplip**
```
sudo apt install hplip
```

6. **Plug the printer USB cable into the Raspberry Pi and turn on the printer power.**

Check if the Pi has recognized the printer:
```
lsusb
```

You will see text containing **Hewlett-Packard** or **HP LaserJet 1020**

7. **Add printer to CUPS system**

On a computer or phone on the same Wi-Fi network, open a web browser.

- Access address: `https://<Pi_IP_Address>:631` (Example: https://192.168.1.10:631). Note: The browser will report a security error (Not Secure), select Advanced > Proceed.

- Go to Administration tab > Click Add Printer.

- Enter Raspberry Pi user/password when asked.

- In the Local Printers list, select **HP LaserJet 1020 USB JL2FT77 HPLIP (HP LaserJet 1020)**

- In Connection screen

  - Type the following command to list all connection ports:
    ```
    lpinfo -v
    ```

  - The system will return a list. Find the line with direct usb:// or direct hp:// related to HP 1020.

  - For example, it will look like this: **direct usb://HP/LaserJet%201020?serial=JL2FT77** Or: **direct hp:/usb/HP_LaserJet_1020?serial=JL2FT77**

  - Copy the entire segment after the word direct (remove the word direct and the space).

  - Example copy: **usb://HP/LaserJet%201020?serial=JL2FT77**

  - Go back to the browser, paste that code into the empty Connection box.

  - Click Continue.


- Fill in information

  - Name: ***HP_1020_Wireless***

  - Suggestion: ***HP 1020 Printer at Raspberry Pi***

  - Location: ***Living_Room*** or ***Bookshelf***

  - Checkbox Sharing: **check** 

  - Connection: Keep value unchanged

- In Model > HP > Add Printer

- In Model > Select the first line **HP LaserJet 1020 ...** > Add Printer

8. **"Proprietary Plugin" Processing Step**
```
sudo hp-setup -i
```

Select USB ( number 0 ) > download ( d )

---

## 2. Manual Firmware Download

Because firmware needs to be loaded directly, we will download the raw .dl file to the system folder:

1. Create storage folder
```
sudo mkdir -p /lib/firmware/hp
```

2. Download firmware from backup link
```
sudo wget -O /lib/firmware/hp/sihp1020.dl https://opt.cn2qq.com/opt-file/printer/sihp1020.dl --no-check-certificate
```

3. Grant read permission to file
```
sudo chmod 755 /lib/firmware/hp/sihp1020.dl
```

---

## 3. Automatic Firmware Loading Configuration (rc.local)

Use the "Hard Reset" method to ensure the printer receives firmware immediately upon startup.

1. **Enable rc-local service**
```
sudo systemctl enable rc-local
sudo systemctl start rc-local
```

2. Edit configuration file
Open file with command: 
```
sudo nano /etc/rc.local. 
```
Delete old content and paste the following script before the exit 0 line:

```
#!/bin/bash
# Script rc.local for HP 1020 - "Hard Reset" Version

(
    # 1. Wait for system boot
    sleep 15
    echo "[START] Auto-loading Firmware HP 1020..." > /tmp/hp1020.log

    # 2. Stop CUPS to release USB port
    systemctl stop cups || true
    sleep 2

    # 3. Reset USB Module (Recreate /dev/usb/lp0)
    modprobe -r usblp || true
    sleep 2
    modprobe usblp || true
    sleep 2

    # 4. Flash Firmware to device
    FIRMWARE="/lib/firmware/hp/sihp1020.dl"
    DEVICE="/dev/usb/lp0"

    if [ -e "$DEVICE" ]; then
        cat "$FIRMWARE" > "$DEVICE"
        echo "[SUCCESS] Firmware loaded successfully at $(date)" >> /tmp/hp1020.log
    else
        echo "[FAIL] $DEVICE not found. Check USB cable." >> /tmp/hp1020.log
    fi

    # 5. Restart CUPS
    sleep 5
    systemctl start cups || true
    echo "[DONE] CUPS has restarted." >> /tmp/hp1020.log

) &

exit 0
```

3. Grant execution permission
```
sudo chmod +x /etc/rc.local
```

---

## 4. Change the printer firmware in CUPS.

1. Open your web browser and navigate to: `https://<Raspberry_Pi_IP>:631`
2. Go to the Printers tab > Click on the printer name. Select the Administration menu > Modify Printer.
3. Click Continue (proceed through the Connection and Name sections...).
4. At the Model section, ensure you select this exact line: 👉 **HP LaserJet 1020 Foomatic/foo2zjs-z1**
5. Click Modify Printer.

---

## 5. Connect on Windows

1. Go to Settings > Bluetooth & devices > Printers & scanners.
2. Select Add device and wait for it to find HP_1020_Wireless @ raspberrypi.
3. If Windows asks for driver, select manually: HP -> HP LaserJet 1020.

---

## 6. Standard Operating Procedure

To avoid firmware recognition errors, strictly follow this order:

### ON:
1. Turn on Printer switch first.
2. Plug in Raspberry Pi power.
3. Wait about 1 minute. Hearing the printer start up ("whirring sound") means it's ready.

### OFF:
1. Run command sudo halt (or use SSH App on phone).
2. Once Pi light is off, unplug Pi and Printer.
