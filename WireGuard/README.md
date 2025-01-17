# Setup WireGuard using PiVPN

## 1. Install PiVPN

```bash
curl -L https://install.pivpn.io | bash
```

## 2. Configure PiVPN

```bash
pivpn add
```

#### 2.1 Configure WireGuard
Follow the installer instructions.

### 3. Create a WireGuard Client

```bash
pivpn -a
```
Show the QR code for the client.

```bash
pivpn -qr
```

### 4. Connect to the VPN

Scan the QR code with the WireGuard app on your device.

