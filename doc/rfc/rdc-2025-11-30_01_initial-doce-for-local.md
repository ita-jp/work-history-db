# RDC-2025-11-30_01 ローカル開発セットアップ（Next.js → Vercel → Supabase 基本導線）

## 0. ゴール
- ローカルで Next.js を立ち上げ、Vercel にデプロイして実環境で動作確認できる。
- Supabase と接続し、ユーザーがログインしてレジュメ（フリーテキスト1項目）を登録できる最小実装まで持っていく。
- LLM/ジョブは後回し。まずは認証＋永続化の基本導線を固める。

## 1. 前提ソフトウェア
- Node.js 20 系（LTS）。`corepack enable` を実行。
- pnpm 10.24.0（ADR-0001）。`corepack prepare pnpm@10.24.0 --activate`
- Supabase CLI（ローカル接続検証用）。`brew install supabase/tap/supabase` など。
- Vercel CLI（デプロイ用）。`pnpm add -D vercel` または `npm i -g vercel`

## 2. Next.js プロジェクト初期化（ローカル）
1) リポジトリ取得/作業ディレクトリへ  
`git clone <repo>` / `cd work-history-db`

2) Node と pnpm 準備  
`corepack enable && corepack prepare pnpm@10.24.0 --activate`

3) Next.js アプリ作成（App Router / TypeScript / Tailwind 例）  
```
pnpm create next-app . --ts --app --tailwind --eslint --src-dir --import-alias "@/*"
```
（既に雛形がある場合はスキップ。必要に応じて`packageManager`フィールドを `pnpm@10.24.0` に揃える）

4) 依存インストール  
`pnpm install`

5) ローカル開発サーバ  
`pnpm dev` → http://localhost:3000 で表示を確認。

## 3. Vercel へのデプロイ（Nextのみ）
1) Vercel ログイン/プロジェクト紐付け  
`vercel login` → `vercel link` でリポジトリと接続。

2) Previewデプロイ  
`vercel deploy --prebuilt`（または `vercel deploy`）。Preview URL で画面確認。

3) Productionデプロイ  
`vercel deploy --prod`。基本動作（トップページ表示）を確認。

## 4. Supabase 接続の基本設定
1) Supabase プロジェクトを作成し、`Project URL` と `anon key` を取得。  
2) `.env.local` と Vercel の Project Env に設定  
```
NEXT_PUBLIC_SUPABASE_URL=<project-url>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>
```
   - ローカル検証を Supabase CLI でやる場合は `supabase start` 後に表示される URL/KEY を同様に設定。
3) SDK 追加  
`pnpm add @supabase/supabase-js`

## 5. レジュメ用テーブル（最小構成）
- Supabase SQL で作成（今後も使う前提のシンプル版）:
```
create table resumes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  resume_text text not null,
  created_at timestamptz not null default now()
);
alter table resumes enable row level security;
create policy "resume owner can read/write"
  on resumes for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```
- 必要に応じてインデックス: `create index on resumes (user_id);`

## 6. Next.js からの接続・画面確認
1) Supabase クライアント初期化を `lib/supabaseClient.ts` 等に定義。`NEXT_PUBLIC_...` を使う。  
2) 認証: Supabase Auth（magic link 等）でログインフォームを用意。ログイン後に `user.id` を取得。  
3) レジュメ登録 UI: フリーテキスト1項目のフォームを作成し、`insert into resumes` を呼ぶ（`user_id` を付与）。  
4) 一覧/単票表示: ログインユーザーの履歴を `select * from resumes order by created_at desc` で表示。  
5) 動作確認（ローカル→Vercel）:  
   - ローカル: ログイン → レジュメ登録 → 取得できること  
   - Vercel: 同じフローを Preview/Production で確認。環境変数が正しく設定されていることをチェック。

## 7. テスト/品質チェック（最小）
- Lint: `pnpm lint`
- 型チェック: `pnpm typecheck`
- （任意）簡易E2E: ログイン→レジュメ登録が通るかを Playwright などで将来追加。

## 8. LLM/ジョブについて
- 現段階では未実装でよい。後続で非同期ジョブ/SSE ポーリングを追加予定。

## 9. 未決/要調整
- 認証UIの方式（magic link / OAuth）をどれに固定するか。
- Prisma/Drizzle 採用とマイグレーションツールの決定。
- レジュメ項目を増やすタイミング、バリデーション方針。
