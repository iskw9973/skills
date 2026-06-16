# 引き継ぎ資料: Orderbook API

> これは記入例（サンプル）です。実在しない架空のプロジェクト「Orderbook API」を題材にしています。

| 項目 | 内容 |
| --- | --- |
| 作成者 | 石川 海斗 (ishikawam.dev@gmail.com) |
| 作成日 | 2026-06-16 |
| 最終出社日 | 2026-06-30 |
| 後任 | 田中 太郎 (tanaka@example.com) |
| リポジトリ | https://github.com/example/orderbook-api |

> 凡例: `🖊 要記入` = 人が後で埋める箇所 / `⚠ 要確認` = 事実確認が必要な箇所

---

## 1. プロジェクト概要

- **何をするものか**: EC サイトの注文を受け付け、在庫引当と決済を行う REST API。
- **ビジネス上の役割 / なぜ存在するか**: フロント（Web/アプリ）と決済・在庫システムの間に立つ中核サービス。ここが落ちると注文が一切受けられない。
- **主なユーザー / 利用者**: 社内のフロントエンドチーム、モバイルアプリチーム。
- **現在のステータス**: 運用中（1日あたり約 12,000 注文）

## 2. 技術スタック / アーキテクチャ

- **言語 / ランタイム**: Python 3.12
- **主要フレームワーク・ライブラリ**: FastAPI, SQLAlchemy, Alembic, Celery
- **データストア**: PostgreSQL 15（注文データ）, Redis（キャッシュ + Celery ブローカー）
- **アーキテクチャ概要**:

```
[Frontend] --HTTP--> [Orderbook API (FastAPI)] --> [PostgreSQL]
                            |                           
                            |--> [Redis] <--> [Celery Worker] --> [決済サービス(Stripe)]
                            |                                  --> [在庫サービス(社内gRPC)]
```

## 3. リポジトリ構成

| ディレクトリ / ファイル | 役割 |
| --- | --- |
| `app/api/` | FastAPI のルーティング・エンドポイント定義 |
| `app/models/` | SQLAlchemy モデル |
| `app/services/` | ビジネスロジック（在庫引当・決済連携） |
| `app/tasks/` | Celery の非同期タスク |
| `migrations/` | Alembic マイグレーション |
| `tests/` | pytest テスト |
| `docker-compose.yml` | ローカル開発用の DB/Redis |

## 4. ローカル開発環境のセットアップ

前提:
- Docker / Docker Compose, Python 3.12, Poetry

手順:
```bash
# 1. 依存のインストール
poetry install
# 2. 環境変数の用意
cp .env.example .env   # 値の取得元は .env.example のコメント参照
# 3. DB/Redis 起動
docker compose up -d
# 4. マイグレーション適用
poetry run alembic upgrade head
# 5. 起動
poetry run uvicorn app.main:app --reload
```

- **必要な環境変数**: `DATABASE_URL`, `REDIS_URL`, `STRIPE_API_KEY`, `INVENTORY_GRPC_HOST`（値はセクション8参照）

## 5. ビルド・テスト・デプロイ

| 操作 | コマンド / 手順 |
| --- | --- |
| テスト | `poetry run pytest` |
| Lint / 静的解析 | `poetry run ruff check . && poetry run mypy app` |
| ローカル実行 | `poetry run uvicorn app.main:app --reload` |

- **CI/CD**: `.github/workflows/ci.yml`。PR で lint+test、`main` マージで Docker イメージを ECR に push し、ECS にデプロイ。
- **デプロイ手順**: `main` へのマージで自動。手動デプロイは `gh workflow run deploy.yml -f env=production`。
- **ロールバック手順**: ECS コンソールで 1 つ前のタスク定義リビジョンに更新。または `deploy.yml` に過去の image tag を指定して再実行。

## 6. 実行環境・インフラ

| 環境 | URL / 場所 | ホスティング | 備考 |
| --- | --- | --- | --- |
| 本番 | https://api.orderbook.example.com | AWS ECS Fargate (ap-northeast-1) | RDS PostgreSQL, ElastiCache |
| ステージング | https://stg.api.orderbook.example.com | AWS ECS Fargate | 本番と同構成の縮小版 |
| 開発 | localhost | docker-compose | |

- **インフラ管理方法**: Terraform（別リポジトリ `example/infra` の `orderbook/` 配下）
- **ドメイン / 証明書の管理**: Route53 + ACM。証明書は自動更新。

