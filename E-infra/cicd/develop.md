# CI/CD実装ステップ

## 重要な概念とキーワード

### 継続的インテグレーション
- 自動ビルド
- 自動テスト
- コード品質チェック
- アーティファクト管理

### 継続的デリバリー
- デプロイメント自動化
- 環境管理
- リリース管理
- ロールバック

### パイプライン
- ワークフロー
- ステージ
- ジョブ
- トリガー

## 実装ステップ

### 1. GitHub Actionsワークフローの作成
```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: '1.19'
    
    - name: Install dependencies
      run: go mod download
    
    - name: Run tests
      run: go test -v ./...
    
    - name: Run linter
      uses: golangci/golangci-lint-action@v2
      with:
        version: latest

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1
    
    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: my-app:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v2
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1
    
    - name: Update ECS service
      run: |
        aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment
```

### 2. テスト自動化の設定
```go
// main_test.go
package main

import (
    "testing"
    "net/http"
    "net/http/httptest"
)

func TestHealthCheck(t *testing.T) {
    req, err := http.NewRequest("GET", "/health", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(healthCheckHandler)

    handler.ServeHTTP(rr, req)

    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            status, http.StatusOK)
    }

    expected := `{"status":"ok"}`
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v",
            rr.Body.String(), expected)
    }
}
```

### 3. デプロイメント設定
```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

### 4. 環境変数の管理
```yaml
# .env.example
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
API_KEY=your-api-key

# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to production
      env:
        DB_HOST: ${{ secrets.DB_HOST }}
        DB_PORT: ${{ secrets.DB_PORT }}
        DB_USER: ${{ secrets.DB_USER }}
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        API_KEY: ${{ secrets.API_KEY }}
      run: |
        # デプロイメントスクリプト
```

### 5. セキュリティスキャンの設定
```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Run Snyk to check for vulnerabilities
      uses: snyk/actions/golang@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
    
    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'My App'
        path: '.'
        format: 'HTML'
```

## ベストプラクティス

### 1. パイプライン設計
- 適切なステージ分割
- 並列実行の活用
- キャッシュの活用
- アーティファクトの管理

### 2. テスト戦略
- ユニットテスト
- 統合テスト
- E2Eテスト
- パフォーマンステスト

### 3. セキュリティ
- シークレット管理
- 脆弱性スキャン
- コンテナスキャン
- アクセス制御

### 4. デプロイメント
- ブルー/グリーンデプロイ
- カナリアリリース
- ロールバック戦略
- 環境分離 
