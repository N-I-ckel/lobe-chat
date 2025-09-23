# Cloudflare Zero Trust セットアップガイド

## 1. Access Application の作成

1. Cloudflare Zero Trust ダッシュボードにアクセス
2. 「Access」→「Applications」→「Add an application」
3. 「Self-hosted」を選択

### Lobe Chat用の設定:
- **Application name**: Lobe Chat
- **Application domain**: `chat.yourdomain.com`
- **Session duration**: 24 hours（お好みで調整）

### 認証ポリシーの設定:
- **Policy name**: Lobe Chat Access
- **Action**: Allow
- **Include**で以下のいずれかを設定:
  - Emails: 特定のメールアドレス
  - Email domains: 会社のドメイン（例：@company.com）
  - GitHub organizations: GitHubの組織
  - Google Workspace: Googleワークスペースのグループ

## 2. Backblaze B2 バケットの設定

### バケットの作成:
1. Backblazeダッシュボードで新しいバケットを作成
2. **Bucket Unique Name**: `lobe-chat-storage`（例）
3. **Files in Bucket are**: 「Public」を選択（重要）

### Application Keyの作成:
1. 「App Keys」セクションに移動
2. 「Add a New Application Key」をクリック
3. **Name**: Lobe Chat
4. **Allow access to bucket**: 作成したバケットを選択
5. **Type of access**: Read and Write
6. 生成されたkeyIDとapplicationKeyを保存

### CORS設定:
バケット設定でCORSルールを追加：

```json
[
  {
    "corsRuleName": "allowFromLobeChat",
    "allowedOrigins": ["https://chat.yourdomain.com"],
    "allowedOperations": ["s3_download", "s3_upload"],
    "allowedHeaders": ["*"],
    "maxAgeSeconds": 3600
  }
]
```

## 3. セットアップ手順

```bash
# 1. 作業ディレクトリの作成
mkdir lobe-chat-production && cd lobe-chat-production

# 2. 設定ファイルのダウンロード
curl -O https://raw.githubusercontent.com/lobehub/lobe-chat/HEAD/docker-compose/local/init_data.json

# 3. 環境変数ファイルの作成
cp .env.cloudflare.example .env

# 4. .envファイルを編集
# - ドメイン名を設定
# - パスワードとシークレットキーを生成・設定
# - Backblaze B2の認証情報を設定
# - LLMプロバイダーのAPIキーを設定

# 5. Docker Composeで起動
docker compose -f docker-compose-cloudflare.yml up -d

# 6. ログの確認
docker logs -f lobe-chat
```

## 4. Casdoorの初期設定

1. `https://auth.yourdomain.com`にアクセス
2. 初期管理者でログイン（パスワードは自動生成されます）
3. 「Authentication」→「Applications」→「Add」
4. 以下を設定:
   - **Name**: LobeChat
   - **Display name**: LobeChat
   - **Logo**: （オプション）
   - **Redirect URIs**: 
     ```
     https://chat.yourdomain.com/api/auth/callback/casdoor
     ```
5. 生成されたClient IDとClient Secretを`.env`に設定
6. Docker Composeを再起動:
   ```bash
   docker compose -f docker-compose-cloudflare.yml restart lobe
   ```

## 5. 動作確認

1. `https://chat.yourdomain.com`にアクセス
2. Cloudflare Zero Trust（設定した場合）でログイン
3. Casdoorでユーザー登録/ログイン
4. LLMプロバイダーが正しく動作することを確認

## トラブルシューティング

### Backblaze B2の接続エラー
- バケットがPublicに設定されているか確認
- Application Keyの権限が正しいか確認
- エンドポイントURLがリージョンと一致しているか確認

### Cloudflare関連
- SSL設定が「Full」以上になっているか確認
- Page Rulesで誤ったキャッシュ設定がないか確認
- Zero Trustのポリシーが正しく設定されているか確認

### Casdoor認証エラー
- Redirect URIが正確に設定されているか確認
- Client ID/Secretが正しく設定されているか確認
- `AUTH_CASDOOR_ISSUER`のURLが正しいか確認