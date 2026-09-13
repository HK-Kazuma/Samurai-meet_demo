# ローカル起動確認の手順

最終更新: 2026-09-11

共有の検証用サーバーは使わず、開発者のPCでバックエンドを起動し、同じLANのiPhone実機（Expo Go）から接続して動作確認するための手順です。Google OAuth / Passkeyの外部登録を避けるため、ログインには**Demoアカウント**を使います。

この手順は開発中の動作確認用です。受入判定としての実機E2Eは [iPhone実機E2E手順書](../ios-real-device-e2e.md) に従ってください。Expo Goで画面が開いたことは実機E2EのPASSにはなりません。

```
[iPhone実機: Expo Go] ──同一LAN──> [PC: Metro :8081 / Go API :8080 / PostgreSQL :5432]
```

## 0. 事前に一度だけ行う設定

### 0.1 PCのLAN IPを確認する

```powershell
Get-NetIPAddress -InterfaceAlias "Wi-Fi" -AddressFamily IPv4 | Select-Object IPAddress, PrefixLength
```

VMware（`VMnet1`/`VMnet8`）やWSL（`vEthernet`）の仮想アダプタが存在する環境では、`Get-NetIPAddress`を引数なしで実行すると複数のIPが出ます。**実機から到達できるのはWi-Fi（または有線LAN）アダプタのIPだけ**です。仮想アダプタのIPを設定すると、PC上のcurlは成功するのに実機からは到達できません。

### 0.2 Wi-Fiをプライベートネットワークにする

「パブリック」のままだとWindowsファイアウォールが受信接続を既定でブロックし、実機から接続できません。管理者権限のPowerShellで実行します。

```powershell
Set-NetConnectionProfile -InterfaceAlias "Wi-Fi" -NetworkCategory Private
```

### 0.3 ファイアウォールで8080・8081を許可する

管理者権限のPowerShellで実行します。

```powershell
New-NetFirewallRule -DisplayName "Samurai Meet Backend (8080)" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "Expo Metro Dev Server (8081)" -Direction Inbound -Protocol TCP -LocalPort 8081 -Action Allow -Profile Any
```

### 0.4 Demo用データベースを作る

Demoモードは本番既定の`samurai_meet`と分離したDBを要求します。

```powershell
cd backend
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml exec -T postgres psql -U samurai_meet_app -d samurai_meet -c "CREATE DATABASE samurai_meet_demo OWNER samurai_meet_app;"
```

### 0.5 `backend/.env` を用意する

`.env.example`をもとに、Demoモードの安全ガードを満たす値にします。`APP_ENV`がproduction以外でも、`DEMO_ACCOUNT_ENABLED=true`のときは次の4点が検証され、満たさないと起動が停止します（`internal/config/config.go`）。

| 要件 | 設定 |
| --- | --- |
| Google OAuthを無効化する | `GOOGLE_LOGIN_ENABLED=false` |
| 本番既定と別のDBにする | `DB_NAME=samurai_meet_demo` |
| 本番既定と別の画像保存先にする | `IMAGE_STORAGE_DIR=storage/demo-images` |
| 署名鍵を設定する | `JWS_SIGNING_KEY=<32 bytesのBase64URL>` |

あわせて次を設定します。

- `GEMINI_API_KEY`: 募集作成時の分類API（`POST /api/v1/recruitments/classify`）に必須。未設定だと503になり募集を公開できません。
- `CHAT_MODERATION_DEV_FREE_MODE=true`: チャットまで確認する場合のみ。OpenAIキーなしで外部送信しないローカル判定に切り替わります（起動時に警告が出ます。実データを入れず、正式運用前に無効化します）。
- `DEV_CLIENT_ORIGIN=http://<PCのLAN IP>:8081`

### 0.6 `frontend/.env` を用意する

```
EXPO_PUBLIC_API_BASE_URL=http://<PCのLAN IP>:8080/api/v1
EXPO_PUBLIC_WEB_APP_ORIGIN=http://<PCのLAN IP>:8081
EXPO_PUBLIC_DEMO_ACCOUNT_ENABLED=true
```

`127.0.0.1`や`localhost`は実機自身を指すため使えません。IPが変わったら`backend/.env`の`DEV_CLIENT_ORIGIN`と合わせて更新し、Expoとバックエンドを再起動します。

