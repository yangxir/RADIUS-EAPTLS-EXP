
## 🔧 Common Issues and Solutions

### **EAP-TLS Authentication Failure**
```sh
TLS Alert: unknown CA
```
**Solution**：
- Ensure the client has imported `ca.crt` correctly.
- Use `openssl verify -CAfile ca.crt server.crt` to verify the certificate.

### **FreeRADIUS Fails to Start**
```sh
Failed to start freeradius.service
```
**Solution**：
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

### **Certificate Expired**
If the certificate is invalid, clean up and regenerate:
```sh
sudo rm /etc/freeradius/3.0/certs/*.crt
sudo rm /etc/freeradius/3.0/certs/*.key
sudo rm /etc/freeradius/3.0/certs/*.csr
sudo rm /etc/freeradius/3.0/certs/*.srl
bash scripts/generate_certs.sh
```
