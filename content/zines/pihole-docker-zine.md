---
title: "Pi-hole Zine: Network Ad Blocking with Docker"
---

# PI-HOLE ZINE
## Network-Wide Ad Blocking on OMV 7

*Block ads on EVERY device with one setup*

---

## WHAT YOU'RE BUILDING

```
                    BEFORE

    +--------+     +--------+     +--------+
    | PHONE  |     | LAPTOP |     | TABLET |
    +--------+     +--------+     +--------+
         |              |              |
         v              v              v
    +----------------------------------------+
    |              ROUTER                    |
    |         DNS: 8.8.8.8 (Google)          |
    +----------------------------------------+
                      |
                      v
    +----------------------------------------+
    |            INTERNET                    |
    |    (ads, trackers, malware)            |
    +----------------------------------------+


                    AFTER

    +--------+     +--------+     +--------+
    | PHONE  |     | LAPTOP |     | TABLET |
    +--------+     +--------+     +--------+
         |              |              |
         v              v              v
    +----------------------------------------+
    |              ROUTER                    |
    |        DNS: 192.168.1.200              |
    +-------------------|--------------------+
                        |
                        v
    +----------------------------------------+
    |             PI-HOLE                    |
    |    "Is this an ad? BLOCKED!"           |
    |    "Is this legit? ALLOWED!"           |
    +----------------------------------------+
                        |
                        v
    +----------------------------------------+
    |         CLEAN INTERNET                 |
    |      (no ads, no trackers)             |
    +----------------------------------------+
```

---

## THE ARCHITECTURE

```
+--------------------------------------------------+
|                RASPBERRY PI                       |
|                                                   |
|  +--------------------+  +--------------------+   |
|  |   OPENMEDIAVAULT   |  |      DOCKER       |   |
|  |    (NAS System)    |  |   (Containers)    |   |
|  +--------------------+  +--------------------+   |
|                               |                   |
|                    +----------+---------+         |
|                    |                    |         |
|              +-----------+      +-----------+     |
|              | PORTAINER |      |  PI-HOLE  |     |
|              | (Manager) |      | (Ad Block)|     |
|              +-----------+      +-----------+     |
|                                       |           |
|                              192.168.1.200        |
|                              (Own IP via         |
|                               macvlan)           |
+--------------------------------------------------+
```

---

## SETUP FLOWCHART

```
[1] INSTALL OMV-EXTRAS
        |
        v
[2] INSTALL DOCKER PLUGIN
    (System > Plugins)
        |
        v
[3] DEPLOY PORTAINER
    (Container Manager)
        |
        v
[4] CREATE DIRECTORIES
    /docker/appdata/pihole-config/
        |
        v
[5] FIX PORT 53 CONFLICT
    (Disable systemd-resolved)
        |
        v
[6] CREATE MACVLAN NETWORK
    (Give Pi-hole its own IP)
        |
        v
[7] DEPLOY PI-HOLE CONTAINER
    (Via Portainer)
        |
        v
[8] CONFIGURE ROUTER DNS
    (Point to Pi-hole)
        |
        v
[9] TEST & ENJOY AD-FREE LIFE!
```

---

## CONFIGURATION VALUES

```
+-------------------+---------------------------+
|    PARAMETER      |      YOUR VALUE           |
+-------------------+---------------------------+
| OMV Server IP     | ___.___.___.___           |
| Router/Gateway    | ___.___.___.___           |
| Pi-hole IP        | ___.___.___.___           |
| Network Interface | eth0 / end0 / enp0s3      |
| Timezone          | _________________________ |
| Pi-hole Password  | _________________________ |
+-------------------+---------------------------+

TIMEZONE EXAMPLES:
* Asia/Kolkata     (India)
* America/New_York (US East)
* Europe/London    (UK)
* Asia/Tokyo       (Japan)
```

---

## PORTAINER INSTALLATION

```bash
# Create volume for Portainer data
docker volume create portainer_data

# Deploy Portainer
docker run -d \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### ACCESS PORTAINER

```
+----------------------------------+
|  Browser: http://OMV_IP:9000     |
|                                  |
|  Create admin account:           |
|  - Username: admin               |
|  - Password: (12+ characters)    |
+----------------------------------+
```

---

## DIRECTORY STRUCTURE

```
/srv/dev-disk-by-uuid-xxxxx/
    |
    +-- docker/
         |
         +-- appdata/
              |
              +-- pihole-config/
                   |
                   +-- pihole/      <-- Config & DB
                   |
                   +-- dnsmasq.d/   <-- DNS config