## 1. 毎回の起動順序

### 1.1 PostgreSQL

```powershell
cd backend
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml ps
```

Docker Desktopが起動していないと `open //./pipe/dockerDesktopLinuxEngine` のエラーになります。先にDocker Desktopを起動し、エンジンが動き出すまで待ちます。

### 1.2 バックエンド

```powershell
cd backend
go run ./cmd/server
```

起動ログに `backend server listening on :8080` が出たら、別ターミナルで疎通を確認します。

```powershell
curl.exe --fail --silent --show-error http://127.0.0.1:8080/healthz
curl.exe --fail --silent --show-error http://127.0.0.1:8080/readyz
```

### 1.3 Expo

**ポート8081で起動すること**を確認します。8081が使用中だと8082へ移り、ファイアウォール未許可のため実機からタイムアウトします。既存のExpoプロセスが残っていないか先に確認してください。

```powershell
cd frontend
$env:REACT_NATIVE_PACKAGER_HOSTNAME="<PCのLAN IP>"
bun run start
```

`REACT_NATIVE_PACKAGER_HOSTNAME`を指定しないと、ExpoのLAN IP自動検出がVMwareなどの仮想アダプタを選ぶことがあります。表示されるURLが `exp://<PCのLAN IP>:8081` になっていることを必ず確認します。

Expoアカウントのログインを求められた場合は、`npx expo login`でログインするか、ログインせずに済ませたい場合は `bun run start:offline`（`expo start --offline --lan`）を使います。

## 2. 実機での確認

### 2.1 接続確認

QRを読む前に、iPhoneのSafariで次を開いて到達性を確認すると切り分けが早くなります。

- `http://<PCのLAN IP>:8080/healthz` → `{"status":"ok"}`
- `http://<PCのLAN IP>:8081/status` → `packager-status:running`

両方とも表示されるのにExpo Goだけ失敗する場合は、QRが指すポート・IPが上記と一致しているかを疑います。

### 2.2 Demoアカウントでログイン

1. Expo GoでQRコードを読み込む。2台で確認する場合、**同じQRコードを両方の端末で読んで構いません**。セッションは端末ごとに保存されます。
2. 未ログイン画面の「デモを体験する」をタップし、表示言語と利用モード（外国人側／日本人側）を選ぶ。メール登録もPasskeyも不要でアカウントが発行されます。

Demoアカウントと通常アカウントはサーバー側でスコープが分離されており（`internal/accountscope`）、混在させるとマッチできません。**2台とも必ずDemoアカウントにします**。Demoアカウントは24時間で失効するため、期限切れになったら発行し直します。

### 2.3 マッチ確認の流れ

1台目を外国人側（募集者）、2台目を日本人側（応募者）にして、募集作成・公開 → 検索・応募 → 承認 → アプリ内通知の順に確認します。各画面と期待APIの詳細は [iPhone実機E2E手順書の4章](../ios-real-device-e2e.md#4-募集--応募--承認--通知) を参照してください。1台しかない場合は、フェーズごとにログアウトして別のDemoアカウントを発行します。

## 3. つまずきやすい点

| 症状 | 原因と対処 |
| --- | --- |
| Expo GoでQRを読むと "There was a problem running the requested project. The request timed out." | QRが指すIP・ポートに実機から到達できていない。仮想アダプタのIPになっていないか、8082など未許可ポートで起動していないかを確認する |
| PCのcurlは成功するのに実機から到達できない | 仮想アダプタのIPを設定している、Wi-Fiがパブリックプロファイル、またはファイアウォール未許可 |
| `configuration validation failed: demo configuration is unsafe` | 0.5のDemoモード要件を満たしていない |
| 募集の確認画面で分類が503になる | `GEMINI_API_KEY`が未設定または無効 |
| 実機への転送が遅くタイムアウトする | Wi-Fiアダプタの詳細設定で`Power Saving`が有効だとパケットロスが出ることがある。`無効`にして再試行する |
| Expoが二重起動している | 既存プロセスを停止してから起動し直す。ポートが8081であることを確認する |

## 4. 参照

- [フロントエンド開発・API接続](../../frontend/README.md)
- [バックエンド](../../backend/README.md)
- [フロントエンドとAPIのつなぎ方](frontend-connection.md)
- [iPhone実機E2E手順書](../ios-real-device-e2e.md)
