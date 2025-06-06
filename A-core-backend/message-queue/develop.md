# メッセージキュー開発ガイド

## 開発環境のセットアップ

### 必要条件
- Docker Desktop
- RabbitMQ 3.12以上
- Node.js 18.x以上（アプリケーション開発用）

### セットアップ手順
1. RabbitMQの起動
```bash
docker-compose up -d
```

2. 管理画面へのアクセス
```
http://localhost:15672
デフォルトユーザー: guest
デフォルトパスワード: guest
```

## アーキテクチャ

### コンポーネント
- プロデューサー（メッセージ送信者）
- コンシューマー（メッセージ受信者）
- エクスチェンジ
- キュー
- バインディング

### 技術スタック
- RabbitMQ
- amqplib（Node.js用クライアント）
- Docker

## 開発フロー

### プロデューサーの実装
```javascript
const amqp = require('amqplib');

async function publishMessage() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  const queue = 'task_queue';
  const msg = 'Hello World!';
  
  channel.assertQueue(queue, {
    durable: true
  });
  
  channel.sendToQueue(queue, Buffer.from(msg));
  console.log(" [x] Sent %s", msg);
}
```

### コンシューマーの実装
```javascript
const amqp = require('amqplib');

async function consumeMessage() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  const queue = 'task_queue';
  
  channel.assertQueue(queue, {
    durable: true
  });
  
  channel.consume(queue, (msg) => {
    console.log(" [x] Received %s", msg.content.toString());
  }, {
    noAck: true
  });
}
```

## パフォーマンス最適化

### メッセージの永続化
- キューとメッセージの永続化設定
- 確認応答（Acknowledgment）の使用
- プリフェッチ数の最適化

### スケーリング
- クラスタリング
- シャーディング
- 負荷分散

## モニタリング

### メトリクス
- メッセージ処理速度
- キュー長
- メモリ使用量
- ディスク使用量

### アラート設定
- キュー長の閾値
- メモリ使用率
- エラー率

## トラブルシューティング

### よくある問題
1. メッセージの損失
2. パフォーマンスの低下
3. メモリ使用量の増加

### 解決方法
- 永続化の確認
- プリフェッチ数の調整
- リソースの増強

## セキュリティ

### 認証と認可
- ユーザー認証
- 仮想ホストの分離
- アクセス制御

### データ保護
- TLS/SSL暗号化
- メッセージの暗号化
- 監査ログ

## 運用管理

### バックアップ
- 設定のバックアップ
- メッセージのバックアップ
- クラスタ設定のバックアップ

### メンテナンス
- キューのクリーンアップ
- ログローテーション
- パフォーマンスチューニング

## ドキュメント

### 設計ドキュメント
- アーキテクチャ図
- フロー図
- エラーハンドリング

### 運用マニュアル
- 監視手順
- バックアップ手順
- 復旧手順 