```

### CREATE DIRECTORIES

```bash
DISK="/srv/dev-disk-by-uuid-YOUR-UUID"

mkdir -p ${DISK}/docker/appdata/pihole-config/pihole
mkdir -p ${DISK}/docker/appdata/pihole-config/dnsmasq.d

chown -R 999:999 ${DISK}/docker/appdata/pihole-config
chmod -R 755 ${DISK}/docker/appdata/pihole-config
```

---

## FIX PORT 53 CONFLICT

### THE PROBLEM

```
+------------------+
|  systemd-resolved | <-- Uses port 53
+------------------+
        |
        X  CONFLICT!
        |
+------------------+
|     Pi-hole      | <-- Also needs port 53
+------------------+
```

### THE FIX

```bash
# Edit configuration
sudo nano /etc/systemd/resolved.conf

# Change this line:
[Resolve]
DNSStubListener=no

# Restart service
sudo systemctl restart systemd-resolved

# Verify port is free
sudo ss -tulpn | grep :53
# Should be empty!
```

---

## MACVLAN NETWORK

### WHY MACVLAN?

```
BRIDGE NETWORK (Problems):
+------------------+
|  OMV + Pi-hole   |  Both want port 53!
|  Same IP = FAIL  |  Port conflict!
+------------------+


MACVLAN NETWORK (Solution):
+------------------+     +------------------+
|       OMV        |     |     Pi-hole      |
|  192.168.1.100   |     |  192.168.1.200   |
+------------------+     +------------------+
         |                        |
         +------------------------+
                    |
            +---------------+
            |    ROUTER     |
            |  192.168.1.1  |
            +---------------+
```

### CREATE THE NETWORK

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/32 \
  -o parent=eth0 \
  pihole_macvlan
```

```
PARAMETER GUIDE:
+------------------+---------------------------+
| --subnet         | Your network (usually     |
|                  | 192.168.1.0/24)           |
+------------------+---------------------------+
| --gateway        | Your router IP            |
+------------------+---------------------------+
| --ip-range       | IP for Pi-hole            |
|                  | /32 = single IP only      |
+------------------+---------------------------+
| -o parent        | Network interface name    |
|                  | (eth0, end0, enp0s3)      |
+------------------+---------------------------+
```

---

## PI-HOLE CONTAINER SETUP

### IN PORTAINER

```
+------------------------------------------+
|  BASIC SETTINGS                          |
+------------------------------------------+
| Name:   pihole                           |
| Image:  pihole/pihole:latest             |
+------------------------------------------+

+------------------------------------------+
|  NETWORK                                 |
+------------------------------------------+
| Network:     pihole_macvlan              |
| IPv4 Address: 192.168.1.200              |
+------------------------------------------+

+------------------------------------------+
|  VOLUMES                                 |
+------------------------------------------+
| /etc/pihole    -> .../pihole-config/pihole    |
| /etc/dnsmasq.d -> .../pihole-config/dnsmasq.d |
+------------------------------------------+

+------------------------------------------+
|  CAPABILITIES                            |
+------------------------------------------+
| [x] NET_ADMIN   <-- REQUIRED!            |
+------------------------------------------+

+------------------------------------------+
|  RESTART POLICY                          |
+------------------------------------------+
| Unless stopped                           |
+------------------------------------------+
```

---

## ENVIRONMENT VARIABLES

```
+-------------------+---------------------------+
|       NAME        |         VALUE             |
+-------------------+---------------------------+
| TZ                | Asia/Kolkata              |
|                   | (your timezone)           |
+-------------------+---------------------------+
| WEBPASSWORD       | YourNewPassword123        |
|                   | (NEW password for         |
|                   |  Pi-hole admin only!)     |
+-------------------+---------------------------+
| SERVERIP          | 192.168.1.200             |
|                   | (Pi-hole's macvlan IP)    |
+-------------------+---------------------------+
| PIHOLE_DNS_       | 1.1.1.1;1.0.0.1           |
|                   | (upstream DNS)            |
+-------------------+---------------------------+
| DNSSEC            | true                      |
+-------------------+---------------------------+
| DNSMASQ_LISTENING | all                       |
+-------------------+---------------------------+
```

### PRIVACY-FOCUSED DNS OPTIONS

```
+-------------+-------------------+------------------+
|  PROVIDER   |     PRIMARY       |    SECONDARY     |
+-------------+-------------------+------------------+
| Cloudflare  | 1.1.1.1           | 1.0.0.1          |
| Quad9       | 9.9.9.9           | 149.112.112.112  |
| OpenDNS     | 208.67.222.222    | 208.67.220.220   |
+-------------+-------------------+------------------+

* NO GOOGLE DNS (8.8.8.8) = More Privacy!
```

