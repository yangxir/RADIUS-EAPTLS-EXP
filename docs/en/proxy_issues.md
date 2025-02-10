# 📡 RADIUS Proxy Server Authentication Guide

## 📌 1. Role of the Proxy Server
### 🔹 Why is a Proxy Server Needed?
In the RADIUS authentication architecture, **Proxy Server** is mainly used to **forward** authentication requests rather than processing authentication directly. Its core role is to ensure that client authentication requests can reach the final RADIUS server in a **distributed environment**.

Proxy Server is **essential** in the following scenarios:
- **Enterprise WiFi authentication** (e.g., Eduroam multi-site roaming)
- **Multi-level RADIUS authentication** (e.g., ISP authentication systems)
- **Cross-national/cross-regional WiFi authentication** (authentication forwarding between RADIUS servers in different countries)

---

## 📡 2. Proxy Server Workflow
1️⃣ **Client** sends an **EAP-TLS** authentication request  
2️⃣ **Proxy Server** receives the request and **forwards** it to the final RADIUS Server  
3️⃣ **RADIUS Server** processes the authentication request and returns the result  
4️⃣ **Proxy Server** sends the result back to the client  
5️⃣ **Client successfully connects to WiFi** 🎉  

---

## ⚙️ 3. Proxy Server Configuration

### 📍 3.1 Device Environment
| Device | Role | IP Address |
|------|------|---------|
| **MiniPC-1** | Authentication Client (EAP-TLS) | `192.168.8.220` |
| **Router (GL-AXT1800-83b-5G)** | Connects all devices, Proxy acts as RADIUS server | N/A |
| **MiniPC-2** | Proxy Server (Forwards RADIUS authentication requests) | `192.168.8.182` |
| **PC (RADIUS Server)** | Authentication Server (FreeRADIUS 3.0.26) | `192.168.8.195` |

---

### 📍 3.2 Ensure Proxy Server Can Connect to RADIUS Server

**Test UDP 1812 Port Connectivity**
```sh
nc -zvu 192.168.8.195 1812
```
✅ **UDP port 1812 is reachable**, indicating Proxy Server can communicate with RADIUS Server.

---

### 📍 3.3 Configure Proxy Server
#### 🔹 1. Modify `proxy.conf`
```ini
proxy_requests = yes

realm DEFAULT {
    type = radius
    authhost = 192.168.8.195:1812
    accthost = 192.168.8.195:1813
    secret = shared_secret_key
}
```
✅ **Ensure Proxy Server acts only as a "relay" and not a "local authentication" server**

---

#### 🔹 2. Configure RADIUS Server to Allow Proxy
Add Proxy Server to `clients.conf`:
```ini
client 192.168.8.182 {
    secret = shared_secret_key
    shortname = proxy
}
```
✅ **Ensure the `secret` is consistent between Proxy Server and RADIUS Server to avoid errors**

---

#### 🔹 3. Start RADIUS Server and Proxy Server
```sh
systemctl restart freeradius
```
View logs:
```sh
journalctl -u freeradius -f
```
✅ **Ensure services are running normally without errors**

---

## 🛠 4. Conducting EAP-TLS Authentication Test

### 📌 4.1 Configure EAP-TLS Client
Create `eapol_test.conf`:
```ini
network={
    ssid="GL-AXT1800-83b-5G"
    key_mgmt=WPA-EAP
    eap=TLS
    identity="testuser"
    anonymous_identity="testuser"
    ca_cert="/home/user/ca.crt"
    client_cert="/home/user/client.crt"
    private_key="/home/user/client.key"
    private_key_passwd=""
}
```

### 📌 4.2 Run EAP-TLS Authentication
```sh
eapol_test -c ~/eapol_test.conf -s shared_secret_key -a 192.168.8.182 -p 1812
```
✅ **Proxy Server forwards the request to RADIUS Server, completing authentication**


## ✅ **5. Issues and Solutions**

### 🔴 **1. Proxy Server Did Not Forward Requests Correctly**
#### ❌ **Issue**
- The Proxy Server **did not forward the EAP-TLS authentication request** but **handled authentication locally** instead.
- No **authentication request forwarding** was found in the `radiusd -X` logs.

#### ✅ **Solution**
1. **Enable proxy forwarding**
   ```ini
   proxy_requests = yes
   ```
2. **Ensure `realm` configuration in `proxy.conf` is correct**
   ```ini
   realm DEFAULT {
       type = radius
       authhost = 192.168.8.195:1812
       accthost = 192.168.8.195:1813
       secret = shared_secret_key
       auth_pool = my_auth_failover  # Replace with your own configuration
   }
   ```
3. **Restart FreeRADIUS**
   ```sh
   systemctl restart freeradius
   ```
✅ **Successfully forwarded EAP-TLS authentication requests to the RADIUS Server!** 🎉

---

### 🔴 **2. Certificate Chain Error**
#### ❌ **Issue**
- `eapol_test` authentication failed, with logs showing:
  ```
  Certificate chain - 1 cert(s) untrusted
  ```
- RADIUS Server logs showed:
  ```
  TLS Alert read: warning: unknown CA
  ```

#### ✅ **Solution**
1. **Ensure `eap.conf` on the RADIUS Server is configured correctly**
   ```ini
   tls-config tls-common {
       private_key_file = /etc/freeradius/3.0/certs/server.key
       certificate_file = /etc/freeradius/3.0/certs/server.pem
       ca_file = /etc/freeradius/3.0/certs/ca.pem
   }
   ```
2. **Import the CA certificate on the client**
   ```sh
   scp root@192.168.8.195:/etc/freeradius/3.0/certs/ca.crt ~/Desktop/
   ```
✅ **Certificates successfully matched, authentication passed!** 🎉



## 📌 6. Future Optimization Directions
- **🌍 Multi-level Proxy Authentication Testing**: Attempt secondary forwarding of authentication requests
- **📊 Authentication Delay Optimization**: Measure additional latency introduced by Proxy Server
- **🔐 Certificate Management Optimization**: Simplify client certificate import process to ensure automatic trust of RADIUS CA  

---

## 📌 References
- 📖 **FreeRADIUS Official Documentation**: [https://freeradius.org/](https://freeradius.org/)
- 📖 **Eduroam Authentication Guide**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)

