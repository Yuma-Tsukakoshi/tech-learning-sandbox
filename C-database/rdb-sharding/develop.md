# RDBシャーディング開発ガイド

## 開発環境のセットアップ

### 必要条件
- Docker Desktop
- PostgreSQL 15.x
- Go 1.21以上

### セットアップ手順
1. シャードDBの起動
```bash
docker-compose up -d
```

2. シャードの初期化
```bash
./scripts/init-shards.sh
```

## アーキテクチャ

### コンポーネント
- シャードマネージャー
- シャードDB（複数）
- プロキシレイヤー

### 技術スタック
- PostgreSQL
- Go
- Docker

## 開発フロー

### シャーディング設定
```go
// シャーディング設定
type ShardConfig struct {
    ShardCount    int
    ShardKey      string
    ShardStrategy string
}

// シャードマネージャーの初期化
type ShardManager struct {
    config     ShardConfig
    connections []*sql.DB
}

func NewShardManager(config ShardConfig) (*ShardManager, error) {
    manager := &ShardManager{
        config: config,
    }
    
    // シャードDBへの接続を初期化
    for i := 0; i < config.ShardCount; i++ {
        conn, err := sql.Open("postgres", fmt.Sprintf(
            "host=localhost port=543%d user=postgres password=postgres dbname=shard_%d sslmode=disable",
            i, i,
        ))
        if err != nil {
            return nil, err
        }
        manager.connections = append(manager.connections, conn)
    }
    
    return manager, nil
}
```

### シャード選択ロジック
```go
// シャード選択ロジック
func (m *ShardManager) GetShard(key interface{}) (*sql.DB, error) {
    var shardIndex int
    
    switch m.config.ShardStrategy {
    case "hash":
        // ハッシュベースのシャード選択
        hash := fnv.New32a()
        hash.Write([]byte(fmt.Sprintf("%v", key)))
        shardIndex = int(hash.Sum32()) % m.config.ShardCount
        
    case "range":
        // レンジベースのシャード選択
        if num, ok := key.(int); ok {
            shardIndex = num % m.config.ShardCount
        } else {
            return nil, fmt.Errorf("invalid key type for range sharding")
        }
        
    default:
        return nil, fmt.Errorf("unknown sharding strategy")
    }
    
    return m.connections[shardIndex], nil
}
```

### データ操作
```go
// データの挿入
func (m *ShardManager) Insert(table string, data map[string]interface{}) error {
    shard, err := m.GetShard(data[m.config.ShardKey])
    if err != nil {
        return err
    }
    
    // カラムと値の準備
    columns := make([]string, 0, len(data))
    values := make([]interface{}, 0, len(data))
    placeholders := make([]string, 0, len(data))
    
    for col, val := range data {
        columns = append(columns, col)
        values = append(values, val)
        placeholders = append(placeholders, "?")
    }
    
    // クエリの構築と実行
    query := fmt.Sprintf(
        "INSERT INTO %s (%s) VALUES (%s)",
        table,
        strings.Join(columns, ", "),
        strings.Join(placeholders, ", "),
    )
    
    _, err = shard.Exec(query, values...)
    return err
}

// データの検索
func (m *ShardManager) Query(table string, shardKey interface{}, conditions map[string]interface{}) ([]map[string]interface{}, error) {
    shard, err := m.GetShard(shardKey)
    if err != nil {
        return nil, err
    }
    
    // WHERE句の構築
    whereClause := make([]string, 0, len(conditions))
    values := make([]interface{}, 0, len(conditions))
    
    for col, val := range conditions {
        whereClause = append(whereClause, fmt.Sprintf("%s = ?", col))
        values = append(values, val)
    }
    
    // クエリの構築と実行
    query := fmt.Sprintf(
        "SELECT * FROM %s WHERE %s",
        table,
        strings.Join(whereClause, " AND "),
    )
    
    rows, err := shard.Query(query, values...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // 結果の取得
    var results []map[string]interface{}
    columns, err := rows.Columns()
    if err != nil {
        return nil, err
    }
    
    for rows.Next() {
        row := make(map[string]interface{})
        values := make([]interface{}, len(columns))
        valuePtrs := make([]interface{}, len(columns))
        
        for i := range columns {
            valuePtrs[i] = &values[i]
        }
        
        if err := rows.Scan(valuePtrs...); err != nil {
            return nil, err
        }
        
        for i, col := range columns {
            row[col] = values[i]
        }
        
        results = append(results, row)
    }
    
    return results, nil
}
```

## パフォーマンス最適化

### シャーディング戦略
- ハッシュベース
- レンジベース
- ディレクトリベース

### クエリ最適化
- シャードキーの選択
- インデックスの設計
- バッチ処理

## バックアップと復旧

### バックアップ戦略
- シャードごとのバックアップ
- ポイントインタイムリカバリ
- レプリケーション

### 復旧手順
1. シャードのリストア
2. データの整合性確認
3. レプリケーションの再同期

## セキュリティ

### アクセス制御
- シャードごとの権限設定
- 接続の暗号化
- 監査ログ

### データ保護
- データの暗号化
- バックアップの暗号化
- アクセスログ

## モニタリング

### シャードモニタリング
- シャードの健全性
- データ分散
- パフォーマンスメトリクス

### リソースモニタリング
- CPU使用率
- メモリ使用量
- ディスク使用量

## トラブルシューティング

### よくある問題
1. シャードの不均衡
2. パフォーマンスの低下
3. データの整合性

### 解決方法
- シャードの再バランス
- インデックスの最適化
- データの再同期

## スケーリング

### 水平スケーリング
- シャードの追加
- データの再分散
- レプリカの設定

### 垂直スケーリング
- リソースの増強
- パフォーマンスチューニング
- キャッシュの最適化 
