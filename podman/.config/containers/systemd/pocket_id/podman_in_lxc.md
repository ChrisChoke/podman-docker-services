# Setup Guide: Pocket ID (Podman Quadlet in Proxmox LXC on ZFS)

## 1. Proxmox VE Host (Enable LXC Features & TUN Device)

Enable the following in the LXC options on your Proxmox Host under **Features**:
* **Nesting** (Required for container-in-container / namespace isolation)
* **Keyctl** (Required for Systemd user keyrings)
* **FUSE** (Required for `fuse-overlayfs` on ZFS filesystems)

Alternatively, execute this directly in the CLI on the **Proxmox VE Host**:

```bash
# Open LXC configuration (replace <VMID> with your LXC ID, e.g., 104)
nano /etc/pve/lxc/<VMID>.conf
```

Add the features, as well as the `cgroup2` and `mount` rules, at the end of the file:
```text
features: keyctl=1,nesting=1,fuse=1
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Restart the LXC container to apply all kernel modules and passthroughs:
```bash
pct reboot <VMID>
```

---

## 2. LXC System Setup (as root)

Execute inside the **LXC container** as `root`:

```bash
# Update package lists and install dependencies
apt update && apt install -y podman fuse-overlayfs systemd-container openssl

# Create an unprivileged user and enable systemd linger
useradd -m -s /bin/bash podmanuser
loginctl enable-linger podmanuser

# Switch to the user session (important: use machinectl for a proper systemd session)
machinectl shell podmanuser@
```

---

## 3. Podman ZFS Storage Configuration (as podmanuser)

Configure `fuse-overlayfs` for ZFS compatibility:

```bash
mkdir -p ~/.config/containers

cat << 'EOF' > ~/.config/containers/storage.conf
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
mountopt = "metacopy=off"
EOF
```

---

## 4. Pocket ID Quadlet Deployment (as podmanuser)

Create directories, generate an encryption key, and create the Quadlet file:

```bash
# Create directories
mkdir -p ~/.config/containers/systemd/ ~/pocket-id-data

# Generate a random Base64 key
KEY=$(openssl rand -base64 32)

# Create the Quadlet file
cat << EOF > ~/.config/containers/systemd/pocket-id.container
[Unit]
Description=Pocket ID OIDC Engine
After=network-online.target

[Container]
Image=ghcr.io/pocket-id/pocket-id:latest
ContainerName=pocket-id
Environment=APP_URL=https://auth.example.com
Environment=ENCRYPTION_KEY=${KEY}
PublishPort=1411:1411
# Important: Bind mounts explicitly require %h/ or ./ (otherwise it becomes a Named Volume)
Volume=%h/pocket-id-data:/app/data:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF

# Run systemd generator and start the service
systemctl --user daemon-reload
systemctl --user start pocket-id

# Check status and container list
systemctl --user status pocket-id
podman container ls
```

---

## 5. Traefik Dynamic Configuration (File Provider)

Create this file on your **Traefik VM** in the file provider directory (e.g., `/etc/traefik/dynamic/pocket-id.yaml`):

```yaml
http:
  routers:
    pocket-id:
      rule: "Host(`id.example.com`)"
      entryPoints:
        - "websecure"
      service: "pocket-id-svc"
      tls:
        certResolver: "myresolver"

  services:
    pocket-id-svc:
      loadBalancer:
        servers:
          - url: "http://<IP-OF-POCKET-ID-LXC>:1411"
```