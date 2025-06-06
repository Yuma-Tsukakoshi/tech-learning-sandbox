# Kubernetes

## 概要
このディレクトリでは、Kubernetesを使用したコンテナオーケストレーションの実装と理解を深めます。

## 学習目標
- Kubernetesの基本的な概念の理解
- マニフェストファイルの作成
- デプロイメントとサービスの管理

## 実装の流れ
1. 上記コンテナをKubernetes上で動かすためのマニフェストファイル（Deployment, Service）を作成
2. MinikubeなどのローカルK8s環境に`kubectl apply`でデプロイする
3. スケーリングとロールアウトの実装

## 完成の定義
- K8sクラスタ経由でアプリケーションにアクセスできる
- スケーリングが適切に機能する
- ロールアウトが適切に実装されている

## 使用技術
- Kubernetes
- kubectl
- YAML
- Minikube 
