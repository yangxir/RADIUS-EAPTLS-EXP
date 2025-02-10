# 📡 RADIUS プロキシサーバー認証ガイド

## 📌 1. プロキシサーバーの役割
### 🔹 なぜプロキシサーバーが必要なのか？
RADIUS 認証アーキテクチャでは、**プロキシサーバー** は **認証要求を転送** するために使用され、直接認証を処理するわけではありません。その主な役割は、**分散環境** においてクライアントの認証要求が最終的な RADIUS サーバーに到達できるようにすることです。

プロキシサーバーは以下のシナリオで **重要** です：
- **企業の WiFi 認証**（例: Eduroam マルチサイト ローミング）
- **多段階の RADIUS 認証**（例: ISP 認証システム）
- **国境を越えた WiFi 認証**（異なる国の RADIUS サーバー間での認証転送）

---

## 📡 2. プロキシサーバーのワークフロー
1️⃣ **クライアント** が **EAP-TLS** 認証要求を送信  
2️⃣ **プロキシサーバー** が要求を受信し、**最終的な RADIUS サーバーに転送**  
3️⃣ **RADIUS サーバー** が認証要求を処理し、結果を返す  
4️⃣ **プロキシサーバー** が結果をクライアントに送信  
5️⃣ **クライアントが WiFi に正常に接続** 🎉  

---

## ⚙️ 3. プロキシサーバーの設定

### 📍 3.1 デバイス環境
| デバイス | 役割 | IP アドレス |
|------|------|---------|
| **MiniPC-1** | 認証クライアント (EAP-TLS) | `192.168.8.220` |
| **Router (GL-AXT1800-83b-5G)** | 全デバイスを接続し、プロキシが RADIUS サーバーとして動作 | N/A |
| **MiniPC-2** | プロキシサーバー (RADIUS 認証要求を転送) | `192.168.8.182` |
| **PC (RADIUS サーバー)** | 認証サーバー (FreeRADIUS 3.0.26) | `192.168.8.195` |

---

### 📍 3.2 プロキシサーバーが RADIUS サーバーと接続できることを確認

**UDP 1812 ポートの接続テスト**
```sh
nc -zvu 192.168.8.195 1812
```
✅ **UDP ポート 1812 への接続成功**、プロキシサーバーが RADIUS サーバーと通信可能。

---

### 📍 3.3 プロキシサーバーの設定
#### 🔹 1. `proxy.conf` の編集
```ini
proxy_requests = yes

realm DEFAULT {
    type = radius
    authhost = 192.168.8.195:1812
    accthost = 192.168.8.195:1813
    secret = shared_secret_key
}
```
✅ **プロキシサーバーが "中継" のみを行い、"ローカル認証" をしないことを確認**

---

#### 🔹 2. RADIUS サーバーの設定（プロキシを許可）
`clients.conf` にプロキシサーバーを追加：
```ini
client 192.168.8.182 {
    secret = shared_secret_key
    shortname = proxy
}
```
✅ **プロキシサーバーと RADIUS サーバーの `secret` が一致することを確認**

---

#### 🔹 3. RADIUS サーバーとプロキシサーバーの起動
```sh
systemctl restart freeradius
```
ログを確認：
```sh
journalctl -u freeradius -f
```
✅ **サービスが正常に動作していることを確認**

---

## 🛠 4. EAP-TLS 認証テストの実行

### 📌 4.1 EAP-TLS クライアントの設定
`eapol_test.conf` を作成：
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

### 📌 4.2 EAP-TLS 認証を実行
```sh
eapol_test -c ~/eapol_test.conf -s shared_secret_key -a 192.168.8.182 -p 1812
```
✅ **プロキシサーバーが要求を RADIUS サーバーに転送し、認証成功**



## ✅ **5. 発生した問題と解決策**

### 🔴 **1. プロキシサーバーがリクエストを正しく転送しない**
#### ❌ **問題**
- Proxy Server が **EAP-TLS 認証リクエストを正しく転送せず、ローカルで処理してしまう**。
- `radiusd -X` ログに **認証リクエストの転送記録がない**。

#### ✅ **解決策**
1. **プロキシ転送を有効にする**
   ```ini
   proxy_requests = yes
   ```
2. **`proxy.conf` の `realm` 設定が正しいことを確認**
   ```ini
   realm DEFAULT {
       type = radius
       authhost = 192.168.8.195:1812
       accthost = 192.168.8.195:1813
       secret = shared_secret_key
       auth_pool = my_auth_failover  # 自分の設定に変更してください
   }
   ```
3. **FreeRADIUS を再起動**
   ```sh
   systemctl restart freeradius
   ```
✅ **EAP-TLS 認証リクエストの転送に成功しました！** 🎉

---

### 🔴 **2. 証明書チェーンのエラー**
#### ❌ **問題**
- `eapol_test` の認証に失敗し、ログに以下のように表示される：
  ```
  Certificate chain - 1 cert(s) untrusted
  ```
- RADIUS サーバーのログ：
  ```
  TLS Alert read: warning: unknown CA
  ```

#### ✅ **解決策**
1. **RADIUS サーバーの `eap.conf` 設定が正しいことを確認**
   ```ini
   tls-config tls-common {
       private_key_file = /etc/freeradius/3.0/certs/server.key
       certificate_file = /etc/freeradius/3.0/certs/server.pem
       ca_file = /etc/freeradius/3.0/certs/ca.pem
   }
   ```
2. **クライアントに CA 証明書をインポート**
   ```sh
   scp root@192.168.8.195:/etc/freeradius/3.0/certs/ca.crt ~/Desktop/
   ```
✅ **証明書が正しく一致し、認証が成功しました！** 🎉
## 📌 6. 今後の最適化の方向性
- **🌍 多段階プロキシ認証テスト**: 認証要求の二次転送を試す
- **📊 認証遅延の最適化**: プロキシサーバーの転送による追加遅延を測定
- **🔐 証明書管理の最適化**: クライアント証明書のインポートプロセスを簡素化し、RADIUS CA の信頼を自動化

---

## 📌 参考資料
- 📖 **FreeRADIUS 公式ドキュメント**: [https://freeradius.org/](https://freeradius.org/)
- 📖 **Eduroam 認証ガイド**: [NII MeatWiki](https://meatwiki.nii.ac.jp/confluence/pages/viewpage.action?pageId=94340973)

