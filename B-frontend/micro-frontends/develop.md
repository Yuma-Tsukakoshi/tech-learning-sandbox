# マイクロフロントエンド開発ガイド

## 開発環境のセットアップ

### 必要条件
- Node.js 18.x以上
- npm 9.x以上
- Docker Desktop

### セットアップ手順
1. リポジトリのクローン
```bash
git clone [repository-url]
cd micro-frontends
```

2. 依存関係のインストール
```bash
npm install
```

3. 開発サーバーの起動
```bash
npm run dev
```

## アーキテクチャ

### 構成要素
- ホストアプリケーション（Next.js）
- マイクロフロントエンド（MFE）
  - MFE1: ヘッダー部分
  - MFE2: ボディ部分

### 技術スタック
- Next.js
- Module Federation
- React
- TypeScript

## 開発フロー

### ホストアプリケーションの設定
```typescript
// next.config.js
const { NextFederationPlugin } = require('@module-federation/nextjs-mf');

module.exports = {
  webpack(config, options) {
    config.plugins.push(
      new NextFederationPlugin({
        name: 'host',
        remotes: {
          header: 'header@http://localhost:3001/remoteEntry.js',
          body: 'body@http://localhost:3002/remoteEntry.js',
        },
        shared: {
          react: {
            singleton: true,
            requiredVersion: false,
          },
        },
      })
    );
    return config;
  },
};
```

### マイクロフロントエンドの設定
```typescript
// next.config.js (MFE側)
const { NextFederationPlugin } = require('@module-federation/nextjs-mf');

module.exports = {
  webpack(config, options) {
    config.plugins.push(
      new NextFederationPlugin({
        name: 'header', // または 'body'
        filename: 'remoteEntry.js',
        exposes: {
          './Header': './components/Header', // または './Body': './components/Body'
        },
        shared: {
          react: {
            singleton: true,
            requiredVersion: false,
          },
        },
      })
    );
    return config;
  },
};
```

### コンポーネントの実装
```typescript
// components/Header.tsx
import React from 'react';

const Header: React.FC = () => {
  return (
    <header>
      <h1>マイクロフロントエンド デモ</h1>
    </header>
  );
};

export default Header;

// components/Body.tsx
import React from 'react';

const Body: React.FC = () => {
  return (
    <main>
      <p>これは別のマイクロフロントエンドです。</p>
    </main>
  );
};

export default Body;
```

## パフォーマンス最適化

### 推奨事項
- コード分割
- 遅延ロード
- キャッシュ戦略

## テスト

### 単体テスト
```bash
npm run test
```

### E2Eテスト
```bash
npm run test:e2e
```

## トラブルシューティング

### よくある問題
1. Module Federationの接続エラー
2. ビルド時の依存関係の問題
3. ホットリロードの問題

### 解決方法
- キャッシュのクリア
- 依存関係の再インストール
- 開発サーバーの再起動

## セキュリティ

### 考慮事項
- クロスオリジンリソース共有（CORS）
- コンテンツセキュリティポリシー（CSP）
- 認証・認可
