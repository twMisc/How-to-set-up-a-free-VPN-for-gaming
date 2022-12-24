# 如何建立自己的免費遊戲VPN

我們將會：

1. 利用Oracle Cloud建立免費的VPS伺服器
2. 在VPS上建立WireGuard VPN伺服器
4. 在Windows上利用WireSock連結VPN伺服器並達到分割通道的功能

--- 

## 利用Oracle Cloud建立免費的VPS伺服器

### 註冊帳號

1. 進入網頁 https://www.oracle.com/tw/cloud/free/ 點選「開始免費試用」。依照指示完成註冊帳號。
![](https://i.imgur.com/AzNhnUq.png)

* 依照遊戲需求，選擇主區域。若沒有特別需求就選日本東京。
* 你會需要一張信用卡
* 如果有困難，可以參考 https://izo.tw/oracle-cloud-vps-apply/
* ~~如果即使填了真實資料還是怎麼樣都註冊不過，可以找朋友幫忙註冊（認真）~~

### 建立VPS伺服器

1. 註冊完進入 https://cloud.oracle.com/ 登入並準備建立自己的VPS伺服器。
2. 點選「建立VM執行處理」
![](https://i.imgur.com/2wbcSzE.png)
3. 在這裡，我們將映像檔變更為Ubuntu（因筆者比較熟Ubuntu）
    A. 點選圖片中的編輯
    ![](https://i.imgur.com/bP1nlER.png)
    B. 選擇變更映像檔
    ![](https://i.imgur.com/2PNEoeR.png)
    C. 選擇Canonical Ubuntu並按選取映像檔
    ![](https://i.imgur.com/hVd1QO2.png)
4. 點選「儲存私密金鑰」下載SSH金鑰（請務必下載！）
![](https://i.imgur.com/wiyQAOV.png)
5. 按「建立」。這樣我們就建好VPS了！
6. 在建立完成的頁面可以看到「公用 IP 位址: xxx.xxx.xxx.xx」。請記下這個ip位置。
![](https://i.imgur.com/9PC7d5G.png)

---

## 在VPS上建立WireGuard VPN伺服器

我們首先要利用SSH連上伺服器終端，請先閱讀[{利用SSH和私密金鑰連結伺服器}](#利用SSH和私密金鑰連結伺服器)。再來我們可以在伺服器本機配置WireGuard或是利用Docker配置WireGuard。如果怕麻煩就使用Docker，詳見[{使用Docker建立WireGuard伺服器}](#使用Docker建立WireGuard伺服器)。如果想要自己動手操作，就參照[{建立WireGuard VPN伺服器}](#建立WireGuard-VPN伺服器)。接著不論你使用Docker或是本機配置，都要[開啟Oracle伺服器對外port](#開啟Oracle伺服器對外port)。

### 利用SSH和私密金鑰連結伺服器

0. 我們將使用Windows系統和[MobaXterm](https://mobaxterm.mobatek.net/download.html)說明如何連線。（當然，你可以使用別的軟體，參閱 https://izo.tw/ssh-putty/ ）
1. 將MobaXterm更新到最新版本
2. 在MobaXterm介面左上點選Session並選擇SSH
![](https://i.imgur.com/J4tIGvU.png)
3. 在Remote host輸入剛剛記下的ip位置。打勾Specify username並輸入ubuntu。
4. 點選Advanced SSH settings，打勾Use private key並點選右側按鈕選擇剛剛下載的私密金鑰檔案。
![](https://i.imgur.com/1BFKpuK.png)
5. 點選OK
6. 這時候如果你沒輸入錯誤應該會看到成功進入終端介面。
![](https://i.imgur.com/pKLkHTY.png)

### 建立WireGuard VPN伺服器

0. （本節部分取自 https://www.cyberciti.biz/faq/ubuntu-20-04-set-up-wireguard-vpn-server/ ）
2. 安裝WireGuard。在終端輸入：
```bash!
sudo apt update
sudo apt install wireguard -y
```
2. 生成WireGuard用戶端用的privatekey和publickey。在終端輸入：
```bash!
wg genkey | tee privatekey | wg pubkey > publickey
cat privatekey
cat publickey
```
![](https://i.imgur.com/wsKyuaB.png)
* 記下這兩組key！
3. 生成WireGuard伺服器用的privatekey和publickey。在終端輸入：
```bash!
sudo -i
cd /etc/wireguard/
umask 077; wg genkey | tee privatekey | wg pubkey > publickey
cat privatekey
cat publickey
```
![](https://i.imgur.com/D3qpp99.png)
* 記下這兩組key！
4. 記下網路裝置的名字：
```
ip link show
```
![](https://i.imgur.com/d3OEEdL.png)
* 圖中紅色底線為網路裝置的名字，如果你的系統版本跟我一樣，那應該是寫`ens3`
5. 輸入`nano /etc/wireguard/wg0.conf`編輯伺服器設定檔如下。請注意將「伺服器privatekey」和「用戶端publickey」換成剛剛分別在3.和2.記下的內容。
```
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = 伺服器privatekey
SaveConfig = false
PostUp = /etc/wireguard/helper/add-nat-routing.sh
PostDown = /etc/wireguard/helper/remove-nat-routing.sh

[Peer]
PublicKey = 用戶端publickey
AllowedIPs = 10.8.0.2/32
```
6. 編輯完成後依序按`Ctrl+x`，`y`，`Enter`儲存並離開編輯器。
7. 建立用來處理防火牆規則的script。建立資料夾，並利用`nano`編輯檔案。
```bash!
mkdir -v /etc/wireguard/helper/
nano /etc/wireguard/helper/add-nat-routing.sh
```
8. 複製貼上以下內容並如第六步儲存離開。（如果你的網路裝置名字不一樣，把`ens3`換成它的名字）
```
#!/bin/bash
IPT="/sbin/iptables"
IPT6="/sbin/ip6tables"

IN_FACE="ens3"                   # NIC connected to the internet
WG_FACE="wg0"                    # WG NIC
SUB_NET="10.8.0.0/24"            # WG IPv4 sub/net aka CIDR
WG_PORT="51820"                  # WG udp port
SUB_NET_6="fd42:42:42:42::/112"  # WG IPv6 sub/net

## IPv4 ##
$IPT -t nat -I POSTROUTING 1 -s $SUB_NET -o $IN_FACE -j MASQUERADE
$IPT -I INPUT 1 -i $WG_FACE -j ACCEPT
$IPT -I FORWARD 1 -i $IN_FACE -o $WG_FACE -j ACCEPT
$IPT -I FORWARD 1 -i $WG_FACE -o $IN_FACE -j ACCEPT
$IPT -I INPUT 1 -i $IN_FACE -p udp --dport $WG_PORT -j ACCEPT
```
9. 建立另一個檔案`remove-nat-routing.sh`，在檔案裡貼上以下內容並儲存離開。
* 終端輸入：
```bash!
nano /etc/wireguard/helper/remove-nat-routing.sh
```
* `remove-nat-routing.sh`內容：
```
#!/bin/bash
IPT="/sbin/iptables"
IPT6="/sbin/ip6tables"

IN_FACE="ens3"                   # NIC connected to the internet
WG_FACE="wg0"                    # WG NIC
SUB_NET="10.8.0.0/24"            # WG IPv4 sub/net aka CIDR
WG_PORT="51820"                  # WG udp port
SUB_NET_6="fd42:42:42:42::/112"  # WG IPv6 sub/net

# IPv4 rules #
$IPT -t nat -D POSTROUTING -s $SUB_NET -o $IN_FACE -j MASQUERADE
$IPT -D INPUT -i $WG_FACE -j ACCEPT
$IPT -D FORWARD -i $IN_FACE -o $WG_FACE -j ACCEPT
$IPT -D FORWARD -i $WG_FACE -o $IN_FACE -j ACCEPT
$IPT -D INPUT -i $IN_FACE -p udp --dport $WG_PORT -j ACCEPT
```
10. 建立系統ip forward規則。輸入`nano /etc/sysctl.d/10-wireguard.conf`並編輯如下：
```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```
11. 輸入以下終端指令開始wiregaurd服務。
```bash!
sysctl -p /etc/sysctl.d/10-wireguard.conf
chmod -v +x /etc/wireguard/helper/*.sh
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service
```
12. （終於）完成！這時候可以斷開和伺服器的連結！

### 使用Docker建立WireGuard伺服器

1. 安裝Docker：
（參照 https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository ）
```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
2. 檢查Docker是否正常執行：
```
service docker status
```

![](https://i.imgur.com/5kMwKR9.png)
* 看到`Active: active (running)`表示沒問題，按下`Ctrl+C`跳出
3. 將自己使用者帳號加入docker群組：
```
sudo usermod -aG docker ubuntu
newgrp docker
```
4. 收集必要資訊：
```
id ubuntu
```
![](https://i.imgur.com/C4FilRP.png)
* 記下uid以及gid
5. 利用Docker部署WireGuard伺服器：
```
docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1001 \
  -e PGID=1001 \
  -e TZ=Asia/Taipei \
  -e SERVERURL=伺服器ip位置 \
  -e PEERS=1 \
  -e PEERDNS=8.8.8.8,8.8.4.4 \
  -e INTERNAL_SUBNET=10.8.0.0/24 \
  -e ALLOWEDIPS=0.0.0.0/0 \
  -e LOG_CONFS=true \
  -p 51820:51820/udp \
  -v /home/ubuntu/wireguard/config:/config \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --restart unless-stopped \
  lscr.io/linuxserver/wireguard:latest
```
* `PUID`為上一步中的`uid`，`PGID`為上一步中的`gid`
* 將`伺服器ip位置`換成剛剛記下的ip
6. 將客戶端所需資訊提出：
```
cat /home/ubuntu/wireguard/config/peer1/peer1.conf
```
![](https://i.imgur.com/0TRPSxq.png)
7. 在客戶端（自己的電腦）新增一個檔案`wg0.conf`，將上一步回傳的內容複製進去，保存。

### 開啟Oracle伺服器對外port

1. 到 https://cloud.oracle.com/networking/vcns ，點選`vcn-xxxxx-`－> `subnet-xxx-x`－> `Default security...`
![](https://i.imgur.com/AtCfPhJ.png)
![](https://i.imgur.com/Kkk4aLh.png)
![](https://i.imgur.com/poJpAQP.png)
2. 點選`新增傳入規則`，並如圖片輸入：
![](https://i.imgur.com/WSWhTRH.png)
3. 點選`新增傳入規則`儲存規則

---

## 在Windows上利用WireSock連結VPN伺服器並達到分割通道的功能

本節介紹如何使用WireSock或是它的一個GUI版本：TunnlTo來連線WireGuard伺服器。相較於一般的WireGuard用戶端，透過WireSock能夠以程序名稱來實現分割通道，這樣就可以很簡單的應用在遊戲上。

### 利用WireSock連線VPN

0. 我們將以Firefox為例，實現只將Firefox通過VPN。
1. 新增一個檔案`wg0.conf`，並編輯為以下內容（請將`用戶端privatekey`、`伺服器publickey`、`伺服器ip位置`變更為剛剛記下的內容）：
```
[Interface]
PrivateKey = 用戶端privatekey
Address = 10.8.0.2/24
DNS = 8.8.8.8, 8.8.4.4

[Peer]
PublicKey = 伺服器publickey
AllowedIPs = 0.0.0.0/0
Endpoint = 伺服器ip位置:51820
PersistentKeepalive = 15

AllowedApps = firefox.exe
```
* 如果使用Docker建立WireGuard伺服器則可以直接編輯先前保存好的檔案
2. 下載並安裝[WireSock](https://www.wiresock.net/downloads/wiresock-vpn-client-x64-1.2.15.1.msi)。
3. 打開powershell或命令提示字元，執行WireSock。注意必須把`YOUR_PATH`換成剛剛建立的`wg0.conf`的路徑！
```powershell!
wiresock-client.exe run -config "C:\YOUR_PATH\wg0.conf" -log-level info
```
5. 可以打開Firefox和Edge驗證ip是否改變：
![](https://i.imgur.com/2MNJgaz.png)
* （成功了！太好了！）

### 利用TunnlTo連線VPN

如果不喜歡指令介面，可以考慮使用TunnlTo的圖形介面。（這個軟體還在開發中，可能會遇到一些bug，本文撰寫時為v0.1.4版）

1. 到 https://github.com/TunnlTo/desktop-app/releases/latest/ 下載並安裝最新的TunnlTo版本。
2. 打開TunnlTo並按`Add Tunnel`，填入需要的內容，並按`Save`儲存：
![](https://i.imgur.com/OEwW9ZL.png)
* 請將`用戶端privatekey`、`伺服器publickey`、`伺服器ip位置`變更為剛剛記下的內容！
* 可以將Allowed Apps寫入想要連線vpn的軟體，在這裡我們以firefox為例
3. 點選`Enable`並打開Firefox測試看看是否連線成功！

### 其他WireSock使用方法 

#### 使用範例：只將遊戲Final Fantasy XIV的連線通過VPN

* 修改AllowedApps指定為Final Fantasy XIV的程序
`AllowedApps = ffxivlauncher.exe, ffxiv_dx11.exe`

#### 其他WireSock功能
WireSock 有一些一般的Wireguard VPN client沒有的功能。可以編輯.conf檔或是在TunnlTo裡面輸入來設定。以下翻譯 https://www.wiresock.net/ 的內容：
* AllowedApps - 指定用逗號分隔的應用程式名稱（或部分名稱）清單，透過 VPN 隧道傳送。這個參數縮小了 AllowedIPs，所以要隧道化的流量應該同時符合 AllowedIPs 和 AllowedApps。例如，'AllowedApps = chrome' 和 'AllowedIPs = 0.0.0.0/0' 會將 Chrome 瀏覽器傳送到 VPN 連接，其他所有東西都會繞過隧道。
* DisallowedApps - 指定用逗號分隔的應用程式名稱（或部分名稱）清單，排除在隧道外。這個參數是 AllowedApps 的相反。請注意，AllowedApps 優先，如果指定了兩者，則會先使用 AllowedApps。
* DisallowedIPs - 指定用逗號分隔的 IP 子網清單，排除在隧道外。例如，AllowedIps = 0.0.0.0/0 和 DisallowedIPs = 192.168.0.1/24 會排除在隧道中的 192.168.0.1/24。
* WireGuard 透過 SOCKS5 代理連線的參數：
    * Socks5Proxy - 指定 SOCKS5 代理端點，例如 Socks5Proxy = socks5.sshvpn.me:1080 或 Socks5Proxy = 13.134.12.31:1080
    * Socks5ProxyUsername - 指定 SOCKS5 用戶名（可選）
    * Socks5ProxyPassword - 指定 SOCKS5 密碼（可選）

---

## FAQ

### 我無法用SSH連線到VPS？

請確認你的MobaXterm是最新版本。某些舊版本的OpenSSH加密有些問題。

### 我的Oracle VPS 無法成功`sudo apt update`？

請確認你的Oracle雲端傳出規則（到 https://cloud.oracle.com/networking/vcns ，點選`vcn-xxxxx-`－> `subnet-xxx-x`－> `Default security...`－＞傳出規則）有設定規則，如圖中設定。
![](https://i.imgur.com/jm6hJj8.png)

### 我連上VPN後但沒有網路？

這可能有很多原因。如：DNS設定錯誤、金鑰輸入錯誤、伺服器位置錯誤等等。請確認每個步驟的檔案都有編輯正確。使用TunnlTo也許會遇到不可預期的問題，如果遇到困難建議使用WireSock指令界面。

---
## 雜記

* 2022.12.21 初版編寫完成。
* 2022.12.22 @jacky50403 新增了使用Docker配置的方法。
* 感謝[WireSock](https://www.wiresock.net/)的作者製作強大的軟體
* 感謝[TunnlTo](https://github.com/TunnlTo/desktop-app)的作者幫忙解決繁體中文的bug