## 7. 外部サービス・依存・連携先

| サービス | 用途 | 管理コンソール / 連絡先 | アカウント所有者 |
| --- | --- | --- | --- |
| Stripe | 決済 | dashboard.stripe.com | 経理部 + 開発チーム共有 |
| 在庫サービス (社内) | 在庫引当 (gRPC) | 担当: 在庫チーム #inventory | 在庫チーム |
| Datadog | 監視 / APM | app.datadoghq.com | SRE チーム |

## 8. 設定とシークレットの管理場所

> ⚠ シークレットの値そのものはここに書かない。保管場所だけ。

| シークレット / 設定 | 保管場所 | 取得・更新方法 |
| --- | --- | --- |
| DB 認証情報 | AWS Secrets Manager `prod/orderbook/db` | ECS タスクに自動注入 |
| Stripe API キー | AWS Secrets Manager `prod/orderbook/stripe` | Stripe ダッシュボードで再発行可 |
| ローカル開発用の値 | 1Password「Orderbook 開発」ボールト | チームメンバーに共有依頼 |

## 9. 定期作業・運用タスク

| タスク | 頻度 | 自動/手動 | 手順 / 場所 |
| --- | --- | --- | --- |
| 注文データの月次集計 | 毎月1日 02:00 | 自動 | Celery Beat `app/tasks/monthly_report.py` |
| 失敗決済のリトライ確認 | 毎朝 | 手動 | Datadog ダッシュボード「Payments」を確認、要対応なら #orders へ |
| 依存ライブラリ更新 | 隔週 | 半自動 | Dependabot PR をレビューしてマージ |

- **監視 / アラート**: Datadog。アラートは PagerDuty 経由で SRE オンコールへ。
- **ログの場所**: CloudWatch Logs `/ecs/orderbook` + Datadog Logs

## 10. 既知の課題・技術的負債・落とし穴

- 在庫引当は**結果整合性**。決済成功後に在庫不足が判明するケースが稀にあり、`app/services/inventory.py` で補償処理しているが完全ではない（⚠ 設計見直し候補）。
- `app/services/payment.py:142` の Stripe リトライは固定回数。ネットワーク断が長いと取りこぼす。
- テストの一部が時刻依存でたまに落ちる（`tests/test_report.py`）。`freezegun` 導入で直したいが未着手。
- コード中の `TODO` / `FIXME` 抜粋:
  - `app/services/payment.py:142` — FIXME: exponential backoff にしたい
  - `app/api/orders.py:88` — TODO: ページネーションの上限を設定で持たせる

## 11. よくあるトラブルと対応（ランブック）

| 症状 | 原因 | 対応 |
| --- | --- | --- |
| 注文 API が 503 | DB コネクション枯渇 | ECS タスク数を一時増やす / 長時間クエリを Datadog で特定 |
| 決済だけ失敗が増える | Stripe 側障害 or キー失効 | status.stripe.com 確認、キーは Secrets Manager で確認 |
| Celery タスクが滞留 | Redis 接続不可 / ワーカー停止 | ElastiCache とワーカータスクの状態を確認、ワーカー再起動 |

## 12. 進行中の作業・TODO・今後の予定

- **進行中**: PR #312「在庫補償処理のリファクタ」レビュー中。田中さんに引き継ぎ済み。
- **次にやる予定**: 決済リトライの exponential backoff 化（上記 FIXME）。
- **将来的にやりたい**: 注文 API のレート制限導入、gRPC 在庫サービスのタイムアウト調整。

## 13. 関係者・連絡先

| 役割 | 氏名 | 連絡先 | 備考 |
| --- | --- | --- | --- |
| プロダクトオーナー | 佐藤 花子 | sato@example.com | 仕様の最終判断者 |
| 後任エンジニア | 田中 太郎 | tanaka@example.com | |
| インフラ / SRE | SRE チーム | #sre (Slack) | オンコール体制あり |
| 外部ベンダー | （なし） | | |

## 14. 関連ドキュメント・リンク集

- 設計ドキュメント: https://confluence.example.com/orderbook/design
- インフラリポジトリ: https://github.com/example/infra
- Datadog ダッシュボード: https://app.datadoghq.com/dashboard/orderbook
- Slack チャンネル: #orders / #orderbook-dev
