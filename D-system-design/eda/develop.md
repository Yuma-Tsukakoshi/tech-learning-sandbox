# イベント駆動アーキテクチャ実装ステップ

## 重要な概念とキーワード

### イベント駆動の基本概念
- イベント
- イベントストリーム
- イベントソーシング
- イベントストア

### メッセージングパターン
- パブリッシュ/サブスクライブ
- トピック
- キュー
- デッドレターキュー

### イベント処理
- 順序保証
- べき等性
- アトミック性
- 一貫性

## 実装ステップ

### 1. イベント定義
```go
package events

type Event interface {
    GetEventType() string
    GetAggregateID() string
    GetTimestamp() time.Time
}

type UserCreatedEvent struct {
    UserID    string    `json:"user_id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Timestamp time.Time `json:"timestamp"`
}

func (e *UserCreatedEvent) GetEventType() string {
    return "user.created"
}

func (e *UserCreatedEvent) GetAggregateID() string {
    return e.UserID
}

func (e *UserCreatedEvent) GetTimestamp() time.Time {
    return e.Timestamp
}
```

### 2. イベントパブリッシャーの実装
```go
package publisher

import (
    "github.com/Shopify/sarama"
)

type EventPublisher struct {
    producer sarama.SyncProducer
}

func NewEventPublisher(brokers []string) (*EventPublisher, error) {
    config := sarama.NewConfig()
    config.Producer.Return.Successes = true
    
    producer, err := sarama.NewSyncProducer(brokers, config)
    if err != nil {
        return nil, err
    }
    
    return &EventPublisher{
        producer: producer,
    }, nil
}

func (p *EventPublisher) PublishEvent(event Event) error {
    eventBytes, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    msg := &sarama.ProducerMessage{
        Topic: event.GetEventType(),
        Key:   sarama.StringEncoder(event.GetAggregateID()),
        Value: sarama.ByteEncoder(eventBytes),
    }
    
    _, _, err = p.producer.SendMessage(msg)
    return err
}
```

### 3. イベントハンドラーの実装
```go
package handler

import (
    "github.com/Shopify/sarama"
)

type EventHandler interface {
    HandleEvent(event Event) error
}

type UserEventHandler struct {
    userService *UserService
}

func (h *UserEventHandler) HandleEvent(event Event) error {
    switch e := event.(type) {
    case *UserCreatedEvent:
        return h.handleUserCreated(e)
    default:
        return fmt.Errorf("unknown event type: %s", event.GetEventType())
    }
}

func (h *UserEventHandler) handleUserCreated(event *UserCreatedEvent) error {
    // ユーザー作成イベントの処理
    return h.userService.CreateUser(event.UserID, event.Name, event.Email)
}
```

### 4. イベントコンシューマーの実装
```go
package consumer

import (
    "github.com/Shopify/sarama"
)

type EventConsumer struct {
    consumer sarama.Consumer
    handler  EventHandler
}

func NewEventConsumer(brokers []string, groupID string, handler EventHandler) (*EventConsumer, error) {
    config := sarama.NewConfig()
    config.Consumer.Return.Errors = true
    
    consumer, err := sarama.NewConsumer(brokers, config)
    if err != nil {
        return nil, err
    }
    
    return &EventConsumer{
        consumer: consumer,
        handler:  handler,
    }, nil
}

func (c *EventConsumer) ConsumeEvents(topics []string) error {
    for _, topic := range topics {
        partitionConsumer, err := c.consumer.ConsumePartition(topic, 0, sarama.OffsetNewest)
        if err != nil {
            return err
        }
        
        go func(pc sarama.PartitionConsumer) {
            for msg := range pc.Messages() {
                var event Event
                if err := json.Unmarshal(msg.Value, &event); err != nil {
                    // エラーハンドリング
                    continue
                }
                
                if err := c.handler.HandleEvent(event); err != nil {
                    // エラーハンドリング
                }
            }
        }(partitionConsumer)
    }
    
    return nil
}
```

### 5. イベントストアの実装
```go
package store

import (
    "github.com/EventStore/EventStore-Client-Go/esdb"
)

type EventStore struct {
    client *esdb.Client
}

func NewEventStore(connectionString string) (*EventStore, error) {
    settings, err := esdb.ParseConnectionString(connectionString)
    if err != nil {
        return nil, err
    }
    
    client, err := esdb.NewClient(settings)
    if err != nil {
        return nil, err
    }
    
    return &EventStore{
        client: client,
    }, nil
}

func (s *EventStore) AppendToStream(streamID string, events []Event) error {
    var eventData []esdb.EventData
    for _, event := range events {
        eventBytes, err := json.Marshal(event)
        if err != nil {
            return err
        }
        
        eventData = append(eventData, esdb.EventData{
            EventType: event.GetEventType(),
            Data:      eventBytes,
        })
    }
    
    _, err := s.client.AppendToStream(context.Background(), streamID, esdb.AppendToStreamOptions{}, eventData...)
    return err
}
```

## ベストプラクティス

### 1. イベント設計
- イベントの命名規則
- イベントのバージョニング
- イベントのスキーマ設計
- イベントのサイズ最適化

### 2. パフォーマンス最適化
- バッチ処理
- 並列処理
- キャッシュ戦略
- メッセージ圧縮

### 3. エラーハンドリング
- リトライメカニズム
- デッドレターキュー
- エラー監視
- リカバリー戦略

### 4. モニタリング
- イベント処理の追跡
- パフォーマンスメトリクス
- エラーレート
- レイテンシ監視 
