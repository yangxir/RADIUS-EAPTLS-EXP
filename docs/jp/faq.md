
## 🔧 よくある質問と解決策

### **EAP-TLS 認証の失敗**
```sh
TLS Alert: unknown CA
```
**解決方法**：
- クライアントが `ca.crt` を正しくインポートしていることを確認する。
- `openssl verify -CAfile ca.crt server.crt` を使用して証明書を検証する。

### **FreeRADIUS が起動しない**
```sh
Failed to start freeradius.service
```
**解決方法**：
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

### **証明書の有効期限切れ**
証明書が無効になっている場合、以下の手順で再生成してください：
```sh
sudo rm /etc/freeradius/3.0/certs/*.crt
sudo rm /etc/freeradius/3.0/certs/*.key
sudo rm /etc/freeradius/3.0/certs/*.csr
sudo rm /etc/freeradius/3.0/certs/*.srl
bash scripts/generate_certs.sh

```