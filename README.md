# US OpenVPN Server Setup from Scratch (US Server)

## US Server

Step 1: Launch EC2 Instance
- EC2 → Launch Instance → Ubuntu 22.04 LTS
- Instance type: t2.micro
- Key pair: No key (use EC2 Instance Connect)
- Subnet: US region
- Auto-assign Public IP: Enable

Security Group:
| Protocol | Port | Source |
| --- | ---: | --- |
| SSH | 22 | Anywhere (for EC2 Connect) |
| UDP | 1194 | Your client IP (or 0.0.0.0/0 for testing) |
| ICMP | - | Anywhere (optional) |

Step 2: Connect via EC2 Instance Connect
- EC2 → Select instance → Connect → EC2 Instance Connect → Connect  
- Logged in as ubuntu.

Step 3: Update System & Install OpenVPN
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y
```

Step 4: Set Up Easy-RSA PKI
```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
```

Step 5: Build Certificate Authority (CA) Without Passphrase
```bash
./easyrsa build-ca nopass
```
- Common Name: Press Enter (default: Easy-RSA CA)

Step 6: Generate Server Key & Certificate Request
```bash
./easyrsa gen-req server nopass
```
- Common Name: Press Enter (or type server)

Step 7: Sign Server Certificate
```bash
./easyrsa sign-req server server
```
- Type `yes` to confirm  
- pki/issued/server.crt will be created

Step 8: Generate Diffie-Hellman Parameters
```bash
./easyrsa gen-dh
```

Step 9: Generate HMAC Key
```bash
openvpn --genkey secret ta.key
```
- Creates ta.key for TLS authentication

Step 10: Copy Keys to OpenVPN Directory
```bash
sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem ta.key /etc/openvpn/
```

Step 11: Create OpenVPN Server Configuration
```bash
sudo nano /etc/openvpn/server.conf
```
Paste:
```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
cipher AES-256-CBC
server 10.8.0.0 255.255.255.0
keepalive 10 120
persist-key
persist-tun
status openvpn-status.log
verb 3
```
Save & exit: Ctrl+O → Enter, Ctrl+X → Exit

Step 12: Enable IP Forwarding
```bash
sudo nano /etc/sysctl.conf
```
- Uncomment the line:
```
net.ipv4.ip_forward=1
```
Save & exit. Apply changes:
```bash
sudo sysctl -p
```

Step 13: Configure Firewall
```bash
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
sudo ufw enable
```
- Type `y` to confirm enabling UFW

Step 14: Start OpenVPN Server
```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

Step 15: Verify VPN Interface
Option 1 (recommended):
```bash
ip addr show tun0
```
Option 2 (if ifconfig is available):
```bash
sudo apt install net-tools -y
ifconfig tun0
```
- You should see subnet 10.8.0.0/24 — VPN is running and ready for clients

![SL Image](https://drive.google.com/file/d/1IYnB5CgQCjrYig-pzHLWgOOHXEYpTm2o/view?usp=drive_link)

---

# SL OpenVPN Client Setup (SL EC2)

Step 1: Launch EC2 Instance
- EC2 → Launch Instance → Ubuntu 22.04 LTS
- Instance type: t2.micro
- Key pair: SL1.pem
- Subnet: SL region
- Auto-assign Public IP: Enable

Security Group:
| Protocol | Port | Source |
| --- | ---: | --- |
| SSH | 22 | Your IP |
| UDP | 1194 | US server public IP |

Step 2: Connect via SSH
```bash
ssh -i ~/openvpn-ca/SL1.pem ubuntu@<SL_PUBLIC_IP>
```
- Logged in as ubuntu.

Step 3: Update System & Install OpenVPN
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn -y
```

Step 4: Transfer Certificates from US Server
From the US server (adjust paths and IPs as needed):
```bash
scp -i ~/openvpn-ca/SL1.pem \
    pki/ca.crt pki/issued/sl-client.crt pki/private/sl-client.key ta.key \
    ubuntu@<SL_PUBLIC_IP>:/home/ubuntu/
```

Step 5: Move Certificates to OpenVPN Directory
```bash
sudo mkdir -p /etc/openvpn/client
sudo mv ~/ca.crt ~/sl-client.crt ~/sl-client.key ~/ta.key /etc/openvpn/client/
```

Step 6: Create OpenVPN Client Configuration
```bash
sudo nano /etc/openvpn/client/sl-client.ovpn
```
Paste:
```
client
dev tun
proto udp
remote <US_SERVER_PUBLIC_IP> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca /etc/openvpn/client/ca.crt
cert /etc/openvpn/client/sl-client.crt
key /etc/openvpn/client/sl-client.key
tls-auth /etc/openvpn/client/ta.key 1
cipher AES-256-CBC
verb 3
```
Save & exit: Ctrl+O → Enter → Ctrl+X

Step 7: Start OpenVPN Client
```bash
sudo openvpn --config /etc/openvpn/client/sl-client.ovpn
```
- Look for "Initialization Sequence Completed"

Step 8: Verify VPN Interface & Connectivity
```bash
ip addr show tun0
ping 10.8.0.1
```

![SL Image](https://drive.google.com/file/d/1rPhCsiVhzmSZAT1xPMzTd096kazkd_6p/view?usp=drive_link)

---