---

## ROUTER CONFIGURATION

```
+------------------------------------------+
|         ROUTER DHCP SETTINGS             |
+------------------------------------------+
|                                          |
|  Primary DNS:   192.168.1.200            |
|                 (Pi-hole IP)             |
|                                          |
|  Secondary DNS: [leave blank]            |
|                 or 1.1.1.1               |
|                                          |
+------------------------------------------+

After saving, devices will get new DNS
when they renew DHCP lease.

FORCE UPDATE:
* Windows: ipconfig /flushdns
* Mobile:  Toggle WiFi off/on
```

---

## TESTING

### ACCESS PI-HOLE ADMIN

```
+----------------------------------+
|  http://192.168.1.200/admin      |
|                                  |
|  Login: (WEBPASSWORD you set)    |
+----------------------------------+
```

### TEST AD BLOCKING

```bash
# This should return 0.0.0.0:
dig @192.168.1.200 doubleclick.net

# Visit this site - ads should be blocked:
https://ads-blocker.com/testing/
```

### CHECK DASHBOARD

```
+------------------------------------------+
|  PI-HOLE DASHBOARD                       |
+------------------------------------------+
|                                          |
|  Total Queries:     [################]   |
|  Queries Blocked:   [########]           |
|  Percent Blocked:   45.2%                |
|                                          |
|  Top Blocked:                            |
|  - ads.facebook.com                      |
|  - googleads.g.doubleclick.net           |
|  - analytics.google.com                  |
|                                          |
+------------------------------------------+
```

---

## TROUBLESHOOTING

### CONTAINER WON'T START

```
CHECK LOGS:
docker logs pihole

COMMON FIXES:
+------------------------------------------+
| Problem          | Solution              |
+------------------------------------------+
| Missing          | Enable NET_ADMIN in   |
| NET_ADMIN        | Portainer Capabilities|
+------------------------------------------+
| Port 53          | Set DNSStubListener=no|
| conflict         | in resolved.conf      |
+------------------------------------------+
| Permission       | chown -R 999:999      |
| denied           | on config directories |
+------------------------------------------+
```

### CAN'T REACH ADMIN PAGE

```
MACVLAN ISOLATION:
Host can't reach containers on macvlan
(this is normal!)

SOLUTIONS:
1. Access from another device on network
2. Create bridge interface (advanced)
```

### RESET PASSWORD

```bash
docker exec -it pihole pihole -a -p
# Enter new password when prompted
```

---

## QUICK REFERENCE CARD

```
+------------------------------------------+
|           PI-HOLE COMMANDS               |
+------------------------------------------+
| View logs        | docker logs pihole    |
| Restart          | docker restart pihole |
| Enter shell      | docker exec -it       |
|                  | pihole bash           |
| Update blocklists| docker exec pihole    |
|                  | pihole -g             |
| Reset password   | docker exec pihole    |
|                  | pihole -a -p          |
| Disable blocking | docker exec pihole    |
|                  | pihole disable        |
| Enable blocking  | docker exec pihole    |
|                  | pihole enable         |
+------------------------------------------+

+------------------------------------------+
|           ACCESS POINTS                  |
+------------------------------------------+
| Pi-hole Admin    | http://PIHOLE_IP/admin|
| Portainer        | http://OMV_IP:9000    |
| OMV Web UI       | http://OMV_IP         |
+------------------------------------------+
```

---

## MAINTENANCE

### UPDATE PI-HOLE

```
IN PORTAINER:
1. Go to Containers
2. Select "pihole"
3. Click "Recreate"
4. Check "Pull latest image"
5. Deploy
```

### BACKUP

```bash
# Backup config
tar -czvf pihole-backup.tar.gz \
  /path/to/pihole-config/

# Built-in backup (includes lists)
docker exec pihole pihole -a -t
```

### ADD BLOCKLISTS

```
Pi-hole Admin > Group Management > Adlists

RECOMMENDED:
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

Then: Tools > Update Gravity
```

---

## WHAT YOU'VE ACHIEVED

```
[x] Pi-hole in Docker on OMV 7
[x] Portainer for easy management
[x] Own IP via macvlan (no conflicts)
[x] Privacy-focused DNS (no Google)
[x] Network-wide ad blocking
[x] All devices protected automatically
```

---

## RELATED

**Set up your NAS first!**
See: [OMV 7 Home NAS Zine](/zines/omv7-nas-zine)

---

*Made with care by Paarth | 2025*
*Print this zine & share it!*
