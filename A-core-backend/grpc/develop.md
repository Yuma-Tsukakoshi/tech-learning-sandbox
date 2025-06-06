# gRPC実装ステップ

## 重要な概念とキーワード

### gRPCの基本概念
- Protocol Buffers
- サービス定義
- ストリーミング
- 双方向通信

### 通信パターン
- Unary RPC
- Server Streaming RPC
- Client Streaming RPC
- Bidirectional Streaming RPC

### 認証とセキュリティ
- TLS/SSL
- トークンベース認証
- インターセプター

## 実装ステップ

### 1. Protocol Buffersの定義
```protobuf
syntax = "proto3";

package user;

service UserService {
    rpc GetUser (GetUserRequest) returns (User) {}
    rpc CreateUser (CreateUserRequest) returns (User) {}
    rpc UpdateUser (UpdateUserRequest) returns (User) {}
    rpc DeleteUser (DeleteUserRequest) returns (Empty) {}
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

message UpdateUserRequest {
    string id = 1;
    string name = 2;
    string email = 3;
}

message DeleteUserRequest {
    string id = 1;
}

message Empty {}
```

### 2. サーバー実装
```go
package main

import (
    "context"
    "log"
    "net"
    pb "path/to/generated/proto"

    "google.golang.org/grpc"
)

type server struct {
    pb.UnimplementedUserServiceServer
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // ユーザー取得ロジック
    return &pb.User{
        Id:    req.Id,
        Name:  "John Doe",
        Email: "john@example.com",
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &server{})

    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### 3. クライアント実装
```go
package main

import (
    "context"
    "log"
    pb "path/to/generated/proto"

    "google.golang.org/grpc"
)

func main() {
    conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewUserServiceClient(conn)

    // ユーザー取得
    user, err := client.GetUser(context.Background(), &pb.GetUserRequest{Id: "1"})
    if err != nil {
        log.Fatalf("could not get user: %v", err)
    }
    log.Printf("User: %v", user)
}
```

### 4. ストリーミング実装
```go
// サーバーストリーミング
func (s *server) StreamUsers(req *pb.StreamUsersRequest, stream pb.UserService_StreamUsersServer) error {
    users := []*pb.User{
        {Id: "1", Name: "User 1"},
        {Id: "2", Name: "User 2"},
    }

    for _, user := range users {
        if err := stream.Send(user); err != nil {
            return err
        }
    }
    return nil
}

// クライアントストリーミング
func (s *server) CreateUsers(stream pb.UserService_CreateUsersServer) error {
    for {
        user, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.CreateUsersResponse{Count: count})
        }
        if err != nil {
            return err
        }
        // ユーザー作成処理
    }
}
```

### 5. エラーハンドリングとリトライ
```go
func main() {
    conn, err := grpc.Dial(
        "localhost:50051",
        grpc.WithInsecure(),
        grpc.WithUnaryInterceptor(grpc_retry.UnaryClientInterceptor()),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
}
```

## ベストプラクティス

### 1. エラーハンドリング
- 適切なエラーコードの使用
- エラーの詳細情報の提供
- リトライ戦略の実装

### 2. パフォーマンス最適化
- コネクションプールの設定
- ストリーミングの適切な使用
- バッチ処理の実装

### 3. セキュリティ
- TLS/SSLの設定
- 認証・認可の実装
- レート制限の設定

### 4. モニタリング
- メトリクスの収集
- トレーシングの実装
- ヘルスチェックの設定 
