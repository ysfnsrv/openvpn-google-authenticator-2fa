Perfect ✅ — here’s a **complete, polished `README.md`** written in English, formatted and ready for direct copy-paste into your GitHub repository.
Everything is included: structure, credits to **Angristan**, explanation of why we enable services, local network + port forwarding notes, and screenshot placeholders.

---

````markdown
# OpenVPN + Google Authenticator (TOTP) 2FA Setup

This repository documents a simple and secure way to deploy **OpenVPN** with **Google Authenticator 2FA (TOTP)** on **Ubuntu**.

It is based on the excellent [angristan/openvpn-install](https://github.com/angristan/openvpn-install) project, extended with a minimal TOTP verification layer and user management helper scripts.

---

## 🎯 Overview

This setup allows you to:

- Install OpenVPN quickly using the Angristan installer  
- Add a **second authentication factor (TOTP)** for each user  
- Manage users via a small script that generates QR codes for Google Authenticator  
- Run the VPN in a **local network behind NAT**, then forward the port (UDP 1194) for remote access

> ⚠️ All credit for the base installer goes to **Angristan**.  
> This project only adds 2FA and configuration examples on top of it.

---

## 🧩 Architecture

1. **OpenVPN** installed via `openvpn-install.sh`  
2. Runs as a systemd service: `openvpn@server`  
3. Adds `/etc/openvpn/server/google-authenticator.sh` for TOTP verification  
4. Stores secrets in `/etc/openvpn/server/google-authenticator.keys`  
5. Users are created with `/usr/local/bin/add-vpn-user-otp.sh`  
6. Clients authenticate using:
   - **Username:** created via the helper script  
   - **Password:** current 6-digit TOTP code from Google Authenticator

---

## ⚙️ Prerequisites

- Ubuntu 22.04 or 24.04 LTS
- Root or sudo access
- Public IP or domain (or a router with NAT port forwarding)
- OpenVPN client software (Windows, macOS, Linux, iOS, Android)

Example network:

| Role | Address | Note |
|------|----------|------|
| VPN Server | 10.0.0.10 | Local IP |
| Public IP | 203.0.113.10 | Router external address |
| Port Forward | UDP 1194 → 10.0.0.10:1194 | Required |

---

## 🧠 Why We Enable Certain Services

- **openvpn@server.service** – runs OpenVPN as a managed systemd instance  
- **IP forwarding** – allows VPN clients to route traffic through the server  
- **oathtool & qrencode** – required for generating and validating TOTP codes  
- **nogroup permissions** – ensures the OpenVPN daemon can securely read OTP secrets

---

## 🚀 Installation Steps

### 1. Install dependencies

```bash
sudo -i
apt update
apt install -y git curl openvpn oathtool qrencode libpam-google-authenticator
````

---

### 2. Clone and run the Angristan OpenVPN installer

```bash
cd /root
git clone https://github.com/angristan/openvpn-install.git
cd openvpn-install
chmod +x openvpn-install.sh
./openvpn-install.sh
```

When prompted, choose:

* **Port:** `1194`
* **Protocol:** `UDP`
* **DNS:** your choice (Google or AdGuard recommended)
* **IPv6:** `n`
* **Encryption:** default settings

At the end, a client profile (e.g. `/root/new.ovpn`) will be generated.

---

### 3. Enable the proper OpenVPN service

```bash
systemctl disable --now openvpn.service || true
systemctl enable --now openvpn@server
systemctl status openvpn@server -l
```

Check that it’s running and listening on UDP 1194:

```bash
ss -tulpn | grep 1194
```

---

### 4. Prepare storage for TOTP keys

```bash
touch /etc/openvpn/server/google-authenticator.keys
chown root:nogroup /etc/openvpn/server/google-authenticator.keys
chmod 640 /etc/openvpn/server/google-authenticator.keys
```

---

### 5. Create the verification script

File: `/etc/openvpn/server/google-authenticator.sh`

```bash
#!/bin/bash
# OpenVPN auth-user-pass-verify script using TOTP (Google Authenticator compatible)

if [ -z "$username" ] || [ -z "$password" ]; then
  exit 1
fi

KEYS_FILE="/etc/openvpn/server/google-authenticator.keys"
secret_key=$(grep "^$username:" "$KEYS_FILE" | cut -d: -f2)
[ -z "$secret_key" ] && exit 1

expected_code=$(oathtool --totp -b "$secret_key")

if [ "$expected_code" = "$password" ]; then
  exit 0
else
  exit 1
fi
```

Permissions:

```bash
chown root:nogroup /etc/openvpn/server/google-authenticator.sh
chmod 750 /etc/openvpn/server/google-authenticator.sh
```

---

### 6. Link the script to OpenVPN configuration

Edit `/etc/openvpn/server.conf` and add at the end:

```conf
script-security 3
auth-user-pass-verify "/etc/openvpn/server/google-authenticator.sh" via-env
username-as-common-name
```

Restart the service:

```bash
systemctl restart openvpn@server
```

---

### 7. Add users with helper script

Create: `/usr/local/bin/add-vpn-user-otp.sh`

```bash
#!/bin/bash
KEYS_FILE="/etc/openvpn/server/google-authenticator.keys"

read -p "Enter username: " USERNAME
[ -z "$USERNAME" ] && echo "Empty username" && exit 1

if grep -q "^$USERNAME:" "$KEYS_FILE"; then
  echo "User already exists."
  exit 1
fi

SECRET=$(head -c 20 /dev/urandom | base32 | tr -d '=' | tr -d '[:space:]')
[ -z "$SECRET" ] && echo "Failed to generate secret" && exit 1

echo "$USERNAME:$SECRET" >> "$KEYS_FILE"
chmod 640 "$KEYS_FILE"

echo "User: $USERNAME"
echo "Secret: $SECRET"
echo
echo "Scan this QR with Google Authenticator:"
qrencode -t ANSIUTF8 "otpauth://totp/VPN:$USERNAME?secret=$SECRET&issuer=VPN"
echo
echo "Add it as 'VPN:$USERNAME' in your TOTP app."
```

Set permissions:

```bash
chmod 700 /usr/local/bin/add-vpn-user-otp.sh
```

Run it:

```bash
/usr/local/bin/add-vpn-user-otp.sh
```

---

### 8. Client configuration

* Use the `.ovpn` file generated by Angristan (`/root/new.ovpn`)
* Ensure it contains:

```conf
remote YOUR_PUBLIC_IP 1194
proto udp
auth-user-pass
auth SHA256
```

When connecting, the user enters:

* **Username:** created via `add-vpn-user-otp.sh`
* **Password:** 6-digit TOTP code from Google Authenticator

---

## 🌐 Port Forwarding Example

If your server is behind NAT:

| Direction | Protocol | External Port | Internal IP | Internal Port |
| --------- | -------- | ------------- | ----------- | ------------- |
| WAN → LAN | UDP      | 1194          | 10.0.0.10   | 1194          |

---

## 📸 Recommended Screenshots

Place screenshots in `images/` and reference them in this README.

1. `images/angristan-install.png` – OpenVPN installation process
2. `images/systemd-status.png` – `systemctl status openvpn@server` output
3. `images/router-forwarding.png` – router NAT/port forwarding rule
4. `images/otp-qr.png` – output of `add-vpn-user-otp.sh` showing QR code
5. `images/authenticator.png` – Google Authenticator app entry
6. `images/openvpn-login.png` – OpenVPN client login window

Add to README if you want:

```markdown
See the `images/` directory for step-by-step screenshots.
```

---

## 🧰 Maintenance Tips

* Check logs:
  `/var/log/openvpn/openvpn.log`
  `/var/log/openvpn/status.log`

* Restart service:
  `systemctl restart openvpn@server`

* Remove a user:
  `sed -i '/^USERNAME:/d' /etc/openvpn/server/google-authenticator.keys`

* Keep system time synced for TOTP:
  `apt install chrony`

---

## 📜 License & Credits

* **Base installer:** [angristan/openvpn-install](https://github.com/angristan/openvpn-install)
  (All credit to the original author)
* **This repository:** TOTP scripts and documentation licensed under **MIT**

---

## 🧭 Summary

| Step | Description                             |
| ---- | --------------------------------------- |
| 1    | Install Angristan’s OpenVPN script      |
| 2    | Configure and enable `openvpn@server`   |
| 3    | Add Google Authenticator script         |
| 4    | Create users with `add-vpn-user-otp.sh` |
| 5    | Connect with username + TOTP code       |
| 6    | Port-forward UDP 1194 if needed         |

---

**Author:** *Your Name*
**Created:** November 2025
**Purpose:** Internal VPN with simple, secure 2FA access for local or small-team environments.

```

---

## ✅ What to do now

1. Create a new repo on GitHub (e.g. `openvpn-google-authenticator-2fa`).
2. Inside it, add:
   - `README.md` (the file above)  
   - `google-authenticator.sh`  
   - `add-vpn-user-otp.sh`
3. Optionally, create an `images/` folder and upload the screenshots I listed.
4. Commit and push.

Then your GitHub page will look professional, clear, and properly credit **Angristan** while documenting your 2FA extension.

---

Would you like me to also generate the **two .sh files** (`google-authenticator.sh` and `add-vpn-user-otp.sh`) in clean standalone form ready for upload into the repo next?
```
