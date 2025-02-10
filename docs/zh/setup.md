# EAP-TLS RADIUS Server on Ubuntu

本项目提供 Ubuntu 环境下 **EAP-TLS 认证的 FreeRADIUS 服务器** 搭建教程，包括：
- **FreeRADIUS 安装**
- **EAP-TLS 证书生成**
- **RADIUS 服务器配置**
- **终端设备连接**
- **调试与排错**

---

## 🚀 实验环境
1. **Ubuntu 虚拟机**（作为 RADIUS 服务器）。
2. **路由器**（作为无线接入点）。
3. **红米 K50 手机**（作为无线客户端）。
4. **无线网卡**（用于调试或抓包）。

---

---

## 📌 1. 安装 FreeRADIUS
```sh
sudo apt update && sudo apt install freeradius freeradius-utils -y
```

检查 RADIUS 服务器是否正常运行：
```sh
sudo systemctl status freeradius
```

如果未启动：
```sh
sudo systemctl enable --now freeradius
```

---

## 📌 2. 配置 FreeRADIUS

### **修改 `clients.conf`**
```sh
sudo nano /etc/freeradius/3.0/clients.conf
```
添加以下内容（⚠️ **请替换 `192.168.51.2` 为你的路由器 IP** ⚠️）：
```ini
client your_router {
    ipaddr = 192.168.51.2  
    secret = testing123  
    shortname = router  
}
```

### **启用 EAP 模块**
```sh
sudo nano /etc/freeradius/3.0/mods-enabled/eap
```
确保以下内容正确：
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
⚠️ **因为我的证书都放在这，你需要换成你自己证书的位置！**


---

## 📌 3. 生成证书
创建证书目录（如果不存在）：
```sh
sudo mkdir -p /etc/freeradius/3.0/certs
cd /etc/freeradius/3.0/certs
```

### **生成 CA 证书**
```sh
sudo openssl genrsa -out ca.key 2048
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=CA"
```

### **生成服务器证书**
```sh
sudo openssl genrsa -out server.key 2048
sudo openssl req -new -key server.key -out server.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=radius-server.local"
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
```

### **生成客户端证书**
```sh
sudo openssl genrsa -out client.key 2048
sudo openssl req -new -key client.key -out client.csr -subj "/C=JP/ST=Miyagi/L=Sendai/O=Tohoku University/OU=Goto Lab/CN=client"
sudo openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```


⚠️ ***注意：
上面的命令中，"Tohoku University"、"Goto Lab" 等内容是我研究室的相关信息，你需要替换成你自己的单位或组织名称！***
### **创建 PKCS#12 客户端证书（用于手机导入）**
```sh
sudo openssl pkcs12 -export -inkey client.key -in client.crt -certfile ca.crt -out client.p12
```

---

## 📌 4. 启动 FreeRADIUS 并测试
```sh
sudo systemctl restart freeradius
sudo journalctl -u freeradius -f
```

使用 `radtest` 进行认证测试：
```sh
radtest user password 127.0.0.1 0 testing123
```
如果成功，应该返回 `Access-Accept`。

---

## 📌 5. 连接无线客户端

### **准备客户端证书**
1. 复制 `client.p12` 和 `ca.crt` 到手机（可以使用 Google Drive）。
2. 在 Android / iPhone 上导入证书：
   - 进入 **Wi-Fi 高级设置**，选择 **EAP-TLS** 认证。
   - 选择 `client.p12` 作为客户端证书。
   - 选择 `ca.crt` 作为 CA 证书。

---

## 📌 6. 配置路由器
1. 进入路由器管理界面。
2. 在 **无线安全** 或 **高级设置** 中，启用 **WPA2-Enterprise**（EAP-TLS）。
3. 配置 RADIUS 服务器地址：
   - **RADIUS Server IP**: `你的 RADIUS 服务器 IP`
   - **Port**: `1812`
   - **Secret**: `testing123` （此密码应与 `clients.conf` 文件中的 `secret` 字段一致）

📌 **确保 `clients.conf` 文件中的 RADIUS 共享密钥一致**
```ini
client your_router {
    ipaddr = 192.168.51.2  # 替换为你的路由器 IP
    secret = testing123    # RADIUS 共享密钥
    shortname = router     # 简称，便于识别
}
```
⚠️ ***注意：如果 clients.conf 中的 secret 与路由器配置的不匹配，RADIUS 认证将失败！***

---

## 🔧 7. 常见问题与解决方案
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

---

## 📜 License
本项目基于 **MIT License**，欢迎自由使用和修改。如需引用或二次开发，请注明来源。

---

## 📚 参考资料
本项目的内容基于以下资源：
- **FreeRADIUS 官方网站**: [https://www.freeradius.org/](https://www.freeradius.org/)
- **NII 学术认证推进室 - Eduroam 与 RADIUS 相关文档**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)

---

## 📩 联系方式
如果你有任何问题或建议，可以提交 Issue 反馈，或者访问我的 GitHub 主页：

🔗 [GitHub 主页](https://github.com/yangxir)  

如果觉得本项目有帮助，欢迎 **Star ⭐ 支持**！

