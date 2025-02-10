# EAP-TLS RADIUS Server on Ubuntu

This project provides a guide to setting up an **EAP-TLS authentication FreeRADIUS server** on Ubuntu, including:
- **Installing FreeRADIUS**
- **Generating EAP-TLS Certificates**
- **Configuring RADIUS Server**
- **Connecting Client Devices**
- **Debugging and Troubleshooting**

---

## 🚀 Experiment Environment
1. **Ubuntu Virtual Machine** (as the RADIUS server).
2. **Router** (as the wireless access point).
3. **Redmi K50 Smartphone** (as the wireless client).
4. **Wireless Network Adapter** (for debugging or packet capture).

---



---

## 📌 1. Installing FreeRADIUS
```sh
sudo apt update && sudo apt install freeradius freeradius-utils -y
```

Check if the RADIUS server is running:
```sh
sudo systemctl status freeradius
```

If not started:
```sh
sudo systemctl enable --now freeradius
```

---

## 📌 2. Configuring FreeRADIUS

### **Modify `clients.conf`**
```sh
sudo nano /etc/freeradius/3.0/clients.conf
```
Add the following entry (**⚠️ Replace `192.168.51.2` with your router's IP ⚠️**):
```ini
client your_router {
    ipaddr = 192.168.51.2  
    secret = testing123  
    shortname = router  
}
```

### **Enable EAP Module**
```sh
sudo nano /etc/freeradius/3.0/mods-enabled/eap
```
Ensure the following configuration:
```ini
eap {
    default_eap_type = tls
}

tls-config tls-common {
    private_key_file = /etc/freeradius/3.0/certs/server.key
    certificate_file = /etc/freeradius/3.0/certs/server.crt
    CA_file = /etc/freeradius/3.0/certs/ca.crt
    dh_file = /etc/freeradius/3.0/certs/dh
    random_file = /dev/urandom
}
```

⚠️ **Since my certificates are stored here, you need to replace them with your own certificate paths!**

---

## 📌 3. Generating Certificates
Create the certificate directory (if it does not exist):
```sh
sudo mkdir -p /etc/freeradius/3.0/certs
cd /etc/freeradius/3.0/certs
```

### **Generate CA Certificate**
```sh
sudo openssl genrsa -out ca.key 2048
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=CA"
```

### **Generate Server Certificate**
```sh
sudo openssl genrsa -out server.key 2048
sudo openssl req -new -key server.key -out server.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=radius-server.local"
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
```

### **Generate Client Certificate**
```sh
sudo openssl genrsa -out client.key 2048
sudo openssl req -new -key client.key -out client.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=client"
sudo openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```

⚠️ ***Note:
The above commands include information such as "Tohoku University" and "Goto Lab," which are specific to my research lab. Please replace them with your own organization details!***

### **Create PKCS#12 Client Certificate (For Mobile Import)**
```sh
sudo openssl pkcs12 -export -inkey client.key -in client.crt -certfile ca.crt -out client.p12
```

---

## 📌 4. Start FreeRADIUS and Test
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

Test authentication using `radtest`:
```sh
radtest user password 127.0.0.1 0 testing123
```
If successful, it should return `Access-Accept`.

---

## 📌 5. Connect Wireless Clients

### **Prepare Client Certificates**
1. Copy `client.p12` and `ca.crt` to your smartphone (e.g., using Google Drive).
2. Import the certificates on Android/iPhone:
   - Go to **Wi-Fi Advanced Settings**, select **EAP-TLS** authentication.
   - Choose `client.p12` as the client certificate.
   - Choose `ca.crt` as the CA certificate.

---

## 📌 6. Configure Router
1. Open your router management interface.
2. In **Wireless Security** or **Advanced Settings**, enable **WPA2-Enterprise (EAP-TLS)**.
3. Set up the RADIUS server address:
   - **RADIUS Server IP**: `Your RADIUS Server IP`
   - **Port**: `1812`
   - **Secret**: `testing123` (This must match the `secret` value in `clients.conf`)

📌 **Ensure the RADIUS shared key in `clients.conf` matches your router configuration**
```ini
client your_router {
    ipaddr = 192.168.51.2  # Replace with your router IP
    secret = testing123    # RADIUS shared secret
    shortname = router     # Alias for easier identification
}
```

⚠️ ***Note: If the `secret` in `clients.conf` does not match the router configuration, RADIUS authentication will fail!***

---

## 📜 License
This project is licensed under the **MIT License**. You are free to use and modify it. If you reference or further develop this project, please acknowledge the source.

---

## 📚 References
This project is based on the following resources:
- **FreeRADIUS Official Website**: [https://www.freeradius.org/](https://www.freeradius.org/)
- **NII Academic Authentication Promotion Office - Eduroam & RADIUS Documentation**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)

---

## 📩 Contact
If you have any questions or suggestions, please submit an issue or visit my GitHub page:

🔗 [GitHub Profile](https://github.com/yangxir)  

If you find this project useful, please consider giving it a **Star ⭐**!
