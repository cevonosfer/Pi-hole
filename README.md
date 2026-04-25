# Pi-hole
Network-wide-ad-blocker with a different touch

# Blocking Ads on My Entire Network with Pi-hole + TP-Link M7350

So I got tired of seeing ads everywhere and decided to set up Pi-hole on my Raspberry Pi. The idea is pretty simple — instead of installing an ad blocker on every single device, you just make the Raspberry Pi sit between your router and all your devices and block ads at the DNS level. Every device on the network gets ad blocking automatically.

You can find this Pi-hole tutorial everywhere on the internet but this is my version that doesnt use a standart modem. This is just me documenting what I did so I don't forget, but hopefully it helps someone else too.

---

## What's the actual setup here?

```
Internet → Your modem → Raspberry Pi running Pi-hole → all my devices
```

The Pi-hole does two jobs:
- **DNS** — intercepts all domain name lookups and blocks the ad/tracker ones
- **DHCP** — hands out IP addresses to devices (normally the router does this)

The TP-Link router basically just becomes a dumb internet pipe at this point which is fine.

---

## What you need

- A Raspberry Pi (Pi Zero would be plenty dont need fancier models)
- A Wi-Fi modem (duh)
- A SD card (16gb is fine)
- Micro USB for the power
- SD Card reader (to transfer the software)

---

## Before You Start 

Go to https://www.raspberrypi.com/software/ to download the imager software and once you have done that go through the configurations to download the image to your SD Card (make sure the ssh connection is on , rest is default) and once you have done that as well, stick the SD Card to your Pi, have it powered, wait a few minutes and ssh into it! You are good to go!  



## Step 1 — Set a static IP

This step is much easier if you have a modem that has a feature to configure DHCP, mine didnt so
i had to configure the Raspberry Pi to have a static ip (See Section 1.548)

This is important because if the Pi's IP keeps changing, all your devices won't know where to find it for DNS. You want it locked in.

I picked `192.168.0.50` since the router's DHCP range is `100–199`, so anything below that is safe to use manually.

### Section 1.548

```bash
# First check what your connection is called
nmcli con show

# Then set everything static (change the "preconfigured", mine was called "netplan-wlan0-TP-Link_BD20")
sudo nmcli con mod "preconfigured" ipv4.addresses 192.168.0.50/24
sudo nmcli con mod "preconfigured" ipv4.gateway 192.168.0.1
sudo nmcli con mod "preconfigured" ipv4.dns "192.168.0.1"
sudo nmcli con mod "preconfigured" ipv4.method manual
sudo nmcli con up "preconfigured"
```

### If you're on older Raspberry Pi OS (dhcpcd)

```bash
sudo nano /etc/dhcpcd.conf
```

Paste this at the bottom:

```
interface wlan0
static ip_address=192.168.0.50/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1
```

Then just reboot:

```bash
sudo reboot
```

### Check it worked

```bash
ip addr show wlan0
```

You should see `inet 'your ip address` with `valid_lft forever`. The "forever" part is what you want — means it won't expire.

---

## Step 2 — Install Pi-hole

This is the easy part honestly, the installer does most of the work:

```bash
curl -sSL https://install.pi-hole.net | bash
```

When it asks for settings just use these:

| Setting | What to put |
|---|---|
| Static IP | `your static ip` |
| Gateway | `your router gateway` |
| Upstream DNS | I went with Cloudflare (`1.1.1.1`) |

---

## Step 3 — Limit the router's DHCP server (Skip this step if your modem have a full DHCP customization)

As i mentioned at the beginning the M7350 doesn't let you fully customize its DHCP server which is annoying. But you can basically bypass it by shrinking its IP pool down to a single address.

Go to your routers settings and change it to only one ip address:

| Field | Change to |
|---|---|
| Start IP Address | `192.168.0.200` |
| End IP Address | `192.168.0.200` |

**After you change the addresses dont hit save yet. Wait until you configured DHCP in Pi-hole as well otherwise you will lose internet connection on all your devices and and have a bad time (dont ask me how i know)**

---

## Step 4 — Turn on DHCP in Pi-hole

Open up the Pi-hole dashboard at `http://'your static ip'/admin` and go to **Settings → DHCP**.

Set it up like this:

| Field | Value |
|---|---|
| DHCP Server | ✅ |
| Range start | `192.168.0.100` |
| Range end | `192.168.0.199` |
| Router (gateway) | `192.168.0.1` |

Also under **Additional DHCP options**, add this so Pi-hole tells every device to use itself for DNS:

```
option:dns-server, 192.168.0.50
```

Save it.

---

## Step 5 — Reconnect everything

On every device just forget the Wi-Fi network and reconnect. When they rejoin they'll get their IP from Pi-hole instead of the router and automatically start using Pi-hole for DNS.

To check if it's actually working:

- Your device IP should be somewhere in `192.168.0.100–199`
- Try going to `http://pi.hole/admin` — if that loads you're going through Pi-hole ✅
- Open the Pi-hole dashboard and watch the query log, you should see requests coming in as you browse

If queries aren't showing up, check what DNS your device is actually using:

```bash
# on mac/linux
cat /etc/resolv.conf
# should say nameserver 192.168.0.50

# on windows
ipconfig /all
# DNS Servers should be 192.168.0.50
```

If it still shows `192.168.0.1` (the router) then the DHCP option didn't take. You can manually set DNS on the device to `192.168.0.50` as a temporary fix.

---

## When the Raspberry Pi shuts down unexpectedly

Since your Pi became your new modem and is handling both DNS AND handing out IPs, if it goes down things you wont be able to have access to internet so just prepare for it : 

- Add `1.1.1.1` as a secondary DNS to Pi-hole DHCP settings so if it goes down devices will fall back to Cloudflare for DNS 
- Make the DHCP lease around 15 minutes so that when it goes down it doesnt keep the IPs 
- If you're doing it planned, just re-enable the router's DHCP range (`100–199`) before shutting Pi-hole down.
---

## Useful links

- [Pi-hole docs](https://docs.pi-hole.net/)
- [Pi-hole GitHub](https://github.com/pi-hole/pi-hole)
- [DHCP Reservation](https://www.youtube.com/watch?v=-G3ePnXAoHc)
- [RaspberryPi Website](https://www.raspberrypi.com/software/)
