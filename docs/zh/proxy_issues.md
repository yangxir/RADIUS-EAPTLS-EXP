# 📡 RADIUS Proxy Server 认证指南

## 📌 1. 代理服务器的作用
### 🔹 为什么需要 Proxy Server？
在 RADIUS 认证架构中，**Proxy Server（代理服务器）** 主要用于 **转发** 认证请求，而非直接处理认证。其核心作用是在 **分布式环境** 下，确保客户端的认证请求能够顺利到达最终的 RADIUS 服务器。

在以下场景中，Proxy Server **至关重要**：
- **企业 WiFi 认证**（如 Eduroam 等多站点漫游）
- **多级 RADIUS 认证**（如 ISP 认证系统）
- **跨国/跨区域 WiFi 认证**（不同国家 RADIUS 服务器间的认证转发）

---

## 📌 2. Proxy Server 的工作流程
1️⃣ **客户端** 发送 **EAP-TLS** 认证请求  
2️⃣ **Proxy Server** 接收请求，并**转发**至最终的 RADIUS Server  
3️⃣ **RADIUS Server** 处理认证请求，并返回认证结果  
4️⃣ **Proxy Server** 将结果返回给客户端  
5️⃣ **客户端成功接入 WiFi** 🎉  

---

## ⚙️ 3. 代理服务器配置

### 📍 3.1 设备环境
| 设备 | 角色 | IP 地址 |
|------|------|---------|
| **MiniPC-1** | 认证客户端 (EAP-TLS) | `192.168.8.220` |
| **Router (GL-AXT1800-83b-5G)** | 连接所有设备，Proxy 作为 RADIUS 服务器 | N/A |
| **MiniPC-2** | Proxy Server (转发 RADIUS 认证请求) | `192.168.8.182` |
| **PC (RADIUS Server)** | 认证服务器 (FreeRADIUS 3.0.26) | `192.168.8.195` |

---

### 📍 3.2 确保 Proxy Server 能连接到 RADIUS Server

**测试 UDP 1812 端口连通性**
```sh
nc -zvu 192.168.8.195 1812
```
✅ **UDP 端口 1812 可达**，表示 Proxy Server 可以与 RADIUS Server 通信。

---

### 📍 3.3 配置 Proxy Server
#### 🔹 1. 修改 `proxy.conf`
```ini
proxy_requests = yes

realm DEFAULT {
    type = radius
    authhost = 192.168.8.195:1812
    accthost = 192.168.8.195:1813
    secret = shared_secret_key
}
```
✅ **确保 Proxy Server 仅作为 "中继"，而非 "本地认证" 服务器**

---

#### 🔹 2. 配置 RADIUS Server，允许 Proxy
在 `clients.conf` 添加 Proxy Server：
```ini
client 192.168.8.182 {
    secret = shared_secret_key
    shortname = proxy
}
```
✅ **确保 Proxy Server 和 RADIUS Server 的 `secret` 一致，否则会报错**

---

#### 🔹 3. 启动 RADIUS Server 和 Proxy Server
```sh
systemctl restart freeradius
```
查看日志：
```sh
journalctl -u freeradius -f
```
✅ **确保服务正常运行，无错误信息**

---

## 🛠 4. 进行 EAP-TLS 认证测试

### 📌 4.1 配置 EAP-TLS 客户端
创建 `eapol_test.conf`：
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

### 📌 4.2 运行 EAP-TLS 认证
```sh
eapol_test -c ~/eapol_test.conf -s shared_secret_key -a 192.168.8.182 -p 1812
```
✅ **Proxy Server 将请求转发到 RADIUS Server，最终完成认证**

---

## ✅ 5. 遇到的问题与解决方案

### 🔴 1. Proxy Server 没有正确转发请求
#### ❌ 现象
- Proxy Server **未正确转发 EAP-TLS 认证请求**，而是**本地处理**认证。
- `radiusd -X` 日志中无 **认证请求发送** 记录。

#### ✅ 解决方案
1. **启用代理转发**
   ```ini
   proxy_requests = yes
   ```
2. **确保 `proxy.conf` 里 `realm` 配置正确**
   ```ini
   realm DEFAULT {
       type = radius
       authhost = 192.168.8.195:1812
       accthost = 192.168.8.195:1813
       secret = shared_secret_key
       auth_pool = my_auth_failover  #换成自己的
   }
   ```
3. **重启 FreeRADIUS**
   ```sh
   systemctl restart freeradius
   ```
✅ **成功转发 EAP-TLS 认证请求至 RADIUS Server！** 🎉

---

### 🔴 2. 证书链错误
#### ❌ 现象
- `eapol_test` 认证失败，日志显示：
  ```
  Certificate chain - 1 cert(s) untrusted
  ```
- RADIUS Server 端日志：
  ```
  TLS Alert read: warning: unknown CA
  ```

#### ✅ 解决方案
1. **确保 RADIUS Server `eap.conf` 配置正确**
   ```ini
   tls-config tls-common {
       private_key_file = /etc/freeradius/3.0/certs/server.key
       certificate_file = /etc/freeradius/3.0/certs/server.pem
       ca_file = /etc/freeradius/3.0/certs/ca.pem
   }
   ```
2. **客户端导入 CA 证书**
   ```sh
   scp root@192.168.8.195:/etc/freeradius/3.0/certs/ca.crt ~/Desktop/
   ```
✅ **证书匹配成功，认证通过！** 🎉

---

## 📌 6. 未来优化方向
- **🌍 多级 Proxy 认证测试**：尝试 Proxy Server **二次转发**认证请求
- **📊 认证延迟优化**：测量 Proxy Server 转发的 **额外延迟**
- **🔐 证书管理优化**：简化客户端证书导入流程，确保设备能够自动信任 RADIUS CA  

---

## 📌 参考资料
- 📖 **FreeRADIUS 官方文档**: [https://freeradius.org/](https://freeradius.org/)
- 📖 **Eduroam 认证指南**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)

