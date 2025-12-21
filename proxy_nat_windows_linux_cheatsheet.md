# Proxy & NAT Cheat Sheet (Windows & Linux)
Senior Sysadmin Reference

---

## 1. Concepts at a Glance

| Concept | OSI Layer | Purpose |
|------|-----------|--------|
| IP Forwarding | L3 | Route packets between interfaces |
| NAT (SNAT/DNAT) | L3/L4 | Translate IP/port |
| Explicit Proxy | L7 | Client-configured proxy |
| Transparent Proxy | L3/L7 | Intercept traffic |
| Reverse Proxy | L7 | Publish services |
| Port Forwarding | L4 | Expose internal service |

---

## 2. WINDOWS

### 2.1 Enable IP Forwarding

```powershell
Get-NetIPInterface -AddressFamily IPv4
Set-NetIPInterface -InterfaceAlias "Ethernet" -Forwarding Enabled
```

Optional global router flag:
```powershell
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters `
 -Name IPEnableRouter -Value 1
```

---

### 2.2 WinNAT – Outbound NAT

Create NAT:
```powershell
New-NetNat -Name "LAN-NAT" -InternalIPInterfaceAddressPrefix 192.168.50.0/24
```

List:
```powershell
Get-NetNat
```

Remove:
```powershell
Remove-NetNat -Name "LAN-NAT"
```

---

### 2.3 WinNAT – Port Forwarding (DNAT)

```powershell
Add-NetNatStaticMapping `
 -NatName "LAN-NAT" `
 -Protocol TCP `
 -ExternalPort 8080 `
 -InternalIPAddress 192.168.50.10 `
 -InternalPort 80
```

Firewall:
```powershell
New-NetFirewallRule -DisplayName "Allow 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

---

### 2.4 portproxy (TCP relay)

```cmd
netsh interface portproxy add v4tov4 ^
 listenaddress=0.0.0.0 listenport=8080 ^
 connectaddress=192.168.50.10 connectport=80
```

List:
```cmd
netsh interface portproxy show all
```

Delete:
```cmd
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

⚠ TCP only, no NAT, no UDP.

---

### 2.5 Squid Proxy on Windows (Explicit)

```conf
http_port 3128
acl localnet src 192.168.50.0/24
http_access allow localnet
http_access deny all
```

Client:
```
Proxy: 192.168.50.1
Port: 3128
```

---

## 3. LINUX

### 3.1 Enable IP Forwarding

```bash
sysctl -w net.ipv4.ip_forward=1
```

Persist:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

---

### 3.2 iptables – Outbound NAT

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
```

---

### 3.3 iptables – Port Forwarding

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080  -j DNAT --to-destination 192.168.50.10:80

iptables -A FORWARD -p tcp -d 192.168.50.10 --dport 80 -j ACCEPT
```

---

### 3.4 Transparent Proxy (HTTP)

```bash
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80  -j REDIRECT --to-port 3128
```

Squid:
```conf
http_port 3128 intercept
```

---

### 3.5 HTTPS Interception (SSL Bump – Advanced)

```conf
https_port 3129 intercept ssl-bump cert=/etc/squid/ca.pem key=/etc/squid/ca.key
ssl_bump peek all
ssl_bump bump all
```

⚠ Requires CA installed on clients.

---

### 3.6 nftables Equivalent

```bash
nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oif eth0 masquerade
```

---

## 4. Comparison Table

| Feature | Windows | Linux |
|------|--------|------|
| Kernel NAT | WinNAT | iptables/nft |
| Transparent Proxy | ❌ | ✅ |
| SSL Bump | ❌ | ✅ |
| Port Forwarding | ✅ | ✅ |
| Performance | Medium | High |
| Production Gateway | ❌ | ✅ |

---

## 5. Best Practices

- Windows: Explicit proxy + WinNAT
- Linux: nftables + Squid intercept
- Avoid portproxy for heavy traffic
- Always monitor disk & memory
- Never delete proxy cache manually

---

## 6. Decision Guide

| Need | Choose |
|----|------|
| Small lab | Windows WinNAT |
| Explicit proxy | Windows/Linux |
| Transparent HTTPS | Linux only |
| High throughput | Linux |
| Enterprise gateway | Linux / Firewall appliance |

---

## 7. Final Rule

> Routing stays in kernel = fast  
> Proxying enters user-space = slower  

---

Author: Senior Sysadmin Cheat Sheet
