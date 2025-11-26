---
title: "Home NAS Zine: OMV 7 Setup Guide"
---

# HOME NAS ZINE
## OpenMediaVault 7 on Raspberry Pi

*A visual guide to building your own network storage*

---

## WHAT YOU'RE BUILDING

```
    +------------------+
    |   YOUR PHONE     |
    |   YOUR LAPTOP    |----+
    |   YOUR TABLET    |    |
    +------------------+    |
                            v
                    +--------------+
                    |   ROUTER     |
                    |  192.168.1.1 |
                    +--------------+
                            |
                            v
                    +--------------+
                    | RASPBERRY PI |
                    |    + OMV 7   |
                    |   + 1TB SSD  |
                    +--------------+
                            |
                            v
                    +------------------+
                    |   YOUR FILES     |
                    | Photos, Videos,  |
                    | Documents, etc.  |
                    +------------------+
```

---

## HARDWARE SHOPPING LIST

```
+------------------------+------------------+
|        ITEM            |   RECOMMENDED    |
+------------------------+------------------+
| Raspberry Pi           | Pi 4/5, 4GB+ RAM |
| MicroSD Card           | 32GB+, Class 10  |
| External Storage       | 1TB+ SSD or HDD  |
| Power Supply           | Official 5V 3A   |
| Ethernet Cable         | Cat5e or Cat6    |
| Case (optional)        | With cooling     |
+------------------------+------------------+
```

---

## SETUP FLOWCHART

```
[1] FLASH OMV IMAGE
        |
        v
[2] BOOT RASPBERRY PI
        |
        v
[3] FIND PI's IP ADDRESS
    (check router admin)
        |
        v
[4] ACCESS WEB UI
    http://YOUR_PI_IP
        |
        v
[5] LOGIN
    user: admin
    pass: openmediavault
        |
        v
[6] CHANGE PASSWORD!
    (very important)
        |
        v
[7] CONNECT STORAGE
    Storage > Disks
        |
        v
[8] CREATE SHARED FOLDER
        |
        v
[9] ENABLE SMB/CIFS
        |
        v
[10] CONNECT FROM DEVICES!
```

---

## THE DREADED ERROR

```
+--------------------------------------------------+
|  ERROR: Unknown device "/dev/sda": No such device |
+--------------------------------------------------+
```

### WHY IT HAPPENS

```
  KERNEL
    |
    v
+-------------+     +-------------+
| BLACKLIST   | --> | USB_STORAGE |
| CONFIG FILE |     | BLOCKED!    |
+-------------+     +-------------+
                          |
                          X (no drive)
```

### THE FIX

```bash
# 1. Check for blacklist
grep -r "blacklist usb" /etc/modprobe.d/

# 2. Edit the file
sudo nano /etc/modprobe.d/raspi-blacklist.conf

# 3. Comment out these lines:
# blacklist usb_storage
# blacklist uas

# 4. Load the modules
sudo modprobe usb_storage
sudo modprobe uas

# 5. Reboot
sudo reboot
```

### VERIFY IT WORKED

```bash
$ lsblk

NAME   SIZE TYPE MOUNTPOINT
sda    1TB  disk
└─sda1 1TB  part /srv/dev-disk-by-uuid-xxx
```

---

## NETWORK SHARE SETUP

### SAMBA CONFIGURATION

```
+------------------------------------------+
|  /etc/samba/smb.conf                     |
+------------------------------------------+
|                                          |
|  [SharedFiles]                           |
|     path = /srv/.../shared               |
|     browseable = yes                     |
|     writeable = yes                      |
|     valid users = your_username          |
|                                          |
+------------------------------------------+
```

### CREATE SAMBA USER

```bash
sudo smbpasswd -a your_username
# Enter password when prompted
# (can be different from system password)
```

---

## CONNECTING FROM WINDOWS

```
+------------------------+
|    WINDOWS PC          |
|                        |
|  Press: Win + R        |
|  Type:  \\192.168.1.X  |
|  Press: Enter          |
|                        |
+------------------------+
         |
         v
+------------------------+
|    LOGIN PROMPT        |
|                        |
|  User: your_username   |
|  Pass: samba_password  |
|                        |
+------------------------+
         |
         v
+------------------------+
|    YOUR FILES!         |
|                        |
|  [SharedFiles]         |
|    - Photos/           |
|    - Documents/        |
|    - Videos/           |
+------------------------+
```

