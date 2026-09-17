# CLAUDE.md — ABeam Skills Registry

AIエージェント用スキルを社内で共有・管理するプラットフォーム。FastAPI monolith + MCP ゲートウェイ + Vanilla JS フロントエンド。

## リポジトリ構成 — legacy（本番稼働中）と backend/（書き換え中）は別物

このリポジトリには**2つの独立した実装**が同居している。混同して片方のディレクトリにもう片方の
規約でファイルを追加しないこと。

- **legacy（現行・本番稼働中）**: ルート直下の `app/`・`scripts/`・`migrations/`・`tests/`・`cli/`、
  ルート `pyproject.toml`/`requirements*.txt`/`alembic.ini`。本 CLAUDE.md の「アーキテクチャ（現状）」
  以下はすべてこの legacy を指す。`scripts/*.py` は legacy 用の運用・データ移行スクリプト
  （`app.database`/`app.services` を直接 import）で、`backend/` とは無関係。
- **backend/（AINPF Clean Architecture 書き換え・issue #1460・進行中）**: `backend/app/`
  配下に `domain/application/infrastructure/presentation/core` の4層。独自の `pyproject.toml`・
  `.venv`・`alembic.ini`・テストを持ち、legacy とは依存を共有しない。規約は
  [backend/CLAUDE.md](backend/CLAUDE.md) を読むこと。まだ骨組み段階で legacy を置き換えていない
  （本番は今も legacy が動いている）。
- 新規の運用スクリプト・移行スクリプトは legacy 向けなら `scripts/`、backend 向けの何かが必要な
  場合は `backend/` 配下に置く — ルート `app/` と `backend/app/` を混同して同じ役割のファイルを
  二重に作らないこと。

## アーキテクチャ（現状 = legacy）

- **バックエンド**: FastAPI 0.115 + SQLAlchemy 2.0、エントリは `app/main.py`（`app` オブジェクト）。
- **DB**: 全環境 PostgreSQL（SQLite は廃止済み）。ローカルは docker-compose（`docker compose up -d db-test`、既定 `localhost:5433`）、dev/本番は Azure Database for PostgreSQL Flexible Server（[ADR-0001](docs/adr/0001-azure-container-apps-migration.md)）。`DATABASE_URL` は `postgresql+psycopg2://…` 必須（`app/database.py`）。マイグレーションは Alembic（`alembic.ini` / `migrations/`）。
- **ストレージ**: スキルファイルは `storage_root`（既定 `./storage/skills`）配下にディレクトリ構造で保管（`{skill_id}/snapshots/{ver}-{hash}/`）。`app/services/storage_service.py` が zip 展開・rglob・atomic rename を多用。
- **MCP**: `app/mcp_server.py`（FastMCP）。`/mcp` にマウント。AIエージェントが検索・取得に使う。
- **フロントエンド**: `app/static/*.html` + Vanilla JS（ビルドステップなし）。ページルーティングは `main.py` の `PAGE_ROUTES`。
- **CLI**: `cli/index.js`（npx `abeam-skills`）。`SKILLS_SERVER_URL` を env 読み。
- **設定**: `app/config.py`（pydantic-settings、`.env` 読み）。
- **LLM**: `app/services/llm_review_service.py` が Azure OpenAI SDK 対応。

ディレクトリ: `app/routers/`（health/skills/reviews/ratings/audit_logs/admin）、`app/services/`（storage/llm_review/audit/search/validator/skill/review）、`tests/`（pytest）。

## 開発コマンド

```bash
# 依存インストール（このプロジェクトは requirements.txt 管理。グローバル uv 共有 venv とは別）
pip install -r requirements.txt -r requirements-dev.txt   # 必要に応じ venv 内で

# 起動（開発）
uvicorn app.main:app --reload

# マイグレーション
alembic upgrade head

# テスト（pytest。TESTING=1 で lifespan の init をスキップ）
pytest
pytest --cov=app --cov-report=term-missing
```

> 注: グローバル CLAUDE.md の「Python は `~/.claude/skills` の uv プロジェクトを既定」ルールは**一時スクリプト用**。本プロジェクト本体の依存は `requirements.txt` で管理し、混在させない。

## 規約・注意点

- **テスト**: 新機能・バグ修正は TDD（RED→GREEN→REFACTOR）。pytest、カバレッジ 80% 目標。`tests/conftest.py` のフィクスチャを使う。
- **シークレット**: ハードコード禁止。`config.py` の `admin_password` は既定値なし（空文字列、環境変数 `ADMIN_PASSWORD` 未設定時は管理者操作を全拒否）。本番は必ず環境変数上書き（Azure 化で Key Vault へ移管予定）。`.env` はコミットしない。
- **イミュータブル志向 / 小さいファイル**: 関数 <50行・ファイル <800行目安。
- **DB セッション**: `app/database.py` の `SessionLocal` を `with` で使う。
- **静的 vs 動的ルート**: `main.py` で静的パス（`/skills/new`）を動的パス（`/skills/{id}`）より先に登録する規則を守る。

## ドキュメント運用

- アーキ意思決定 → `docs/adr/NNNN-kebab.md`（MADR、本体）+ `docs/adr/README.md` 索引 + ai-shared-memory にポインタ。
- 主要変更 → `CHANGELOG.md`（逆時系列、トップに追記）。
- ユーザー向け主要変更 → `app/data/whats_new.json` の先頭にもユーザー向け文言で 1 エントリ（`{date, title, body, link?}`）を追記（What's New 配信・[ADR-0019](docs/adr/0019-whats-new-feed.md)）。CHANGELOG 追記と同時に行う。
- 設計詳細 → `docs/`（`azure-target-architecture.md`, `deploy-azure-containerapps.md`, `azure-naming-tagging-conventions.md`, `ABeam-Design-System.md`, `enterprise_agent_skills_hub_mvp_spec.md`）。参考資料は `docs/reference/`。
- `/docops sync|changelog|adr|memory-audit|claudemd-check` で運用。

## Azure 移行（進行中）

ターゲット構成は [ADR-0001](docs/adr/0001-azure-container-apps-migration.md) / [azure-target-architecture.md](docs/azure-target-architecture.md) に確定。Container Apps + ACR + PostgreSQL Flexible + Azure Files + Key Vault + Entra ID 全経路認証 + internal ingress。足回りのコード改修（Dockerfile・config の DB/シークレット・mcp_server の allowed_hosts 環境変数化・cli の Bearer 送信・health の live/ready 分離）は**実装済み**（詳細は azure-target-architecture.md §7）。残るのはインフラ側の展開・結線作業（各 ADR 参照）。
