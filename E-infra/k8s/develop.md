# Kubernetes実装ステップ

## 重要な概念とキーワード

### Kubernetesの基本概念
- Pod
- Deployment
- Service
- ConfigMap/Secret

### リソース管理
- リソース制限
- リソース要求
- オートスケーリング
- ノードセレクタ

### ネットワーク
- Service
- Ingress
- NetworkPolicy
- DNS

## 実装ステップ

### 1. 基本的なDeploymentの作成
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
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
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 2. Serviceの作成
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### 3. Ingressの設定
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

### 4. ConfigMapとSecretの設定
```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  config.yaml: |
    database:
      host: db.example.com
      port: 5432
    logging:
      level: info

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
type: Opaque
data:
  db-password: base64encodedpassword
  api-key: base64encodedapikey
```

### 5. HorizontalPodAutoscalerの設定
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## ベストプラクティス

### 1. リソース管理
- 適切なリソース制限の設定
- リソース要求の最適化
- オートスケーリングの設定
- ノードの効率的な利用

### 2. セキュリティ
- 最小権限の原則
- ネットワークポリシー
- シークレット管理
- コンテナセキュリティ

### 3. 可用性
- レプリカ数の適切な設定
- 分散配置の戦略
- ヘルスチェックの実装
- 障害復旧計画

### 4. モニタリング
- メトリクスの収集
- ログ管理
- アラート設定
- パフォーマンス監視 
