# キャッシュ戦略実装ステップ

## 重要な概念とキーワード

### キャッシュ戦略
- Cache-Aside (Read-Through)
- Write-Through
- Write-Behind
- Refresh-Ahead

### キャッシュの有効期限
- TTL (Time To Live)
- スライディング有効期限
- 絶対有効期限

### キャッシュの一貫性
- キャッシュ無効化
- キャッシュ更新
- キャッシュコヒーレンス

## 実装ステップ

### 1. Redisのセットアップ
```yaml
# docker-compose.yml
version: '3'
services:
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
```

### 2. 基本的なキャッシュ実装
```go
package main

import (
    "github.com/go-redis/redis/v8"
    "context"
    "time"
)

func main() {
    // Redisクライアントの初期化
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })

    ctx := context.Background()

    // キャッシュの設定
    err := rdb.Set(ctx, "key", "value", time.Hour).Err()
    if err != nil {
        panic(err)
    }

    // キャッシュの取得
    val, err := rdb.Get(ctx, "key").Result()
    if err != nil {
        panic(err)
    }
}
```

### 3. Cache-Asideパターンの実装
```go
func getDataWithCache(ctx context.Context, rdb *redis.Client, key string) (string, error) {
    // キャッシュから取得を試みる
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return val, nil
    }

    // キャッシュミスの場合、DBから取得
    data, err := fetchFromDB(key)
    if err != nil {
        return "", err
    }

    // キャッシュに保存
    err = rdb.Set(ctx, key, data, time.Hour).Err()
    if err != nil {
        return "", err
    }

    return data, nil
}
```

### 4. キャッシュの更新戦略
```go
// Write-Through
func updateDataWithCache(ctx context.Context, rdb *redis.Client, key, value string) error {
    // DBを更新
    err := updateDB(key, value)
    if err != nil {
        return err
    }

    // キャッシュを更新
    return rdb.Set(ctx, key, value, time.Hour).Err()
}

// Write-Behind
func updateDataWithCacheAsync(ctx context.Context, rdb *redis.Client, key, value string) error {
    // キャッシュを更新
    err := rdb.Set(ctx, key, value, time.Hour).Err()
    if err != nil {
        return err
    }

    // 非同期でDBを更新
    go func() {
        err := updateDB(key, value)
        if err != nil {
            // エラーハンドリング
        }
    }()

    return nil
}
```

### 5. キャッシュの無効化
```go
func invalidateCache(ctx context.Context, rdb *redis.Client, key string) error {
    return rdb.Del(ctx, key).Err()
}

// パターンマッチによる一括無効化
func invalidateCachePattern(ctx context.Context, rdb *redis.Client, pattern string) error {
    keys, err := rdb.Keys(ctx, pattern).Result()
    if err != nil {
        return err
    }

    if len(keys) > 0 {
        return rdb.Del(ctx, keys...).Err()
    }
    return nil
}
```

## ベストプラクティス

### 1. キャッシュの設計
- 適切なTTLの設定
- キャッシュサイズの制限
- メモリ使用量の監視

### 2. パフォーマンス最適化
- バッチ処理
- パイプライン
- コネクションプール

### 3. エラーハンドリング
- フォールバック戦略
- リトライメカニズム
- サーキットブレーカー

### 4. モニタリング
- キャッシュヒット率
- レイテンシ
- メモリ使用量 
