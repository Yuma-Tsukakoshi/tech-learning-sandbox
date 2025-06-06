# オブザーバビリティ実装ステップ

## 重要な概念とキーワード

### メトリクス
- カウンター
- ゲージ
- ヒストグラム
- サマリー

### ログ
- 構造化ログ
- ログレベル
- ログローテーション
- ログ集約

### トレース
- 分散トレーシング
- スパン
- トレースID
- コンテキスト伝播

## 実装ステップ

### 1. Prometheusの設定
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['my-app:8080']
    metrics_path: '/metrics'
```

### 2. アプリケーションのメトリクス実装
```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
}

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(r.Method, r.URL.Path))
        next.ServeHTTP(w, r)
        timer.ObserveDuration()
        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
    })
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    // ... 他のルートハンドラ
}
```

### 3. Grafanaダッシュボードの作成
```json
{
  "dashboard": {
    "id": null,
    "title": "My App Dashboard",
    "panels": [
      {
        "title": "HTTP Requests",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Request Duration",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      }
    ]
  }
}
```

### 4. アラートルールの設定
```yaml
# alert.rules
groups:
- name: my-app
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: High error rate detected
      description: Error rate is above 10% for the last 5 minutes

  - alert: HighLatency
    expr: rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: High latency detected
      description: Average request duration is above 1 second
```

### 5. ログ集約の設定
```yaml
# fluentd.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match my-app.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix my-app
  include_tag_key true
  tag_key @log_name
</match>
```

## ベストプラクティス

### 1. メトリクス設計
- 適切なメトリクス名の使用
- ラベルの適切な使用
- バケットサイズの最適化
- メトリクスの集約

### 2. ログ設計
- 構造化ログの採用
- 適切なログレベルの使用
- ログローテーションの設定
- 機密情報の除外

### 3. トレース設計
- トレースコンテキストの伝播
- スパンの適切な設計
- サンプリングレートの設定
- トレースデータの保持期間

### 4. アラート設計
- 適切な閾値の設定
- アラートの優先順位付け
- アラートのグルーピング
- アラートの抑制 
