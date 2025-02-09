## 🔧 常见问题与解决方案
### **EAP-TLS 认证失败**
```sh
TLS Alert: unknown CA
```
**解决方法**：
- 确保客户端已导入 `ca.crt`。
- 使用 `openssl verify -CAfile ca.crt server.crt` 进行验证。

### **FreeRADIUS 无法启动**
```sh
Failed to start freeradius.service
```
**解决方法**：
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

### **证书过期**
如果证书无效，清理后重新生成：
```sh
sudo rm /etc/freeradius/3.0/certs/*.crt
sudo rm /etc/freeradius/3.0/certs/*.key
sudo rm /etc/freeradius/3.0/certs/*.csr
sudo rm /etc/freeradius/3.0/certs/*.srl
bash scripts/generate_certs.sh
```
