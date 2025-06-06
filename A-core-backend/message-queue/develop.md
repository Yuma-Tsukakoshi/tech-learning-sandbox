# メッセージキュー開発ガイド

## 開発環境のセットアップ

### 必要条件
- Docker Desktop
- RabbitMQ 3.12以上
- Go 1.21以上

### セットアップ手順
1. RabbitMQの起動
```bash
docker-compose up -d
```

2. 管理画面へのアクセス
```
http://localhost:15672
デフォルトユーザー: guest
デフォルトパスワード: guest
```

## アーキテクチャ

### コンポーネント
- プロデューサー（メッセージ送信者）
- コンシューマー（メッセージ受信者）
- エクスチェンジ
- キュー
- バインディング

### 技術スタック
- RabbitMQ
- Go AMQP Client
- Docker

## 開発フロー

### プロデューサーの実装
```go
package main

import (
    "log"
    "github.com/streadway/amqp"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatalf("Failed to connect to RabbitMQ: %v", err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatalf("Failed to open a channel: %v", err)
    }
    defer ch.Close()

    q, err := ch.QueueDeclare(
        "task_queue", // name
        true,         // durable
        false,        // delete when unused
        false,        // exclusive
        false,        // no-wait
        nil,          // arguments
    )
    if err != nil {
        log.Fatalf("Failed to declare a queue: %v", err)
    }

    body := "Hello World!"
    err = ch.Publish(
        "",     // exchange
        q.Name, // routing key
        false,  // mandatory
        false,  // immediate
        amqp.Publishing{
            DeliveryMode: amqp.Persistent,
            ContentType:  "text/plain",
            Body:        []byte(body),
        })
    if err != nil {
        log.Fatalf("Failed to publish a message: %v", err)
    }
    log.Printf(" [x] Sent %s", body)
}
```

### コンシューマーの実装
```go
package main

import (
    "log"
    "github.com/streadway/amqp"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatalf("Failed to connect to RabbitMQ: %v", err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatalf("Failed to open a channel: %v", err)
    }
    defer ch.Close()

    q, err := ch.QueueDeclare(
        "task_queue", // name
        true,         // durable
        false,        // delete when unused
        false,        // exclusive
        false,        // no-wait
        nil,          // arguments
    )
    if err != nil {
        log.Fatalf("Failed to declare a queue: %v", err)
    }

    msgs, err := ch.Consume(
        q.Name, // queue
        "",     // consumer
        true,   // auto-ack
        false,  // exclusive
        false,  // no-local
        false,  // no-wait
        nil,    // args
    )
    if err != nil {
        log.Fatalf("Failed to register a consumer: %v", err)
    }

    forever := make(chan bool)

    go func() {
        for d := range msgs {
            log.Printf("Received a message: %s", d.Body)
        }
    }()

    log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
    <-forever
}
```

## パフォーマンス最適化

### メッセージの永続化
- キューとメッセージの永続化設定
- 確認応答（Acknowledgment）の使用
- プリフェッチ数の最適化

### スケーリング
- クラスタリング
- シャーディング
- 負荷分散

## モニタリング

### メトリクス
- メッセージ処理速度
- キュー長
- メモリ使用量
- ディスク使用量

### アラート設定
- キュー長の閾値
- メモリ使用率
- エラー率

## トラブルシューティング

### よくある問題
1. メッセージの損失
2. パフォーマンスの低下
3. メモリ使用量の増加

### 解決方法
- 永続化の確認
- プリフェッチ数の調整
- リソースの増強

## セキュリティ

### 認証と認可
- ユーザー認証
- 仮想ホストの分離
- アクセス制御

### データ保護
- TLS/SSL暗号化
- メッセージの暗号化
- 監査ログ

## 運用管理

### バックアップ
- 設定のバックアップ
- メッセージのバックアップ
- クラスタ設定のバックアップ

### メンテナンス
- キューのクリーンアップ
- ログローテーション
- パフォーマンスチューニング

## ドキュメント

### 設計ドキュメント
- アーキテクチャ図
- フロー図
- エラーハンドリング

### 運用マニュアル
- 監視手順
- バックアップ手順
- 復旧手順 
