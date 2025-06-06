# データベースチューニング開発ガイド

## 開発環境のセットアップ

### 必要条件
- Docker Desktop
- PostgreSQL 15.x
- Go 1.21以上

### セットアップ手順
1. データベースの起動
```bash
docker-compose up -d
```

2. テストデータの投入
```bash
./scripts/load-test-data.sh
```

## アーキテクチャ

### コンポーネント
- データベースサーバー
- アプリケーションサーバー
- モニタリングツール

### 技術スタック
- PostgreSQL
- Go
- GORM
- Docker

## 開発フロー

### スロークエリの特定
```go
// スロークエリのログ設定
type SlowQueryLogger struct {
    Threshold time.Duration
    Logger    *log.Logger
}

func (l *SlowQueryLogger) LogQuery(query string, duration time.Duration) {
    if duration > l.Threshold {
        l.Logger.Printf("Slow query detected (%.2fs): %s", duration.Seconds(), query)
    }
}

// クエリの実行と計測
func (db *DB) ExecuteQuery(query string, args ...interface{}) ([]map[string]interface{}, error) {
    start := time.Now()
    
    rows, err := db.Query(query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    duration := time.Since(start)
    db.slowQueryLogger.LogQuery(query, duration)
    
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

### インデックスの設計と実装
```go
// インデックスの作成
func (db *DB) CreateIndex(table, column string) error {
    indexName := fmt.Sprintf("idx_%s_%s", table, column)
    query := fmt.Sprintf("CREATE INDEX IF NOT EXISTS %s ON %s (%s)", indexName, table, column)
    
    _, err := db.Exec(query)
    return err
}

// インデックスの分析
func (db *DB) AnalyzeIndexUsage() ([]map[string]interface{}, error) {
    query := `
        SELECT
            schemaname,
            tablename,
            indexname,
            idx_scan,
            idx_tup_read,
            idx_tup_fetch
        FROM
            pg_stat_user_indexes
        ORDER BY
            idx_scan DESC
    `
    
    return db.ExecuteQuery(query)
}
```

### 実行計画の分析
```go
// 実行計画の取得
func (db *DB) GetExecutionPlan(query string, args ...interface{}) (string, error) {
    explainQuery := fmt.Sprintf("EXPLAIN ANALYZE %s", query)
    
    rows, err := db.Query(explainQuery, args...)
    if err != nil {
        return "", err
    }
    defer rows.Close()
    
    var plan strings.Builder
    for rows.Next() {
        var line string
        if err := rows.Scan(&line); err != nil {
            return "", err
        }
        plan.WriteString(line + "\n")
    }
    
    return plan.String(), nil
}

// 実行計画の分析
func (db *DB) AnalyzeQueryPerformance(query string, args ...interface{}) (*QueryAnalysis, error) {
    plan, err := db.GetExecutionPlan(query, args...)
    if err != nil {
        return nil, err
    }
    
    analysis := &QueryAnalysis{
        Plan: plan,
    }
    
    // 実行計画の解析
    if strings.Contains(plan, "Seq Scan") {
        analysis.Suggestions = append(analysis.Suggestions, "シーケンシャルスキャンが検出されました。インデックスの追加を検討してください。")
    }
    
    if strings.Contains(plan, "Nested Loop") {
        analysis.Suggestions = append(analysis.Suggestions, "ネステッドループが検出されました。結合条件の最適化を検討してください。")
    }
    
    return analysis, nil
}
```

## パフォーマンス最適化

### クエリ最適化
- インデックスの活用
- 結合の最適化
- サブクエリの最適化

### 設定最適化
- メモリ設定
- ディスクI/O設定
- 接続設定

## モニタリング

### パフォーマンスモニタリング
- クエリ実行時間
- リソース使用率
- キャッシュヒット率

### リソースモニタリング
- CPU使用率
- メモリ使用量
- ディスクI/O

## トラブルシューティング

### よくある問題
1. スロークエリ
2. メモリ不足
3. ディスクI/Oのボトルネック

### 解決方法
- クエリの最適化
- インデックスの追加
- 設定の調整

## メンテナンス

### 定期的なメンテナンス
- バキューム
- インデックスの再構築
- 統計情報の更新

### バックアップと復旧
- バックアップ戦略
- 復旧手順
- テスト復旧

## セキュリティ

### アクセス制御
- ユーザー権限
- ロールベースアクセス制御
- ネットワークセキュリティ

### 監査
- アクセスログ
- 変更履歴
- セキュリティ監査

## ドキュメント

### 設定ドキュメント
- パラメータ設定
- チューニング履歴
- 変更管理

### 運用マニュアル
- 監視手順
- バックアップ手順
- 復旧手順 
