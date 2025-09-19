# Omada Controller Installation on Debian 13 Cloud

This guide documents the installation process of **TP-Link Omada Controller** on a Debian 13 Cloud VM. It is based on a real-world deployment on Proxmox with Debian Cloud image.

---

## 1. System Environment
- **OS**: Debian 13 Cloud image (no LVM by default).
- **Hypervisor**: Proxmox (disk resized via GUI).
- **Target Software**: TP-Link Omada Controller (latest version).

---

## 2. Disk Expansion
### Problem
During installation, an error occurred:
```
No space left on device in /opt
```

### Solution
1. Increased disk size in **Proxmox**.
2. Verified that the VM uses a **single root partition** (no LVM).
3. Resized filesystem:
   ```bash
   sudo growpart /dev/sda 1
   sudo resize2fs /dev/sda1
   ```
4. Verified new space:
   ```bash
   df -h /
   ```

---

## 3. Java Installation
### Problem
Debian 13 provides only **OpenJDK 21**, but Omada requires **Java 17**.

### Solution
1. Added Debian 12 (Bookworm) repo temporarily:
   ```bash
   echo "deb http://deb.debian.org/debian bookworm main" | sudo tee /etc/apt/sources.list.d/bookworm.list
   sudo apt update
   ```
2. Installed Java 17 and jsvc:
   ```bash
   sudo apt install -y openjdk-17-jre-headless jsvc
   ```
3. Verified installation:
   ```bash
   java -version
   ```
   Expected output:
   ```
   openjdk version "17..."
   ```

---

## 4. MongoDB Installation
Omada requires a specific MongoDB version. Installed via official MongoDB repo.

1. Import MongoDB GPG key:
   ```bash
   curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-6.0.gpg
   ```
2. Add MongoDB repository:
   ```bash
   echo "deb [signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/6.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
   sudo apt update
   ```
3. Install MongoDB:
   ```bash
   sudo apt install -y mongodb-org
   ```
4. Start and enable MongoDB:
   ```bash
   sudo systemctl enable --now mongod
   ```

---

## 5. Network Configuration
### Initial config
```yaml
network:
  version: 2
  ethernets:
    eth0:
      match:
        macaddress: "bc:24:11:4b:39:01"
      dhcp4: true
      set-name: "eth0"
```

### Added second interface (ens19)
```yaml
network:
  version: 2
  ethernets:
    eth0:
      match:
        macaddress: "bc:24:11:4b:39:01"
      dhcp4: true
      set-name: "eth0"
    ens19:
      dhcp4: true
```

Apply changes:
```bash
sudo netplan apply
```

---

## 6. Omada Installation
1. Download the latest `.deb` package from TP-Link:
   ```bash
   wget https://static.tp-link.com/upload/software/omada/controller/omada-sdn-controller-x.x.x-linux-x64.deb
   ```
   *(replace `x.x.x` with latest version)*

2. Install package:
   ```bash
   sudo dpkg -i omada-sdn-controller-*.deb
   sudo apt-get -f install -y
   ```

3. Start and enable service:
   ```bash
   sudo systemctl enable --now omada
   ```

4. Check status:
   ```bash
   systemctl status omada
   ```

---

## 7. Access Control Note
Omada supports **client access restrictions per AP**, allowing configuration to permit only specific clients if needed.

---

## Troubleshooting
- **Disk full error** → expand Proxmox disk, run `resize2fs`.
- **Java not found** → ensure Bookworm repo enabled, install Java 17.
- **MongoDB service not starting** → check logs with `journalctl -u mongod`.

---

## Result
Omada Controller successfully installed and running on Debian 13 Cloud with:
- Expanded disk.
- Java 17 from Bookworm repo.
- MongoDB 6.0 from official repo.
- Latest Omada `.deb` package.

