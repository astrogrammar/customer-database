# サーバー仕様概要（Webアプリ構築AI向け）

> **稼働場所**: `charapla.net/app/customer-manager/`

------

## 0. 目的と前提

- 目的：本VPS上で **Webアプリ（PHP/Python/JavaScript）** を安全に稼働させ、既存サービス（Seafile / SFTP運用 / Apache運用ルール）と衝突しない設計・配置・デプロイを行う。
- 前提：OS/ミドルウェアの制約、SELinux、firewalld、既存vhost/DNS方針、運用監査（httpd-audit）に従う。

------

## 1. インフラ基盤（OS / セキュリティ）

- OS：Rocky Linux 9
- SELinux：Enforcing
- firewall：22 / 80 / 443 を許可（他ポートは原則閉）
- Fail2ban：sshd jail 有効（SSH防御）
- SSH鍵：公開鍵は `authorized_keys` をユーザー別に `/etc/ssh/authorized_keys/%u` に置く設計

------

## 2. ドメイン運用方針（DNS・役割分担）

### 2.1 役割

- `charapla.net`：VPS（ただし **トップのみ** `www.charapla.net` へ 301 リダイレクト）
- `www.charapla.net`：さくら共有（WordPress想定）
- `files.charapla.net`：VPS（Seafile）
- 将来：アプリは `charapla.net/app/` または `app.charapla.net`（VPS）

### 2.2 リダイレクト仕様（charapla.net）

- ルート `/` のみ `https://www.charapla.net/` に 301
- `/dir/` 等の下層はリダイレクトしない（裸ドメイン側で通常応答）

------

## 3. Webサーバ（Apache）運用ルール（重要）

### 3.1 Listen定義のゴールデンルール

- `Listen 80`：`/etc/httpd/conf/httpd.conf` に **1行だけ**
- `Listen 443`：`/etc/httpd/conf.d/ssl.conf` に **1行だけ**
- vhostファイル内に Listen は書かない
- confの無効化ファイルは **`.conf` で終わらせない**（conf.d の読込対象に入って事故るため）

### 3.2 反映手順

- 標準：`apachectl -t` で構文OK → `systemctl reload httpd`
- `restart` は潜在エラーを露見させやすいので計画的に

### 3.3 監査（httpd-audit）

- Apache設定/稼働状態を監査する仕組みがあり、設定変更時の健全性を担保する（Listen重複、vhostダンプ、証明書期限、疎通など）
- ログ出力先：`/var/log/httpd/audit`

------

## 4. 主要サービス構成

### 4.1 Seafile（files.charapla.net）

- Apache（SSL + リバースプロキシ）
- Seafile v11.0.13
  - fileserver：`:8082`（systemd: `seafile.service`）
  - web UI：`127.0.0.1:8000`（systemd: `seahub.service`）
- DB：MariaDB
- TLS：Let's Encrypt（certbot timer運用）

### 4.2 Webアプリ（PHP / Python / Node/JS）想定

- 外部公開は基本 80/443 のみ。
- 追加のアプリ用内部ポート（例：127.0.0.1:3000 等）を使う場合：
  - 外部から直接開けず、Apacheのリバースプロキシ配下で公開するのが基本方針（Seafile構成と整合）。
  - 既存vhost・リダイレクト方針と衝突しないURL設計が必要。

------

## 5. Web公開ディレクトリ（DocumentRoot）と配置規約

- DocumentRoot（例）
  - `charapla.net`：`/var/www/charapla.net/public`
- 重要：SFTP運用と一致させることで「アップロードしたのに404」事故を避ける（過去に発生）

------

## 6. SFTP（アップロード運用）設計

### 6.1 SFTP専用ユーザー

- SFTP専用ユーザーを用意し、chroot + internal-sftp を使う設計が標準
- `/sftp/<username>` を chroot ルートにする

### 6.2 bind mount で実体に接続

- SFTPの見かけ上のアップロード先から、実際のWeb公開ディレクトリ配下へ bind mount で接続する方式が推奨
- これにより「SFTPで置いたファイルがWeb側に反映されない」構造事故を避ける。

### 6.3 権限・所有（Web配信と両立）

- 運用方針として、SFTPで触る公開領域は `sftpuser:apache` など Webサーバと整合するグループ設計を取る
- SELinux コンテキスト：書込みが必要な領域は `httpd_sys_rw_content_t` を前提に設計する

------

## 7. ログとトラブルシュート導線

### 7.1 Apache

- `/var/log/httpd/error_log`
- `/var/log/httpd/access_log`

### 7.2 Seafile

- `/opt/seafile/logs/seafile.log`
- `/opt/seafile/logs/seahub.log`

### 7.3 よくある障害パターン（設計時に潰す）

- vhost競合／意図しないリダイレクト：古いconfが残っている、`.conf` のまま退避して読み込まれている
- Apacheが「停止→起動」で落ちる：Listen重複（reload運用で潜伏しやすい）
- SFTP後に404：アップロード先とDocumentRootがズレている／vhost整理不備

------

## 8. アプリ設計に要求される「守るべき制約」

- 外部公開ポートは 80/443 を前提にする（追加開放は原則しない）。
- 新規Apache設定は以下を絶対遵守：
  - Listenの追加禁止（所定ファイルの1行運用を壊す）
  - 無効化ファイルを `.conf` で残さない
  - 反映は `apachectl -t` → reload を標準にする
- SFTPで触る領域とアプリの書込み領域（uploads/cache/tmp等）を分離し、SELinux/所有/パーミッションが破綻しないように設計する。

------

## 9. 本アプリの実装判断パラメータ

> 以下は requirements.md と照らし合わせて確定した値

| 項目 | 値 |
|------|-----|
| **公開先** | `charapla.net/app/customer-manager/` |
| **実行方式** | Python（WSGI / Apacheリバースプロキシ）推奨 |
| **永続データ** | SQLite（軽量・単一ユーザー前提のため） |
| **書込みディレクトリ** | `data/`（DB）、`logs/`（ログ）、`uploads/`（CSV一時保存） |
| **認証** | アプリ独自（簡易認証、単一ユーザー） |
| **デプロイ経路** | Git pull または SFTP |
| **バックアップ対象** | SQLiteファイル、設定ファイル |

------

## 10. ディレクトリ構成案

```
/var/www/charapla.net/
├── public/                    # DocumentRoot
│   └── app/
│       └── customer-manager/  # アプリ公開ディレクトリ
│           └── static/        # 静的ファイル（CSS/JS）
└── customer-manager/          # アプリ本体（非公開領域）
    ├── app/                   # Pythonアプリケーション
    ├── data/                  # SQLite DB
    ├── logs/                  # アプリログ
    ├── uploads/               # 一時アップロード
    └── config/                # 設定ファイル
```

------

## 11. SELinux コンテキスト設計

| ディレクトリ | コンテキスト | 備考 |
|-------------|-------------|------|
| `public/app/customer-manager/` | `httpd_sys_content_t` | 読み取り専用 |
| `customer-manager/data/` | `httpd_sys_rw_content_t` | DB書込み |
| `customer-manager/logs/` | `httpd_sys_rw_content_t` | ログ書込み |
| `customer-manager/uploads/` | `httpd_sys_rw_content_t` | 一時ファイル |
