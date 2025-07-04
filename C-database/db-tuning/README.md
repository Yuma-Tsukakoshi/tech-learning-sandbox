# DBのパフォーマンスチューニング

## 概要
このディレクトリでは、データベースのパフォーマンスチューニングの実装と理解を深めます。

## 学習目標
- スロークエリの特定と改善
- インデックスの適切な設計と実装
- 実行計画の理解と最適化

## 実装の流れ
1. スロークエリ（例: N+1問題）を意図的に実装
2. `EXPLAIN`で実行計画を確認
3. クエリの改善やINDEXの追加、ORMの利用法を修正する

## 完成の定義
- `EXPLAIN`の結果が改善され、アプリケーションのレスポンスタイムが短縮される
- インデックスが適切に設計・実装されている
- N+1問題などの一般的なパフォーマンス問題が解決されている

## 使用技術
- SQL
- Go
- ORM（GORMなど）
- PostgreSQL/MySQL 
