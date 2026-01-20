# Cloudflare Pages vs Workers - デプロイガイド

## 結論：Cloudflare Pages を推奨

このプロジェクトでは **Cloudflare Pages** を使用することを強く推奨します。

## なぜ Cloudflare Pages なのか？

### Cloudflare Pages の特徴

```
┌─────────────────────────────────────────────┐
│         Cloudflare Pages                    │
├─────────────────────────────────────────────┤
│  フロントエンド (Next.js Static Export)     │
│  ├─ HTML, CSS, JavaScript                   │
│  ├─ 静的アセット                            │
│  └─ クライアントサイド React               │
├─────────────────────────────────────────────┤
│  Pages Functions (サーバーレス)             │
│  ├─ /functions/api/reports.ts              │
│  ├─ /functions/api/upload.ts               │
│  └─ Lark API との通信                       │
└─────────────────────────────────────────────┘
         ↓ 全て自動デプロイ
    GitHub リポジトリと連携
```

### ✅ Cloudflare Pages の利点

#### 1. **GitHub 統合が超簡単**
```bash
# 必要な操作
1. GitHub リポジトリを作成
2. Cloudflare ダッシュボードで「Connect to Git」をクリック
3. リポジトリを選択
→ 完了！以降は git push で自動デプロイ
```

#### 2. **フロントエンド + バックエンドを一緒にデプロイ**
```
プロジェクト構成:
nippo_demo/
├── app/                    # Next.js フロントエンド
├── public/                 # 静的ファイル
├── functions/              # Pages Functions (バックエンド)
│   ├── api/
│   │   ├── reports.ts      # Lark API ラッパー
│   │   ├── upload.ts       # R2 アップロード
│   │   └── auth.ts         # 認証処理
│   └── _middleware.ts      # グローバルミドルウェア
└── package.json

→ これ全体が1つの git push でデプロイされる
```

#### 3. **無料枠が充実**
```
Cloudflare Pages 無料プラン:
✓ 無制限のリクエスト
✓ 無制限の帯域幅
✓ 500 ビルド/月
✓ Pages Functions: 100,000 リクエスト/日
✓ プレビューデプロイ無制限
```

#### 4. **プレビューデプロイ自動生成**
```bash
# PR を作成すると...
git checkout -b feature/new-form
git push origin feature/new-form
# → GitHub で PR 作成

# Cloudflare が自動で生成:
Preview URL: https://abc123.nippo-demo.pages.dev
                      ↑
            PR ごとに一意の URL
```

#### 5. **設定が超シンプル**
```yaml
# Cloudflare ダッシュボードでの設定例
Framework preset: Next.js
Build command: npm run build
Build output directory: out
Root directory: /

# 環境変数
LARK_APP_ID=cli_xxxxx
LARK_APP_SECRET=xxxxx
R2_BUCKET_NAME=nippo-attachments
```

---

## Cloudflare Workers との違い

### Cloudflare Workers の特徴

```
┌─────────────────────────────────────┐
│      Cloudflare Workers             │
├─────────────────────────────────────┤
│  単独のサーバーレス関数              │
│  ├─ API エンドポイントのみ          │
│  ├─ フロントエンドは別途ホスティング │
│  └─ wrangler CLI で個別デプロイ     │
└─────────────────────────────────────┘
```

### ❌ Workers が不向きな理由（このプロジェクトでは）

#### 1. **フロントエンドを別でホスティング必要**
```
Workers でデプロイすると:
├── フロントエンド → Cloudflare Pages や Vercel に別途デプロイ
└── バックエンド API → Workers にデプロイ

→ 管理が2箇所に分かれる（複雑）
```

#### 2. **GitHub 連携が手動**
```bash
# Workers の場合
npm install -g wrangler
wrangler login
wrangler publish

# git push では自動デプロイされない
# CI/CD を自分で設定する必要がある
```

#### 3. **静的ファイルの提供に向いていない**
```
Workers は動的な API 処理に特化
HTML/CSS/JS の配信は Pages の方が効率的
```

---

## 実際の構成比較

### 🏆 Cloudflare Pages（推奨）

```typescript
// プロジェクト構成
nippo_demo/
├── app/
│   ├── page.tsx                 # トップページ
│   ├── reports/
│   │   ├── page.tsx            # 日報一覧
│   │   └── [id]/page.tsx       # 日報詳細
│   └── layout.tsx
├── functions/
│   └── api/
│       └── reports.ts          # API エンドポイント

// functions/api/reports.ts
export async function onRequestGet({ env }) {
  const token = await getLarkToken(env.LARK_APP_ID, env.LARK_APP_SECRET);
  const reports = await fetchFromLarkBase(token);
  return Response.json(reports);
}

// デプロイ方法
$ git push origin main
→ 自動デプロイ完了！
```

### ❌ Cloudflare Workers（不向き）

```typescript
// 2つのプロジェクトが必要
project-frontend/      # Pages または Vercel
└── Next.js アプリ

project-api/           # Workers
└── src/
    └── index.ts

// wrangler.toml
name = "nippo-api"
main = "src/index.ts"

// デプロイ方法
$ wrangler publish     # 手動 or CI/CD 構築必要
```

