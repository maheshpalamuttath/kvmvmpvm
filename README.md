# KVM Virtualization Management with php-virt-manager on Debian

A step-by-step guide to setting up **KVM/QEMU, libvirt and php-virt-manager** on a Debian Linux server and managing virtual machines through a web browser.

This setup provides a lightweight web-based alternative to desktop virtualization management tools such as Virtual Machine Manager (virt-manager).

## Overview

The setup uses the following components:

| Component            | Purpose                          |
| -------------------- | -------------------------------- |
| Debian               | Host operating system            |
| KVM/QEMU             | Hardware-assisted virtualization |
| libvirt              | Virtualization management        |
| Docker Compose       | Container deployment             |
| php-virt-manager     | Web-based VM management          |
| Linux Bridge (`br0`) | Connect VMs directly to the LAN  |
| `virbr0`             | Default NAT network              |

### Architecture

```text
                 Web Browser
                      |
                HTTPS :9443
                      |
              php-virt-manager
                  (Docker)
                      |
              /var/run/libvirt
                      |
                libvirt
                      |
                  KVM/QEMU
                      |
             Debian Host Server
                      |
             +--------+--------+
             |                 |
          virbr0              br0
          (NAT)             (Bridge)
             |                 |
        NAT Network       Physical LAN
             |                 |
          Internet         LAN Router
```

## 1. Install KVM and libvirt

Update the Debian package repositories:

```bash
sudo apt update
```

Install the required packages:

```bash
sudo apt install -y qemu-system libvirt-daemon-system libvirt-clients
```

Enable and start libvirt:

```bash
sudo systemctl enable --now libvirtd
```

Verify the libvirt connection:

```bash
sudo virsh -c qemu:///system list --all
```

## 2. Deploy php-virt-manager

Create a project directory:

```bash
mkdir -p ~/php-virt-manager
cd ~/php-virt-manager
```

Create the Docker Compose configuration:

```bash
nano compose.yaml
```

Add the following configuration:

```yaml
services:
  php-virt-manager:
    image: ghcr.io/spidero/php-virt-manager:latest
    container_name: php-virt-manager
    network_mode: host
    restart: unless-stopped
    environment:
      PVM_USER: admin
      PVM_PASSWORD: admin123
      HTTPS_PORT: "9443"
      HTTP_PORT: "9080"
    volumes:
      - /var/run/libvirt:/var/run/libvirt
      - pvm-data:/app/data
      - pvm-ssl:/etc/nginx/ssl

volumes:
  pvm-data:
  pvm-ssl:
```

**Important:** Change the default username and password before exposing the management interface to your network.

Deploy the container:

```bash
docker compose up -d
```

Check the container status:

```bash
docker ps
```

### Access the web interface

Open the following URL in your browser:

```text
https://SERVER-IP:9443/
```

Log in using the credentials configured in the Compose file.

You may encounter a self-signed SSL certificate warning when accessing the interface.

## 3. Configure libvirt authentication

To allow php-virt-manager to communicate with the system libvirt socket, configure Unix socket permissions.

Open the libvirt configuration:

```bash
sudo nano /etc/libvirt/libvirtd.conf
```

Add the following settings at the bottom:

```ini
auth_unix_rw = "none"
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```

These settings disable authentication on the read-write Unix socket and allow members of the `libvirt` group to access it.

The php-virt-manager Docker image is designed to handle the socket group access.

Restart libvirt:

```bash
sudo systemctl restart libvirtd
```

Check the socket permissions:

```bash
ls -l /var/run/libvirt/libvirt-sock
```

Restart php-virt-manager:

```bash
docker restart php-virt-manager
```

Refresh the web interface:

```text
https://SERVER-IP:9443/
```

**Security note:** `auth_unix_rw = "none"` removes authentication for clients accessing this socket. Only use this configuration on a trusted host, restrict access to the socket, and do not expose the libvirt socket or management interface to untrusted networks.

## 4. Configure the default storage pool

Initially, php-virt-manager may report that no active storage pool is available.

Libvirt uses storage pools to organise virtual machine disk images.

