# nixos-initrd-systemd_ssh-clevis-tailscale
Basic configuration to run sshd in initrd-systemd in NixOS, optionally including tailscale connectivity and clevis unlock of LUKS volumes

#  One-time imperative setup steps

Run these on rocinante itself before the initrd modules are fully functional. Order matters: SSH host key first, Clevis enrollment second, Tailscale state file last (optional).

---

## 1. Initrd SSH host key

Generates a stable host key so `known_hosts` doesn't churn on every boot. Do this once — the key is copied into the initrd as a secret.

```bash
sudo ssh-keygen -t ed25519 -N "" -f /etc/ssh/initrd_ssh_host_ed25519_key
sudo chmod 400 /etc/ssh/initrd_ssh_host_ed25519_key

# Record the fingerprint for your known_hosts / trust anchor
ssh-keygen -l -f /etc/ssh/initrd_ssh_host_ed25519_key.pub
```

---

## 2. Clevis LUKS enrollment

Binds each LUKS slot to the tang server (`192.168.1.22:7500`) so clevis can unlock at boot. Must be done while the live system is running (devices already open). You'll be prompted to confirm the tang server's JWK thumbprint.

```bash
# cryptroot
sudo clevis luks bind \
  -d /dev/disk/by-uuid/7ee92ea7-b3df-4a8c-8d32-d8326a944a2b \
  tang '{"url":"http://192.168.1.22:7500"}'

# cryptswap
sudo clevis luks bind \
  -d /dev/disk/by-uuid/e1116d79-ce06-4a68-bed7-31c7ffaf95a1 \
  tang '{"url":"http://192.168.1.22:7500"}'
```

Verify enrollment — each device should list a `clevis` token:

```bash
sudo clevis luks list -d /dev/disk/by-uuid/7ee92ea7-b3df-4a8c-8d32-d8326a944a2b
sudo clevis luks list -d /dev/disk/by-uuid/e1116d79-ce06-4a68-bed7-31c7ffaf95a1
```

> **Note:** JWE tokens are stored in the LUKS header via `luksmeta`. `clevis-initrd.nix` calls `clevis luks unlock` directly against the block device at boot — no separate token export is needed.

---

## 3. Tailscale initrd state file

> **Optional** — skip this if LAN-only SSH access is sufficient.

Creates a persistent Tailscale node identity for the initrd environment, separate from the live system node. Run `tailscaled` temporarily out-of-band to auth once; the resulting state file preserves identity across reboots.

```bash
sudo mkdir -p /etc/tailscale-initrd

# Start a temporary tailscaled against the initrd state path.
# --tun=userspace-networking avoids needing a real TUN device.
sudo tailscaled \
  --state=/etc/tailscale-initrd/state.json \
  --socket=/run/tailscale-initrd-setup.sock \
  --tun=userspace-networking &

# Authenticate — generate a key at https://login.tailscale.com/admin/settings/keys
# Use ephemeral=false so the node persists across reboots.
sudo tailscale \
  --socket=/run/tailscale-initrd-setup.sock \
  up \
  --auth-key=tskey-auth-xxxxxx \
  --hostname=rocinante-initrd

# Note the assigned Tailscale IP
sudo tailscale --socket=/run/tailscale-initrd-setup.sock ip -4

# Tear down the temporary daemon
sudo tailscale --socket=/run/tailscale-initrd-setup.sock down
sudo pkill -f "tailscaled.*tailscale-initrd-setup"

# Protect the state file
sudo chmod 400 /etc/tailscale-initrd/state.json
```

Then update `configuration.nix`:

1. Set `sshInitrd.tailscaleIp` to the IP noted above
2. Set `sshInitrd.listenOn.tailscale = true`
3. Rebuild and reboot

---

## 4. First-boot verification

SSH into the initrd from another machine on the LAN:

```bash
ssh -i ~/.ssh/your_key root@192.168.1.6
```

Inside the initrd, check that clevis unlocked automatically:

```bash
systemctl status clevis-decrypt-cryptroot
systemctl status clevis-decrypt-cryptswap
```

If clevis failed (tang unreachable, enrollment issue, etc.), unlock manually:

```bash
initrd-unlock
```

After unlocking, the initrd switches root automatically and boot continues.