### HOSTNAME NOT WORKING?

```
"Windows cannot find \\RASPBERRYPI"

SOLUTION: Use IP address instead!
\\192.168.1.X  <-- works every time
```

---

## MOBILE ACCESS

### ANDROID

```
+---------------------------+
|  RECOMMENDED APPS         |
+---------------------------+
| * Solid Explorer          |
| * CX File Explorer        |
| * Total Commander         |
+---------------------------+

SETUP:
1. Install app
2. Add Network Storage
3. Choose SMB/CIFS
4. Enter:
   - Host: 192.168.1.X
   - User: your_username
   - Pass: samba_password
5. Connect!
```

### iOS

```
+---------------------------+
|  BUILT-IN FILES APP       |
+---------------------------+

1. Open Files app
2. Tap "..." menu
3. "Connect to Server"
4. Enter: smb://192.168.1.X
5. Login with credentials
6. Done!
```

---

## REMOTE ACCESS WITH TAILSCALE

### THE PROBLEM

```
  HOME NETWORK              INTERNET              COFFEE SHOP
+-------------+          +----------+          +-------------+
|  Your NAS   |   XXX    | FIREWALL |   XXX   |  Your Phone |
| 192.168.1.X |<---------|   NAT    |-------->|             |
+-------------+          +----------+          +-------------+
              ^                                       |
              |         CAN'T CONNECT!                |
              +---------------------------------------+
```

### THE SOLUTION

```
  HOME NETWORK              TAILSCALE              COFFEE SHOP
+-------------+          +-----------+          +-------------+
|  Your NAS   |          | ENCRYPTED |          |  Your Phone |
| 100.64.x.x  |<-------->|   TUNNEL  |<-------->| 100.64.y.y  |
+-------------+          +-----------+          +-------------+
              ^                                       |
              |          CONNECTED!                   |
              +---------------------------------------+
```

### SETUP

```bash
# On Raspberry Pi:
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Get your Tailscale IP:
tailscale ip -4
# Returns: 100.x.x.x

# On your phone/laptop:
# Install Tailscale app
# Login with same account
# Connect using 100.x.x.x
```

---

## QUICK REFERENCE CARD

```
+------------------------------------------+
|          ESSENTIAL COMMANDS              |
+------------------------------------------+
| Find Pi IP      | hostname -I            |
| Check Samba     | systemctl status smbd  |
| Restart Samba   | systemctl restart smbd |
| Check disks     | lsblk                  |
| Add Samba user  | smbpasswd -a USERNAME  |
| Tailscale IP    | tailscale ip -4        |
+------------------------------------------+

+------------------------------------------+
|          IMPORTANT FILES                 |
+------------------------------------------+
| Samba config    | /etc/samba/smb.conf    |
| Module blacklist| /etc/modprobe.d/       |
| OMV config      | /etc/openmediavault/   |
+------------------------------------------+

+------------------------------------------+
|          DEFAULT CREDENTIALS             |
+------------------------------------------+
| OMV Web UI      | admin / openmediavault |
|                 | (CHANGE IMMEDIATELY!)  |
+------------------------------------------+
```

---

## TROUBLESHOOTING FLOWCHART

```
                    PROBLEM?
                       |
          +------------+------------+
          |                         |
    Can't see drive          Can't connect
          |                         |
          v                         v
    Run: lsblk              Samba running?
          |                 systemctl status smbd
     +----+----+                    |
     |         |               +----+----+
  Shows    Doesn't         Running   Not running
  drive    show                |         |
     |         |               v         v
     v         v           Check IP   Start it:
  Check    Check          hostname -I  systemctl
  blacklist power                      start smbd
```

---

## SECURITY CHECKLIST

```
[ ] Changed default OMV password
[ ] Created strong Samba password (12+ chars)
[ ] System updated (sudo apt update && upgrade)
[ ] Using Tailscale for remote (NOT port forward)
[ ] Installed fail2ban for SSH protection
[ ] Regular backups configured
```

---

## NEXT STEPS

**Add network-wide ad blocking!**
See: [Pi-hole on OMV 7 Zine](/zines/pihole-docker-zine)

---

*Made with care by Paarth | 2025*
*Print this zine & share it!*