### Step 1. Create the storage directory

```bash
sudo mkdir -p /var/lib/libvirt/images
```

Set directory ownership and permissions:

```bash
sudo chown root:root /var/lib/libvirt/images
sudo chmod 755 /var/lib/libvirt/images
```

### Step 2. Define the default storage pool

```bash
sudo virsh pool-define-as default dir \
--target /var/lib/libvirt/images
```

### Step 3. Start the storage pool

```bash
sudo virsh pool-start default
```

Enable automatic startup:

```bash
sudo virsh pool-autostart default
```

### Step 4. Verify

```bash
sudo virsh pool-list --all
```

Expected output:

```text
Name      State    Autostart
--------------------------------
default   active   yes
```

The default storage pool is now ready for virtual machine disk images.

## 5. Enable the default NAT network

Libvirt provides a default virtual network using NAT.

This allows virtual machines to access the internet through the Debian host.

### Step 1. Check available networks

```bash
sudo virsh net-list --all
```

### Step 2. Start the default network

```bash
sudo virsh net-start default
```

Enable automatic startup:

```bash
sudo virsh net-autostart default
```

### Step 3. Verify

```bash
sudo virsh net-list --all
```

Expected configuration:

```text
Name      State    Autostart
--------------------------------
default   active   yes
```

Check the virtual bridge:

```bash
ip addr show virbr0
```

The default network typically uses:

- Bridge: `virbr0`
- Network: `192.168.122.0/24`
- Gateway: `192.168.122.1`
- DHCP: Enabled

Virtual machines connected to this network receive private IP addresses and access the internet through NAT.

## 6. Create a Linux bridge (br0)

A Linux bridge allows virtual machines to connect directly to your physical LAN.

Unlike NAT networking, bridged virtual machines can obtain IP addresses from your physical router and communicate with other devices on the LAN.

### Part A: Create the Linux bridge

#### Step 1. Identify the physical network interface

Run:

```bash
ip -br link
```

Example:

```text
lo       UNKNOWN
enp2s0   UP
```

In this example, `enp2s0` is the physical Ethernet interface.

Your interface name may differ.

#### Step 2. Create the bridge connection

Create a bridge named `br0`:

```bash
sudo nmcli connection add type bridge \
ifname br0 con-name br0 \
ipv4.method manual \
ipv4.addresses 192.168.29.2/24 \
ipv4.gateway 192.168.29.1 \
ipv4.dns "8.8.8.8" \
ipv6.method auto \
bridge.stp no
```

**Adjust the IP configuration** according to your LAN network.

For this example:

| Setting | Value           |
| ------- | --------------- |
| Bridge  | br0             |
| Host IP | 192.168.29.2    |
| Subnet  | 192.168.29.0/24 |
| Gateway | 192.168.29.1    |
| DNS     | 8.8.8.8         |

#### Step 3. Attach the physical Ethernet interface

List existing NetworkManager connections:

```bash
nmcli connection show
```

Identify the existing Ethernet connection profile.

For example:

```text
Wired connection 1
```

Attach it to the bridge:

```bash
sudo nmcli connection modify "Wired connection 1" \
master br0 slave-type bridge
```

The physical interface will now act as a bridge port rather than holding the host's IP address directly.

#### Step 4. Enable automatic activation

Enable autoconnect for the bridge:

```bash
sudo nmcli connection modify br0 \
connection.autoconnect yes
```

Enable autoconnect for the Ethernet connection:

```bash
sudo nmcli connection modify "Wired connection 1" \
connection.autoconnect yes
```

#### Step 5. Activate the bridge

```bash
sudo nmcli connection up br0
```

Activate the Ethernet connection:

```bash
sudo nmcli connection up "Wired connection 1"
```

**Network interruption warning:** Activating the bridge may temporarily disconnect your SSH session. Ensure you have local console access or another recovery method before proceeding.

#### Step 6. Verify the bridge

Check network interfaces:

```bash
ip -br addr
```

Expected result:

```text
enp2s0   UP
br0      UP    192.168.29.2/24
```

