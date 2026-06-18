# プロジェクトナレッジベース: サンプル商事コーポレートサイト

> これは記入例（サンプル）です。実在しない架空の受託案件「株式会社サンプル商事 コーポレートサイトリニューアル」を題材にしています。

- 作成日（この資料の情報時点）: 2026-06-16

> 凡例: `🖊 要記入` = 人が後で埋める箇所 / `⚠ 要確認` = 事実確認が必要な箇所

---

# ① 案件概要

## サイト概要

- 何のサイト / システムか: サンプル商事のコーポレートサイト（会社案内・採用・お知らせ）と、取引先向けの会員マイページ。
- 目的・背景: 旧サイト（WordPress）の老朽化に伴うリニューアル。更新作業を CMS で内製化したい、という要望が背景。

## 機能・ページ等

- トップ / 会社案内 / 事業紹介
  - 概要: 静的ページ。内容は microCMS から取得
- お知らせ一覧・詳細
  - 概要: microCMS で得意先が更新
- 採用情報
  - 概要: microCMS 管理
- お問い合わせフォーム
  - 概要: 送信内容を SendGrid でメール通知 + DB保存
  - 備考: 送信先は「環境変数」参照
- 会員マイページ
  - 概要: ログイン後に請求書PDFを閲覧
  - 備考: 認証は Laravel 側

## 案件体制

- ディレクター
  - 氏名: 佐藤 花子
  - 連絡先: sato@example.com
- エンジニア
  - 氏名: 田中 太郎
  - 連絡先: tanaka@example.com

## インフラ環境（社内 or 得意先）

- 管理主体: 得意先（サンプル商事）の AWS アカウント上に構築。日常運用は保守ベンダー（インフラ社）が代行。
- アカウント所有者: AWS = 得意先 / ドメイン `sample-shoji.example` = 得意先がお名前.comで管理 / microCMS = 社内アカウント
- 契約・支払い: AWS・ドメインは得意先請求。microCMS・SendGrid・Vercel は社内契約で得意先に再請求。

## 関連資料

- Wiki: https://wiki.example.com/sample-shoji
- 案件ドライブ: https://drive.example.com/sample-shoji/

---

# ② 開発環境・セットアップ

## リポジトリ構成

- `sample-shoji-front`（フロントエンド・Nuxt）
  - URL: https://github.com/example/sample-shoji-front
- `sample-shoji-api`（バックエンド・Laravel）
  - URL: https://github.com/example/sample-shoji-api

## 技術スタック

- フロントエンド: Nuxt 3 / TypeScript / Tailwind CSS
- バックエンド: Laravel 11 / PHP 8.3
- データベース: MySQL 8 (Amazon RDS)
- インフラ: AWS（Vercel 併用）
- その他: microCMS（ヘッドレスCMS）, SendGrid（メール）

## フロントエンド（デプロイ方法のみ）

- デプロイ方法 / ホスティング: Vercel（GitHub 連携。`main` push で本番、`develop` push でステージングへ自動デプロイ）

## バックエンド設定

- フレームワーク / 言語: Laravel 11 / PHP 8.3
- 起動 / マイグレーション: `php artisan serve` / `php artisan migrate`
- 主要なバッチ / ジョブ: 請求書PDFの夜間生成（`php artisan invoice:generate`、cron で 03:00）
- 注意点: 会員認証は Laravel Sanctum。フロントとは別ドメインなので CORS 設定（`config/cors.php`）に注意。

## インフラ設定

- 構成: フロント=Vercel / API=AWS ECS (Fargate) / DB=RDS MySQL / 静的ファイル=S3
- 管理方法: Terraform（`sample-shoji-api` リポジトリの `infra/` 配下）。適用は保守ベンダーが実施。
- ドメイン / SSL証明書: `sample-shoji.example` は得意先がお名前.comで管理。証明書は ACM で自動更新。
- DNS: API サブドメイン `api.sample-shoji.example` の Route53 設定は社内管理。変更時は得意先に連絡。

## 開発セットアップ手順

前提:
- Node 20, PHP 8.3, Composer, Docker（MySQL 用）

