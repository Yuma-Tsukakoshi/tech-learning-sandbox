# GraphQL実装ステップ

## 重要な概念とキーワード

### GraphQLの基本概念
- スキーマ定義
- クエリとミューテーション
- リゾルバー
- フラグメント

### データフェッチング
- 単一リクエスト
- バッチ処理
- データローダー
- N+1問題の解決

### 認証とセキュリティ
- コンテキスト
- ディレクティブ
- ミドルウェア
- レート制限

## 実装ステップ

### 1. スキーマ定義
```graphql
type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]
}

type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
}

type Query {
    user(id: ID!): User
    users: [User!]!
    post(id: ID!): Post
    posts: [Post!]!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
}

input CreateUserInput {
    name: String!
    email: String!
}

input UpdateUserInput {
    name: String
    email: String
}
```

### 2. リゾルバーの実装
```go
package main

import (
    "github.com/99designs/gqlgen/graphql"
    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
)

type Resolver struct {
    userService *UserService
    postService *PostService
}

func (r *Resolver) Query() QueryResolver {
    return &queryResolver{r}
}

func (r *Resolver) Mutation() MutationResolver {
    return &mutationResolver{r}
}

type queryResolver struct{ *Resolver }

func (r *queryResolver) User(ctx context.Context, id string) (*User, error) {
    return r.userService.GetUser(ctx, id)
}

func (r *queryResolver) Users(ctx context.Context) ([]*User, error) {
    return r.userService.GetUsers(ctx)
}

type mutationResolver struct{ *Resolver }

func (r *mutationResolver) CreateUser(ctx context.Context, input CreateUserInput) (*User, error) {
    return r.userService.CreateUser(ctx, input)
}
```

### 3. データローダーの実装
```go
func DataLoaderMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        
        // ユーザーローダーの作成
        userLoader := dataloader.NewBatchedLoader(func(keys dataloader.Keys) []*dataloader.Result {
            // バッチでユーザーを取得
            users, err := userService.GetUsersByIDs(keys.Keys())
            if err != nil {
                return []*dataloader.Result{{Error: err}}
            }

            // 結果のマッピング
            results := make([]*dataloader.Result, len(keys))
            for i, key := range keys {
                if user, ok := users[key.String()]; ok {
                    results[i] = &dataloader.Result{Data: user}
                } else {
                    results[i] = &dataloader.Result{Error: fmt.Errorf("user not found")}
                }
            }
            return results
        })

        ctx = context.WithValue(ctx, "userLoader", userLoader)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 4. エラーハンドリング
```go
func (r *queryResolver) User(ctx context.Context, id string) (*User, error) {
    user, err := r.userService.GetUser(ctx, id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, &gqlerror.Error{
                Message: "User not found",
                Extensions: map[string]interface{}{
                    "code": "NOT_FOUND",
                },
            }
        }
        return nil, err
    }
    return user, nil
}
```

### 5. 認証と認可
```go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            next.ServeHTTP(w, r)
            return
        }

        user, err := validateToken(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func (r *mutationResolver) CreateUser(ctx context.Context, input CreateUserInput) (*User, error) {
    user := ctx.Value("user").(*User)
    if !user.IsAdmin {
        return nil, &gqlerror.Error{
            Message: "Unauthorized",
            Extensions: map[string]interface{}{
                "code": "UNAUTHORIZED",
            },
        }
    }
    return r.userService.CreateUser(ctx, input)
}
```

## ベストプラクティス

### 1. パフォーマンス最適化
- データローダーの使用
- クエリの複雑さの制限
- キャッシュの実装
- バッチ処理の活用

### 2. セキュリティ
- 入力バリデーション
- レート制限
- クエリの複雑さの制限
- 認証・認可の実装

### 3. エラーハンドリング
- 適切なエラーメッセージ
- エラーコードの使用
- エラーの詳細情報の提供

### 4. モニタリング
- クエリの実行時間
- エラー率
- リソース使用量
- パフォーマンスメトリクス 
