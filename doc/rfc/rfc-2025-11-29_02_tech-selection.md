# RFC-2025-11-29_02 技術選定と段階的アーキテクチャ

## 目的と条件
- RFC-2025-11-29_01（サービスコンセプト）を実現するための技術選定方針。
- 初期は低コスト・高速開発を最優先し、将来的に1,000万ユーザー規模へ耐える段階的なアーキテクチャを定義する。
- TypeScript を主軸に、移行容易性を重視した選定を行う。

## プロダクト特性と要求
- 主要機能: 業務ログ蓄積（OJT）、テスト自動生成、弱点可視化、学習推薦（Off-JT）、求人一致・レジュメ生成。
- トラフィック特性: 書き込みは日次・中量、読み込みは推薦/履歴参照・検索が多め。LLM呼び出しは非同期バッチ化したい。
- 非機能: 低コスト運用、個人情報保護（PII）、SLOに耐える信頼性、将来のマルチリージョン展開を阻害しない構成。

## 技術原則
- Managed-first / Serverless-first で初期コストを圧縮する。
- TypeScript 単一言語でフロント/バックを統一し、学習コストを下げる。
- PostgreSQL 互換を軸にし、ロックインを避けた移行経路を確保する。
- スキーマ/イベントファースト設計（Prisma/Drizzle + CDC想定）で、将来の分割とデータパイプライン拡張を容易にする。
- LLM呼び出しはジョブキュー経由で非同期化し、失敗リトライを標準化する。

## フェーズ別アーキテクチャ
### P0: クローズドβ〜初期リリース（〜1万MAU）
- フロント: Next.js (App Router) + TypeScript を Vercel にデプロイ。ISRで速度とSEOを両立。
- BaaS: Supabase（PostgreSQL + Auth + Storage + Edge Functions）を採用。Route Handler or Edge FunctionsでBFF/APIを実装。
- ORマッパー: Prisma または Drizzle でスキーマ管理し、マイグレーションを自動化。
- ジョブ: pg_cron + DBテーブルによる簡易ジョブキューで LLM生成・集計を非同期実行。
- LLM: OpenAI/Anthropic などのマネージドAPI。結果はDBに保存し、UIはポーリング or SSE。
- 監視/品質: Vercel Analytics + Supabase ログ/メトリクス + Sentry（フロント/バック）。CIはGitHub Actionsでlint/test/build。
- コスト理由: インフラはVercel無料枠/低額プラン + Supabase プロ/ベーシックのみで運用、運用工数も最小。

### P1: 一般公開〜成長期（〜50万MAU）
- バックエンド分離: BFF/APIを NestJS/Express をコンテナ化し、Fly.io/Render/ECS Fargate などのオートスケール基盤へ。フロントはVercel継続。
- DB: Supabase から AWS RDS PostgreSQL (Multi-AZ) へ移行。Prisma Migrationでスキーマ維持。ストレージは S3 + CloudFront へ。
- 認証: Auth0/Cognito もしくは Supabase Auth self-host。OIDC/JWT 前提の実装にし差し替えやすくする。
- キャッシュ: Redis (Upstash/ElastiCache) でセッション・推薦キャッシュ・ジョブデデュプ。
- 検索: Typesense/Algolia で業務ログ・求人全文検索とランキングを提供。
- ジョブ基盤: SQS + ワーカー（Fargate/Cloud Run Job/Fly Machines）で非同期処理を標準化。LLM/レポート生成をここへ移管。
- 監視/運用: OpenTelemetry + Grafana Cloud/Datadog。Feature Flag (LaunchDarkly 等) で段階的リリース。
- コスト抑制: コンテナはscale-to-zero可能なプラットフォームを選択し、RDSは最小クラス+自動ストレージ拡張で開始。