---

## 実装ステップ（Cloudflare Pages）

### Step 1: プロジェクト作成

```bash
# Next.js プロジェクト作成
npx create-next-app@latest nippo-demo --typescript --tailwind --app

cd nippo-demo

# Static Export 設定
# next.config.js
module.exports = {
  output: 'export',
  images: {
    unoptimized: true,
  },
}
```

### Step 2: Pages Functions 追加

```bash
# functions ディレクトリ作成
mkdir -p functions/api

# functions/api/reports.ts
export async function onRequestGet({ env }) {
  // Lark Base から日報取得
  const token = await getLarkToken(env);
  const response = await fetch(
    `https://open.larksuite.com/open-apis/bitable/v1/apps/${env.LARK_BASE_ID}/tables/${env.LARK_TABLE_ID}/records`,
    {
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    }
  );

  const data = await response.json();
  return Response.json(data);
}

export async function onRequestPost({ request, env }) {
  // 日報作成
  const body = await request.json();
  // ... Lark API 呼び出し
}
```

### Step 3: GitHub にプッシュ

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/OfficePlata/nippo_demo.git
git push -u origin main
```

### Step 4: Cloudflare Pages 設定

```
1. Cloudflare ダッシュボード → Pages → "Create a project"
2. "Connect to Git" → GitHub 認証
3. リポジトリ選択: OfficePlata/nippo_demo
4. Build settings:
   - Framework preset: Next.js
   - Build command: npm run build
   - Build output directory: out
5. 環境変数設定:
   LARK_APP_ID=cli_xxxxx
   LARK_APP_SECRET=xxxxx
   LARK_BASE_ID=xxxxx
   LARK_TABLE_ID=xxxxx
   R2_BUCKET_NAME=nippo-attachments
6. "Save and Deploy"
```

### Step 5: R2 バケット連携

```bash
# Cloudflare ダッシュボード → R2 → "Create bucket"
バケット名: nippo-attachments

# Pages Settings → Functions → R2 bucket bindings
変数名: R2_BUCKET
バケット: nippo-attachments
```

これで完成！以降は `git push` するだけで自動デプロイされます。

---

## Pages Functions と Workers の技術的な違い

### Pages Functions は Workers のラッパー

```
Pages Functions = Cloudflare Workers + 便利機能

実際には Pages Functions は Workers 上で動作しますが:
✓ ファイルベースルーティング（/functions/api/[name].ts）
✓ 自動的に API エンドポイント生成
✓ ミドルウェアサポート
✓ 環境変数の簡単な管理
✓ GitHub 統合
```

### API エンドポイントの比較

#### Pages Functions
```typescript
// functions/api/reports.ts
export async function onRequestGet() {
  return Response.json({ message: 'Hello' });
}

// 自動的に以下の URL で利用可能:
// https://nippo-demo.pages.dev/api/reports
```

#### Workers
```typescript
// src/index.ts
export default {
  async fetch(request) {
    const url = new URL(request.url);

    // ルーティングを自分で実装
    if (url.pathname === '/api/reports' && request.method === 'GET') {
      return Response.json({ message: 'Hello' });
    }

    return new Response('Not Found', { status: 404 });
  }
}

// wrangler.toml でルート設定必要
routes = [
  { pattern = "api.example.com/*", zone_name = "example.com" }
]
```

---

## よくある質問

### Q1: Workers の方が高機能では？

**A**: 基本的に同じです。Pages Functions は Workers をベースにしているため、Workers でできることはほぼすべて Pages Functions でも可能です。

### Q2: パフォーマンスの違いは？

**A**: ほぼ同じです。どちらも Cloudflare のエッジネットワークで実行されます。

### Q3: いつ Workers を使うべき？

**A**: 以下の場合に Workers を検討:
- API のみ提供（フロントエンドなし）
- 複雑なルーティング・ミドルウェア
- Durable Objects 使用
- Cron Triggers 使用
- WebSocket サーバー

このプロジェクトは「フロントエンド + API」なので **Pages が最適** です。

---

## まとめ

| 項目 | Cloudflare Pages | Cloudflare Workers |
|-----|-----------------|-------------------|
| フロントエンド | ✅ 同梱 | ❌ 別途必要 |
| GitHub 統合 | ✅ ワンクリック | ⚠️ 手動設定 |
| 自動プレビュー | ✅ PR ごと | ❌ なし |
| デプロイ方法 | `git push` | `wrangler publish` |
| API エンドポイント | ✅ ファイルベース | 手動実装 |
| 学習コスト | 低い | やや高い |
| **推奨度（本PJ）** | **🏆 強く推奨** | 不向き |

## 次のステップ

以下の手順で進めることをお勧めします:

1. ✅ Next.js プロジェクト作成
2. ✅ Pages Functions でサンプル API 実装
3. ✅ GitHub にプッシュ
4. ✅ Cloudflare Pages 連携
5. ✅ 動作確認

実装を開始しますか？それとも他に質問はありますか？
