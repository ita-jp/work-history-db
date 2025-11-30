# ADR-0001 Node package manager

- Status: Accepted
- Date: 2025-11-30

## Context
- プロジェクトは Node.js 20 系を前提にし、フロント/バック/ジョブを TypeScript で統一する。
- パッケージマネージャは npm/yarn/pnpm の選択肢があるが、速度・ディスク効率・依存解決の厳密性に差がある。
- CI/開発環境の差異を減らし、lockfile を安定維持できるツールを採用したい。

## Decision
- パッケージマネージャは pnpm を採用する。
- `corepack enable && corepack prepare pnpm@10.24.0 --activate` を初期セットアップに含め、全員が同一バージョンの pnpm を使う（バージョンは意図的に bump するまで固定）。

## Rationale
- 共有ストア＋ハードリンクによりインストールが高速で、ディスク使用量が少ない。
- `node_modules` をフラット化しないため、未宣言依存の混入を防ぎ、依存解決が厳密になる。
- npm/yarn 互換のため移行コストが低く、将来 npm へ戻す場合も lockfile 以外の変更は少ない。
- Corepack 経由でバージョン固定を強制でき、CI/ローカル差分を抑制できる。`latest` にはせず、意図的に指定したバージョンを利用する。

## Consequences
- 開発者は pnpm を前提に `pnpm install`, `pnpm dev`, `pnpm lint` 等を実行する。
- npm/yarn を併用すると lockfile が競合するため、基本は禁止（必要時は明示して使い分ける）。
- Corepack が無効な環境では `corepack enable` を事前に実行する必要がある。

## How to check/bump pnpm version
- 最新版の確認: `npm view pnpm version` または `corepack install pnpm@latest --dry-run` で取得できる。
- バージョンを上げるときは、リリースノート確認 → ローカル/CIで試す → `corepack prepare pnpm@<new> --activate` と `packageManager` フィールドを同じバージョンに更新する、の順で行う。
