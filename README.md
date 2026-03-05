
# OpenVPN Lab — Site-to-Client VPN Setup

A hands-on lab where I built a fully functional VPN from scratch using OpenVPN and Easy-RSA on two Linux virtual machines. This includes setting up a Certificate Authority, issuing certificates, configuring the server and client, and verifying encrypted tunnel traffic.

---

## Lab Environment

| Role | OS | IP (Host-Only) | VPN IP |
|---|---|---|---|
| VPN Server | Ubuntu 24.04 | 192.168.56.101 | 10.8.0.1 |
| VPN Client | Kali Linux | 192.168.56.103 | 10.8.0.6 |

**Virtualization:** VirtualBox with NAT + Host-Only Adapter configuration

---

## What I Built

- A full **PKI (Public Key Infrastructure)** using Easy-RSA
- A self-signed **Certificate Authority (CA)**
- Signed **server and client certificates**
- A **TLS Auth key** for additional security against unauthorized connection attempts
- A working **OpenVPN encrypted tunnel** between Ubuntu (server) and Kali (client)
- Verified encrypted traffic flowing through the tunnel via ping on the `10.8.0.0/24` VPN subnet

---

## Network Diagram

```
+-------------------+          VPN Tunnel (UDP 1194)         +-------------------+
|   Ubuntu 24.04    |  <----------------------------------->  |    Kali Linux     |
|   VPN Server      |       Encrypted - AES-256-GCM          |    VPN Client     |
|  192.168.56.101   |                                         |  192.168.56.103   |
|    10.8.0.1       |                                         |    10.8.0.6       |
+-------------------+                                         +-------------------+
```

---

## Steps Taken

### 1. Environment Setup
- Created two VMs in VirtualBox
- Configured Adapter 1 as NAT (internet access) and Adapter 2 as Host-Only (VM-to-VM communication)
- Verified connectivity between VMs using `ping`

### 2. Installed OpenVPN and Easy-RSA
```bash
sudo apt update && sudo apt install openvpn easy-rsa -y
```

### 3. Built the Certificate Authority
```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
```
- Initialized the PKI directory structure
- Created the CA certificate (`ca.crt`) and private key (`ca.key`)

### 4. Generated and Signed the Server Certificate
```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
```

### 5. Generated Diffie-Hellman Parameters
```bash
./easyrsa gen-dh
```
- Required for secure key exchange during VPN handshake

### 6. Generated and Signed the Client Certificate
```bash
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

### 7. Generated TLS Auth Key
```bash
openvpn --genkey secret ~/openvpn-ca/pki/ta.key
```
- Adds a pre-shared key layer — any connection without this key is dropped before TLS handshake

### 8. Copied Files to OpenVPN Directory
```bash
sudo cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/
sudo cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/
sudo cp ~/openvpn-ca/pki/ta.key /etc/openvpn/
```

### 9. Configured the Server
- Created `/etc/openvpn/server.conf` (see [configs/server.conf](configs/server.conf))
- Started and enabled the OpenVPN service:
```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

### 10. Transferred Client Files to Kali via SCP
```bash
scp ~/openvpn-ca/pki/ca.crt IQ@192.168.56.103:~/vpn/
scp ~/openvpn-ca/pki/issued/client1.crt IQ@192.168.56.103:~/vpn/
scp ~/openvpn-ca/pki/private/client1.key IQ@192.168.56.103:~/vpn/
scp ~/openvpn-ca/pki/ta.key IQ@192.168.56.103:~/vpn/
```

### 11. Configured and Connected the Client
- Created `~/vpn/client.ovpn` on Kali (see [configs/client.ovpn](configs/client.ovpn))
- Connected to the VPN:
```bash
sudo openvpn --config ~/vpn/client.ovpn
```

### 12. Verified the Tunnel
```bash
# On Kali - confirmed VPN IP assigned
ip a show tun0
# Result: 10.8.0.6

# Pinged server through encrypted tunnel
ping 10.8.0.1
# Result: successful replies through VPN tunnel
```

---

## Problems I Hit and How I Fixed Them

| Problem | Cause | Fix |
|---|---|---|
| `gen-req` failed with PEM error | Typed `nopaass` instead of `nopass` | Re-ran command with correct spelling |
| Client couldn't find `ta.key` | Config used relative path, OpenVPN looked in wrong directory | Updated config to use full absolute paths |
| Cipher mismatch warning | Old `cipher` directive deprecated in OpenVPN 2.6 | Replaced with `data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305` |
| VPN connection hanging, no response from server | Server had stopped running | Restarted with `sudo systemctl restart openvpn@server` |
| Kali lost its Host-Only IP | VirtualBox network reset | Manually re-assigned with `sudo ip addr add 192.168.56.103/24 dev eth1` |

---

## Key Concepts Learned

- **PKI (Public Key Infrastructure)** — how certificates establish trust between systems
- **Certificate Authority** — issuing and signing certificates to verify identities
- **TLS Authentication** — pre-shared key layer that drops unauthorized connections before handshake
- **Diffie-Hellman** — secure key exchange without transmitting the key
- **AES-256-GCM** — encryption standard securing the VPN data channel
- **tun interface** — virtual network interface created by OpenVPN for the tunnel
- **SCP** — secure file transfer between Linux machines

---

## Config Files

- [configs/server.conf](configs/server.conf) — OpenVPN server configuration
- [configs/client.ovpn](configs/client.ovpn) — OpenVPN client configuration

---

## Tools Used

- OpenVPN 2.6
- Easy-RSA 3.1.7
- VirtualBox
- Ubuntu 24.04
- Kali Linux
- OpenSSL 3.x
