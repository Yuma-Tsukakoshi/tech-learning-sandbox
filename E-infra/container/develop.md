# コンテナ技術実装ステップ

## 重要な概念とキーワード

### コンテナの基本概念
- コンテナイメージ
- レイヤー
- 名前空間
- cgroups

### Dockerの基本概念
- Dockerfile
- マルチステージビルド
- ボリューム
- ネットワーク

### コンテナオーケストレーション
- サービスディスカバリ
- ロードバランシング
- スケーリング
- ヘルスチェック

## 実装ステップ

### 1. 基本的なDockerfileの作成
```dockerfile
# ベースイメージの選択
FROM golang:1.21-alpine

# 作業ディレクトリの設定
WORKDIR /app

# 依存関係のコピーとインストール
COPY go.mod go.sum ./
RUN go mod download

# ソースコードのコピー
COPY . .

# アプリケーションのビルド
RUN go build -o main .

# 実行ユーザーの設定
USER nobody

# ポートの公開
EXPOSE 8080

# アプリケーションの実行
CMD ["./main"]
```

### 2. マルチステージビルドの実装
```dockerfile
# ビルドステージ
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# 実行ステージ
FROM alpine:latest

WORKDIR /app

# 必要なファイルのみをコピー
COPY --from=builder /app/main .
COPY --from=builder /app/config ./config

USER nobody
EXPOSE 8080

CMD ["./main"]
```

### 3. Docker Composeの設定
```yaml
version: '3'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      - db
    volumes:
      - ./config:/app/config
    networks:
      - app-network

  db:
    image: postgres:latest
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### 4. ヘルスチェックの実装
```dockerfile
# Dockerfileにヘルスチェックを追加
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1
```

```yaml
# docker-compose.ymlにヘルスチェックを追加
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

### 5. セキュリティ設定
```dockerfile
# 非rootユーザーの作成
RUN adduser -D -g '' appuser

# 必要な権限のみを付与
RUN chown -R appuser:appuser /app

# 非rootユーザーに切り替え
USER appuser

# セキュリティオプションの設定
RUN apk add --no-cache ca-certificates tzdata
```

## ベストプラクティス

### 1. イメージ最適化
- マルチステージビルドの使用
- 不要なファイルの削除
- レイヤーの最適化
- キャッシュの活用

### 2. セキュリティ
- 非rootユーザーの使用
- 最小限の権限設定
- セキュリティスキャン
- シークレット管理

### 3. パフォーマンス
- イメージサイズの最適化
- キャッシュの活用
- リソース制限の設定
- ネットワーク最適化

### 4. 運用
- ログ管理
- モニタリング
- バックアップ
- 障害復旧 
