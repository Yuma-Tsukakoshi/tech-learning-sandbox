# CQRSとSagaパターン実装ステップ

## 重要な概念とキーワード

### CQRS（Command Query Responsibility Segregation）
- コマンドとクエリの分離
- 読み取りモデルと書き込みモデル
- イベントソーシング
- 最終的な一貫性

### Sagaパターン
- 分散トランザクション
- 補償トランザクション
- オーケストレーション
- コレオグラフィ

### イベント処理
- イベントストア
- イベントハンドラー
- イベントバージョニング
- スナップショット

## 実装ステップ

### 1. CQRSの基本実装
```go
// コマンド
type CreateOrderCommand struct {
    UserID    string
    ProductID string
    Quantity  int
}

type OrderCommandHandler struct {
    eventStore EventStore
}

func (h *OrderCommandHandler) HandleCreateOrder(cmd CreateOrderCommand) error {
    // 注文作成のビジネスロジック
    order := NewOrder(cmd.UserID, cmd.ProductID, cmd.Quantity)
    
    // イベントの生成と保存
    event := OrderCreatedEvent{
        OrderID:   order.ID,
        UserID:    order.UserID,
        ProductID: order.ProductID,
        Quantity:  order.Quantity,
    }
    
    return h.eventStore.AppendToStream(order.ID, []Event{event})
}

// クエリ
type GetOrderQuery struct {
    OrderID string
}

type OrderQueryHandler struct {
    readModel OrderReadModel
}

func (h *OrderQueryHandler) HandleGetOrder(query GetOrderQuery) (*OrderDTO, error) {
    return h.readModel.GetOrder(query.OrderID)
}
```

### 2. イベントハンドラーの実装
```go
type OrderEventHandler struct {
    readModel OrderReadModel
}

func (h *OrderEventHandler) HandleOrderCreated(event OrderCreatedEvent) error {
    // 読み取りモデルの更新
    return h.readModel.CreateOrder(OrderDTO{
        ID:        event.OrderID,
        UserID:    event.UserID,
        ProductID: event.ProductID,
        Quantity:  event.Quantity,
        Status:    "created",
    })
}

func (h *OrderEventHandler) HandleOrderCancelled(event OrderCancelledEvent) error {
    // 読み取りモデルの更新
    return h.readModel.UpdateOrderStatus(event.OrderID, "cancelled")
}
```

### 3. Sagaパターンの実装
```go
// オーケストレーションベースのSaga
type OrderSaga struct {
    eventStore EventStore
    services   map[string]Service
}

func (s *OrderSaga) StartOrderProcess(orderID string) error {
    // 注文処理の開始
    return s.eventStore.AppendToStream(orderID, []Event{
        OrderProcessStartedEvent{OrderID: orderID},
    })
}

func (s *OrderSaga) HandleOrderProcessStarted(event OrderProcessStartedEvent) error {
    // 在庫確認
    if err := s.services["inventory"].ReserveStock(event.OrderID); err != nil {
        return s.compensateOrderProcess(event.OrderID)
    }
    
    // 支払い処理
    if err := s.services["payment"].ProcessPayment(event.OrderID); err != nil {
        return s.compensateOrderProcess(event.OrderID)
    }
    
    // 注文完了
    return s.eventStore.AppendToStream(event.OrderID, []Event{
        OrderProcessCompletedEvent{OrderID: event.OrderID},
    })
}

func (s *OrderSaga) compensateOrderProcess(orderID string) error {
    // 補償トランザクションの実行
    return s.eventStore.AppendToStream(orderID, []Event{
        OrderProcessFailedEvent{OrderID: orderID},
    })
}
```

### 4. イベントストアの実装
```go
type EventStore interface {
    AppendToStream(streamID string, events []Event) error
    GetStream(streamID string) ([]Event, error)
    GetStreamVersion(streamID string) (int, error)
}

type PostgresEventStore struct {
    db *sql.DB
}

func (s *PostgresEventStore) AppendToStream(streamID string, events []Event) error {
    tx, err := s.db.Begin()
    if err != nil {
        return err
    }
    
    version, err := s.GetStreamVersion(streamID)
    if err != nil {
        tx.Rollback()
        return err
    }
    
    for i, event := range events {
        _, err = tx.Exec(
            "INSERT INTO events (stream_id, version, event_type, data) VALUES ($1, $2, $3, $4)",
            streamID,
            version+i+1,
            event.GetEventType(),
            event,
        )
        if err != nil {
            tx.Rollback()
            return err
        }
    }
    
    return tx.Commit()
}
```

### 5. 読み取りモデルの実装
```go
type OrderReadModel interface {
    CreateOrder(order OrderDTO) error
    UpdateOrderStatus(orderID string, status string) error
    GetOrder(orderID string) (*OrderDTO, error)
}

type PostgresOrderReadModel struct {
    db *sql.DB
}

func (m *PostgresOrderReadModel) CreateOrder(order OrderDTO) error {
    _, err := m.db.Exec(
        "INSERT INTO orders (id, user_id, product_id, quantity, status) VALUES ($1, $2, $3, $4, $5)",
        order.ID,
        order.UserID,
        order.ProductID,
        order.Quantity,
        order.Status,
    )
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
- キャッシュ戦略
- インデックス最適化
- 非同期処理

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