手順:
```bash
# --- フロントエンド ---
git clone git@github.com:example/sample-shoji-front.git
cd sample-shoji-front
npm install
cp .env.example .env.local   # microCMS のキー等は 1Password から
npm run dev                   # http://localhost:3000

# --- バックエンド ---
git clone git@github.com:example/sample-shoji-api.git
cd sample-shoji-api
composer install
cp .env.example .env
docker compose up -d          # MySQL
php artisan key:generate
php artisan migrate --seed
php artisan serve             # http://localhost:8000
```

## 環境変数

> ⚠ 値そのものはここに書かない。保管場所と取得方法を書く。

- `MICROCMS_API_KEY`
  - 用途: CMS コンテンツ取得
  - 保管場所 / 取得方法: 1Password「サンプル商事」ボールト
- `SENDGRID_API_KEY`
  - 用途: フォーム送信メール
  - 保管場所 / 取得方法: 1Password 同上
- `DB_PASSWORD`
  - 用途: DB接続
  - 保管場所 / 取得方法: 本番は AWS Secrets Manager `sample-shoji/db`
- `NEXT_PUBLIC_API_BASE`
  - 用途: API のベースURL
  - 保管場所 / 取得方法: `.env.example` 参照

## ブランチ運用

- 運用ルール: `main` = 本番、`develop` = ステージング、`feature/*` で作業して `develop` へ PR。
- PR / レビュー: PR は佐藤さん or 後任がレビュー。得意先確認が必要なものはステージングで確認後に `main` へ。

---

# ③ 運用・保守

## 環境URL一覧

- 本番
  - URL: https://sample-shoji.example
  - ホスティング: Vercel + AWS
  - Basic認証等: なし
- ステージング
  - URL: https://stg.sample-shoji.example
  - ホスティング: Vercel + AWS
  - Basic認証等: Basic認証あり（1Password参照）
- 開発
  - URL: localhost:3000 / :8000
  - ホスティング: ローカル
  - Basic認証等: なし

## デプロイ手順

- 本番デプロイ: `develop` で得意先確認 → `main` へマージで自動デプロイ（Vercel + GitHub Actions → ECS）。
- ステージングデプロイ: `develop` への push で自動。
- ロールバック手順: Vercel は管理画面で前デプロイに Promote。API は ECS で前タスク定義リビジョンに戻す。
- デプロイ時の注意: お知らせ等が反映されない場合は CloudFront のキャッシュ invalidate（`/*`）が必要なことがある。

## 定常作業（必要に応じて）

- microCMS の利用状況確認
  - 頻度: 随時
  - 手順 / 場所: プラン上限（API転送量）に注意。上限超過でお知らせが出なくなる
- 請求書バッチの稼働確認
  - 頻度: 月初
  - 手順 / 場所: CloudWatch Logs `/ecs/invoice` を確認
- ドメイン / 証明書期限
  - 頻度: 年1回
  - 手順 / 場所: ドメインは得意先管理。期限前にリマインド

## 既知の問題や注意事項等

- microCMS の API 転送量が無料枠に近い。アクセス増時は有料プランへ（得意先承認が必要）。⚠ 要確認: 現在の使用量。
- お問い合わせフォームの送信失敗時、ユーザーにエラーは出るが管理側に通知が飛ばない（`app/Http/Controllers/ContactController.php` 付近）。未対応。
- 会員マイページの請求書PDFは S3 に保存。古いものを消す仕組みが無く溜まり続ける（要対応）。
- お知らせ一覧はページネーション未実装。件数が増えると重くなる。

## トラブルシューティング

- 症状: お知らせを更新したのにサイトに出ない
  - 原因: ページキャッシュ / CDN キャッシュ
  - 対応: Vercel を再デプロイ、または CloudFront を invalidate
- 症状: 会員ログインできない
  - 原因: CORS / Sanctum セッション
  - 対応: API 側 `config/cors.php` と `SESSION_DOMAIN` を確認
- 症状: お問い合わせメールが届かない
  - 原因: SendGrid キー失効 or 送信上限
  - 対応: SendGrid ダッシュボードで状態確認、キーは 1Password
- 症状: 請求書PDFが生成されない
  - 原因: 夜間バッチ失敗
  - 対応: `/ecs/invoice` のログ確認、`php artisan invoice:generate` を手動実行
