# マイクロサービス実装ステップ

## 重要な概念とキーワード

### アーキテクチャパターン
- サービスディスカバリ
- サーキットブレーカー
- バルクヘッド
- フォールバック

### 通信パターン
- 同期通信（gRPC/REST）
- 非同期通信（メッセージキュー）
- イベント駆動
- CQRS

### デプロイメント
- コンテナ化
- オーケストレーション
- ブルー/グリーンデプロイ
- カナリアリリース

## 実装ステップ

### 1. サービス定義とgRPC実装
```protobuf
syntax = "proto3";

package user;

service UserService {
    rpc GetUser (GetUserRequest) returns (User) {}
    rpc CreateUser (CreateUserRequest) returns (User) {}
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
}

message GetUserRequest {
    string id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}
```

### 2. サービスディスカバリの実装
```go
package main

import (
    "github.com/hashicorp/consul/api"
)

func registerService(serviceName, serviceID, address string, port int) error {
    config := api.DefaultConfig()
    client, err := api.NewClient(config)
    if err != nil {
        return err
    }

    registration := &api.AgentServiceRegistration{
        ID:      serviceID,
        Name:    serviceName,
        Address: address,
        Port:    port,
        Check: &api.AgentServiceCheck{
            HTTP:     "http://localhost:8080/health",
            Interval: "10s",
            Timeout:  "5s",
        },
    }

    return client.Agent().ServiceRegister(registration)
}
```

### 3. サーキットブレーカーの実装
```go
package main

import (
    "github.com/sony/gobreaker"
)

func newCircuitBreaker() *gobreaker.CircuitBreaker {
    return gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        "user-service",
        MaxRequests: 3,
        Interval:    60 * time.Second,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
    })
}

func callService(cb *gobreaker.CircuitBreaker, req interface{}) (interface{}, error) {
    return cb.Execute(func() (interface{}, error) {
        // サービス呼び出し
        return callUserService(req)
    })
}
```

### 4. サービス間通信の実装
```go
package main

import (
    "github.com/go-kit/kit/endpoint"
    "github.com/go-kit/kit/transport/grpc"
)

type UserService struct {
    GetUserEndpoint    endpoint.Endpoint
    CreateUserEndpoint endpoint.Endpoint
}

func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    response, err := s.GetUserEndpoint(ctx, req)
    if err != nil {
        return nil, err
    }
    return response.(*pb.User), nil
}

func makeGetUserEndpoint(svc UserService) endpoint.Endpoint {
    return func(ctx context.Context, request interface{}) (interface{}, error) {
        req := request.(*pb.GetUserRequest)
        return svc.GetUser(ctx, req)
    }
}
```

### 5. デプロイメント設定
```yaml
# docker-compose.yml
version: '3'
services:
  user-service:
    build: ./user-service
    ports:
      - "8080:8080"
    environment:
      - CONSUL_HOST=consul
      - CONSUL_PORT=8500
    depends_on:
      - consul

  order-service:
    build: ./order-service
    ports:
      - "8081:8081"
    environment:
      - CONSUL_HOST=consul
      - CONSUL_PORT=8500
    depends_on:
      - consul

  consul:
    image: consul:latest
    ports:
      - "8500:8500"
```

## ベストプラクティス

### 1. サービス設計
- 適切なサービス境界
- 疎結合の維持
- データの一貫性
- スケーラビリティ

### 2. レジリエンス
- サーキットブレーカー
- リトライメカニズム
- タイムアウト設定
- フォールバック戦略

### 3. モニタリング
- 分散トレーシング
- メトリクス収集
- ログ集約
- アラート設定

### 4. セキュリティ
- サービス間認証
- 暗号化通信
- アクセス制御
- セキュリティ監査 