### P2: 1,000万ユーザー視野（〜1,000万MAU）
- フロント: Next.js を Vercel もしくは自前 ECS/EKS で水平スケール。CDNでISR/静的を配信。
- API: ステートレスAPIを ECS/EKS/Cloud Run で水平スケール。bounded context（履歴収集、学習推薦、求人マッチング、認証）ごとに分割。
- DB: Aurora PostgreSQL クラスター + リードレプリカ。OLTP/OLAP 分離で Snowflake/BigQuery/Redshift Serverless に CDC (Debezium + MSK/Kinesis) で配信。
- キュー/ストリーム: SQS + SNS もしくは Kafka/Kinesis でイベント駆動化。LLM生成・推薦計算をジョブ/Batch/EKSジョブへ。
- キャッシュ/検索: Redis Cluster (Multi-AZ) + OpenSearch/Typesense クラスタ。HotパスはRedis、全文/集計は検索基盤へ。
- 信頼性/運用: SLO/SLA設定、Otel/Datadog で可観測性、Blue-Green/Canary デプロイ。Feature Flag で段階的機能展開。
- セキュリティ/DR: PIIカラム暗号化(KMS)、RLS/ACL、監査ログ。マルチAZ→マルチリージョン冗長、RPO/RTOを設定。
- コスト最適化: Graviton/スポット活用、サーバレス優先、ワークロード別コスト可視化とアラート。

## 実装スタック案（P0基準）
- フロント: Next.js + TailwindCSS。ログイン/ダッシュボード/履歴・学習UI。
- バックエンド: Next.js Route Handler + Supabase Edge Functions (Node/Deno)。APIスタイルは tRPC または REST を統一。
- データ: Supabase PostgreSQL + Prisma/Drizzle。業務ログ、テスト、学習計画、求人・スキル差分をスキーマ管理。
- 認証/権限: Supabase Auth（メール/Social）。Row Level Security + RBAC でテナント隔離。
- ストレージ: Supabase Storage (S3互換) にレジュメ・求人票・生成ファイルを保存。署名URLで配布。
- LLM: OpenAI API（gpt-4.x）を優先。プロバイダ差し替え用のAdapter層を設計。
- 分析: Supabase Analytics + Vercel + PostHog（イベント計測）。初期は無料枠を活用。

## マイグレーション戦略
- スキーマ: Prisma/Drizzle でバージョン管理し、CIで dry-run → 本番 apply。PostgreSQL 互換を守り、Aurora/RDSへの移行を容易にする。
- 認証: OIDC/JWT 前提で実装し、Auth0/Cognito/Supabase Auth self-host へ差し替え可能にする。
- ストレージ: S3互換ライブラリを使用し、Supabase Storage → S3 移行をオブジェクトコピーで完結できる設計にする。
- ジョブ: DBキュー抽象を設け、ポーラーを DB → SQS/Kafka に差し替え可能にする。ジョブは idempotent に実装。
- LLM: 「ジョブ登録 → 非同期実行 → 結果保存 → UI更新」のパターンを標準化し、モデル/プロバイダ変更の影響を局所化。

## セキュリティとプライバシー
- 行レベルセキュリティ/ACLでユーザー毎にデータを隔離。署名付きURLで限定共有。
- 最小権限のAPIキー管理、Secretsは環境変数/Secret Manager で集中管理。
- 個人情報はカラムレベル暗号化を検討し、バックアップ/ログはマスキング。監査ログを保存しアクセス経路を明確化。

## 開発プロセス/DevOps
- CI: GitHub Actions で lint/test/build。P0では Vercel Preview と Supabase migration dry-run を回す。
- CD: main マージで Vercel デプロイ、Supabase migration apply を自動化。P1以降は Canary/Blue-Green を導入。
- IaC: P0は省略可。P1以降、Terraform/CDKTF で RDS/S3/SQS/Redis などを管理し、環境差分をなくす。

## まとめ
- 今は「Vercel + Supabase + Next.js」の最小構成でスピードと低コストを実現する。
- 成長に応じて「API分離 + RDS + Redis + SQS/Typesense」へ拡張し、運用コストを抑えつつ性能を上げる。
- 1,000万規模では「Auroraクラスタ + イベント駆動 + CDN/キャッシュ/検索 + OLAP分離」で耐久性と信頼性を確保する。