Check bridge membership:

```bash
bridge link
```

You should see the physical Ethernet interface attached to `br0`.

### Part B: Register br0 with libvirt

Creating the Linux bridge makes it available at the host operating-system level.

However, it must also be registered with libvirt to make it appear as a selectable network in php-virt-manager.

#### Step 7. Create the libvirt network configuration

Create an XML configuration file:

```bash
sudo nano /tmp/br0.xml
```

Add:

```xml
<network>
  <name>br0</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

Save the file.

#### Step 8. Register the network

```bash
sudo virsh net-define /tmp/br0.xml
```

#### Step 9. Start the network

```bash
sudo virsh net-start br0
```

#### Step 10. Enable automatic startup

```bash
sudo virsh net-autostart br0
```

#### Step 11. Verify

```bash
sudo virsh net-list --all
```

Expected output:

```text
Name      State    Autostart   Persistent
------------------------------------------
br0       active   yes         yes
default   active   yes         yes
```

Your Linux bridge is now registered with libvirt.

## 7. Create a virtual machine using php-virt-manager

Open the web interface:

```text
https://SERVER-IP:9443/
```

Follow these steps:

1. Open php-virt-manager.
2. Connect to the system libvirt instance.
3. Navigate to the virtual machine creation interface.
4. Select the installation ISO or other supported installation source.
5. Configure CPU and memory allocation.
6. Select the default storage pool.
7. Create the virtual disk.
8. Configure the virtual machine network.

### Select the network interface

You can select either of the following networks:

| Network   | Purpose                               |
| --------- | ------------------------------------- |
| `default` | NAT networking with internet access   |
| `br0`     | Direct connection to the physical LAN |

For a LAN-accessible server VM, select `br0`.

The virtual machine can then obtain an IP address directly from your router's DHCP server, provided DHCP is enabled on the LAN.

## 8. Useful management commands

### Libvirt

```bash
sudo systemctl status libvirtd
sudo virsh list --all
sudo virsh nodeinfo
```

### Storage pools

```bash
sudo virsh pool-list --all
sudo virsh pool-info default
```

### Virtual networks

```bash
sudo virsh net-list --all
sudo virsh net-info default
sudo virsh net-info br0
```

### Docker

```bash
docker ps
docker logs php-virt-manager
docker restart php-virt-manager
docker compose down
docker compose up -d
```

### Remove cloud image from php-virt-manager

```bash
cd /var/lib/libvirt/images
sudo rm cloud-debian-12.qcow2
sudo virsh vol-delete cloud-debian-12.qcow2 --pool default
```

### Add iso images to /var/lib/libvirt/images

```bash
cp debian-12.15.0-amd64-netinst.iso /var/lib/libvirt/images
#Refresh the storage pool:
sudo virsh pool-refresh default
```

## 9. Final configuration checklist

- [ ] KVM/QEMU installed.
- [ ] libvirt service enabled and running.
- [ ] php-virt-manager deployed using Docker Compose.
- [ ] libvirt Unix socket permissions configured.
- [ ] Default storage pool created and activated.
- [ ] Default NAT network enabled.
- [ ] Linux bridge `br0` created.
- [ ] Physical Ethernet interface attached to `br0`.
- [ ] `br0` registered with libvirt.
- [ ] Both NAT and bridge networks visible in php-virt-manager.
- [ ] Test virtual machine created successfully.

## Conclusion

With Debian, KVM/QEMU, libvirt and php-virt-manager, you can build a lightweight virtualization host and manage virtual machines through a browser.

The combination of NAT networking and Linux bridge networking provides flexibility for running isolated test environments as well as LAN-accessible servers.

This setup is particularly useful for homelabs, self-hosting, Linux server administration, and virtualization learning.

---

**Author:** Mahesh Palamuttath
**Website:** [https://maheshpalamuttath.info](https://maheshpalamuttath.info)

**Topics:** Debian, KVM, QEMU, libvirt, php-virt-manager, Docker, Linux Bridge, Virtualization, Homelab, Self-Hosting.

