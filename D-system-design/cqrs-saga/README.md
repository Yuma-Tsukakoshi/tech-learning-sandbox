# CQRS & Saga Pattern

## 概要
このディレクトリでは、CQRS（Command Query Responsibility Segregation）とSaga Patternの実装と理解を深めます。

## 学習目標
- CQRSパターンの設計と実装
- Saga Patternの設計と実装
- 分散トランザクションの管理

## 実装の流れ
1. 書き込み（Command）用のAPIと、読み取り（Query）用のAPIを分離して実装
2. （Saga）マイクロサービス環境で、一連の処理が失敗した際に、補償トランザクション（処理の取り消し）が実行されるロジックを実装する

## 完成の定義
- 書き込み処理と読み取り処理で、参照するデータモデルやDBが分離されている
- Sagaでは、処理失敗時に補償イベントが発行される
- 分散トランザクションが適切に管理されている

## 使用技術
- Go
- メッセージキュー
- Docker
- Docker Compose 
