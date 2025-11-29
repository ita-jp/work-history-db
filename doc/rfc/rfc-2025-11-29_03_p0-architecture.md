# RFC-2025-11-29_03 P0 アーキテクチャ詳細（低コスト・高速開発版）

## 0. スコープ
- P0（〜1万MAU想定）の具体的な技術構成と実装方針を定義する。
- Vercel + Supabase + Next.js（TypeScript）を前提に、LLM呼び出しは非同期ジョブ化する。
- 目的: 初速を最大化しつつ、P1以降の移行を阻害しない最小構成を決める。

## 1. ゴールと制約
- ゴール: OJTログ蓄積、弱点可視化用テスト生成、学習推薦、求人マッチ支援のMVPを最短で立ち上げる。
- 非機能: 低コスト（無料枠/低額）、シンプル運用、週次〜日次の改善サイクルを回せること。
- 制約: プロダクト全体で TypeScript 統一、PostgreSQL 互換を維持、OIDC/JWT 前提で将来の認証差し替えを可能にする。

## 2. 全体構成（P0）
- フロント/SSR: Next.js (App Router) on Vercel。ISR/SSRを併用。
- BFF/API: Next.js Route Handler + Supabase Edge Functions（必要最低限）。tRPC もしくは REST を統一。
- データ: Supabase PostgreSQL。Prisma または Drizzle でスキーマ管理・マイグレーション。
- 認証: Supabase Auth（メール/ソーシャル）。JWTでフロントからAPIへ。Row Level Security (RLS) でデータ隔離。
- ストレージ: Supabase Storage（S3互換）。レジュメ/求人票/生成結果を保存、署名URLで配布。
- ジョブ: 簡易DBキュー + pg_cron + Edge Functions/Route Handler バッチで LLM生成や集計を非同期実行。
- LLM: OpenAI API を優先（gpt-4.x）。プロバイダ差し替え用のAdapter層を設計。
- 監視/品質: Vercel Analytics + Supabase ログ/モニタ + Sentry（フロント/バック）。CIは GitHub Actions で lint/test/build。

## 3. ドメイン/ユースケース（P0で実装する範囲）
- OJTログ: 日々の業務記録（タスク、役割、所感、使ったスキル）。
- テスト生成: OJTログを基にした弱点可視化テストのLLM生成（選択式/記述式シンプル版）。
- 学習推薦: テスト結果 + 目標ロールから、短い学習タスクを推薦（LLMまたは静的ルール）。
- 求人マッチ: アップロード求人票と業務履歴の一致度/不足スキル/改善ポイントの提示（LLM要約）。
- 基本UI: ダッシュボード、OJTログ入力/一覧、テスト結果表示、学習タスク一覧、求人アップロード/マッチ結果表示。

## 4. 初期データモデル（サマリ）
- users: id, email, profile (target_role, experience_years, interests)。
- work_logs: id, user_id, title, description, tags(skills), role, started_at, ended_at, outcome, learnings。
- tests: id, user_id, work_log_id, status(enum: queued/running/completed/failed), prompt_ref, score, weaknesses, created_at。
- test_items: id, test_id, question, choices(jsonb), correct_answer, user_answer, evaluation。
- learning_tasks: id, user_id, source(enum: llm/manual), title, description, priority, due, status。
- job_posts: id, user_id, title, raw_text, parsed(jsonb)。
- job_matches: id, job_post_id, score, strengths, gaps, recommended_actions, generated_resume_url。
- jobs_queue: id, type(enum: test_gen/job_match/report), payload(jsonb), status, attempts, run_at, last_error。
- audit_logs: id, user_id, action, entity, entity_id, metadata。

## 5. BFF/API 設計の方針
- 認証: Supabase Auth の JWT を Next.js Middleware で検証し、ユーザーコンテキストを BFF に注入。
- APIスタイル: tRPC（型安全）を推奨。RESTの場合も型は zod でスキーマバリデーション。
- RLS: 全テーブル user_id を持ち、Postgres RLS で user_id = auth.uid() を強制。管理用に service role を限定使用。
- エラーハンドリング: 共通エラーラッパを設け、LLMやキュー失敗を可観測に。

## 6. ジョブ/LLM 実行フロー（P0）
- 1) フロントから「テスト生成」「求人マッチ」をリクエスト → jobs_queue に enqueue。
- 2) pg_cron で数分おきにジョブポーラーが起動（Edge Function or Route Handler バッチ）。
- 3) ポーラーが status=queued を取得し、LLM APIを呼び出して結果をテーブルに保存。失敗はリトライし attempts をインクリメント。
- 4) UIはポーリング/SSEで status を監視し、完了時に結果を表示。
- 5) ジョブは idempotent（同じペイロードは再実行しても安全）に実装。

## 7. インフラとデプロイ
- フロント/SSR/BFF: Vercel。環境: Preview/Production。環境変数で Supabase/LLM キーを分離。
- データ/認証/ストレージ: Supabase プロジェクト1つ（dev/prodは別プロジェクト推奨）。
- Edge Functions: Supabase 上に配置。小さいジョブ/ポーラー/管理系に利用。
- CI/CD: GitHub Actions → Vercel Deploy。Supabase migration を CIで dry-run、本番は手動承認 or Deploy 時に apply。
- DNS/CDN: Vercel標準。独自ドメインは CNAME。静的アセットはVercel/CDN配信。

## 8. 運用・可観測性
- ログ: Supabase (Postgres/Edge) + Vercel logs。LLMエラー/キュー失敗は構造化ログで記録。
- トレース: 初期は不要。Sentry をフロント/バック両方に導入し、エラー率を監視。
- アラート: 重要クエリ失敗率/LLM失敗連続/キュー滞留で通知（Supabase + Sentry webhook）。
- バックアップ: Supabase 自動バックアップ依存。PIIカラムは暗号化を検討。

## 9. セキュリティ/プライバシー
- RLS必須、サービスロールの利用は Edge Functions など限定的に。
- ファイルは署名付きURLで期限付き配布。PII/秘匿情報は Storage と DB の両方でアクセス制御。
- Secrets は Vercel Env / Supabase Secrets に保存、レポに含めない。
- 監査: audit_logs に主要操作を記録（ビュー/更新/LLM生成要求など）。

## 10. コスト見積もり（目安）
- Vercel Hobby/Pro: 数十ドル以内（トラフィック次第）。
- Supabase Pro ベーシック: $25〜。無料枠から開始可だがDBサイズ/ログで早めに上げる。
- OpenAI: 生成量に依存。P0は試験的トークン使用＋レート制限でコスト上限を設定。
- Sentry/PostHog: 無料枠内で開始し、閾値超過時にプランアップ。

## 11. 将来の移行余地（P1への布石）
- Prisma/Drizzle スキーマを維持し、RDS/Aurora へエクスポート可能な形を保つ。
- DBキューを抽象化し、SQS/Kafka への差し替えを容易にする。
- Auth を JWT/OIDC 前提に実装し、Auth0/Cognito/Supabase self-host へ移行しやすく。
- Storage は S3互換クライアントを利用し、Supabase Storage → S3 へのコピーで移行可能。

## 12. 決定事項と未決事項
- 決定: P0は「Vercel + Supabase + Next.js + Prisma/Drizzle + LLM非同期ジョブ(DBキュー)」で開始する。
- 未決: tRPC vs REST の最終決定、Prisma vs Drizzle の選択、LLMプロバイダの正式選定（OpenAI優先だがコスト/品質比較を実施）。
