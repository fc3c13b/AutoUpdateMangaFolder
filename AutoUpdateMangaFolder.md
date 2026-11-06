# AutoUpdateMangaFolder - ソフトウェア仕様書

> **バージョン**: 1.0-draft  
> **最終更新日**: 2026/11/06  
> **ステータス**: ドラフト（実装進行中）

---

## 目次

1. [システム概要](#1-システム概要)
   - 1.1 アプリケーションの目的
   - 1.2 システム要件
   - 1.3 技術スタック
2. [アーキテクチャ](#2-アーキテクチャ)
   - 2.1 全体構成図
   - 2.2 コンポーネント一覧
   - 2.3 データフロー
3. [データモデル](#3-データモデル)
   - 3.1 MangaItem（漫画アイテム）
   - 3.2 VolumeRecord（巻数履歴）
   - 3.3 AppSettings（アプリケーション設定）
   - 3.4 Checkpoint（チェックポイント）
4. [API仕様書](#4-api仕様書)
   - 4.1 エンドポイント一覧
   - 4.2 リクエスト/レスポンス形式
5. [機能仕様](#5-機能仕様)
   - 5.1 クロール機能（MangaCrawler）
   - 5.2 ダウンロード機能（DownloadService）
   - 5.3 タイトル解析（TitleParser）
   - 5.4 設定管理（SettingsManager）
6. [UIコンポーネント](#6-uiコンポーネント)
7. [実装ロードマップ](#7-実装ロードマップ)

---

## 1. システム概要

### 1.1 アプリケーションの目的

**AutoUpdateMangaFolder** は、DL-RAW.SI などの漫画配信サイトから最新巻情報を自動でクロール・検知し、ローカルPCへのダウンロードを補助するデスクトップアプリケーションです。

主な機能：
- **自動クロール**: DL-RAW.SI のカテゴリページを巡回して新刊を検出
- **タイトル正規化**: 投稿タイトルから作品名・巻数・作者情報を抽出・統合
- **チェックポイント機能**: 前回までの進捗を保存し、再開時に続きからクロール可能
- **一括ダウンロード**: 検知された新刊をローカルフォルダに自動ダウンロード
- **リアルタイムログ**: クロール/ダウンロードの進行状況を表示

### 1.2 システム要件

| 項目 | 仕様 |
|------|------|
| OS | Windows 10/11 (64-bit) |
| .NET | .NET 8.0 |
| WebView2 | Microsoft Edge WebView2 Runtime |
| 記憶域 | `%LocalAppData%\AutoUpdateMangaFolder\manga.json`（漫画データ）<br>`%AppData%\AutoUpdateMangaFolder\settings.json`（設定） |

### 1.3 技術スタック

| レイヤ | 技術 | バージョン/説明 |
|--------|------|-----------------|
| **デスクトップフレームワーク** | WPF + WebView2 | .NET 8.0 / Microsoft.Web.WebView2 1.0.4314.35 |
| **HTMLパーサー（C#）** | HtmlAgilityPack | 1.13.1 |
| **フロントエンドフレームワーク** | React 19 | TypeScript + Vite |
| **UIライブラリ** | TailwindCSS v4 | lucide-react (アイコン) / framer-motion (アニメーション) |
| **HTTPクライアント（C#）** | System.Net.Http | HttpClient |
| **HTTPクライアント（Node）** | axios | 1.21.0 |
| **HTMLパーサー（Node）** | cheerio | 1.2.0 |
| **テストフレームワーク** | xUnit | 2.9.3 + coverlet.collector |

---

## 2. アーキテクチャ

### 2.1 全体構成図

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Desktop App (WPF)                           │
│  ┌──────────────┐    ┌──────────────────────────────────────────┐   │
│  │  MainWindow   │    │           WebView2 Host                 │   │
│  │  .xaml.cs     │◀──▶│  (React Frontend via file:// protocol) │   │
│  └──────┬─────────┘    └──────────────────────┬─────────────────┘   │
│         │                                     │                     │
│         ▼                                     ▼                     │
│  ┌──────────────┐                    ┌─────────────────┐            │
│  │ ApiHandler    │◀── chrome.webview │   React UI      │            │
│  │ Service       │     .postMessage  │   (App.tsx)     │            │
│  │ (C# Backend)  │──── fetch proxy ──▶                 │            │
│  └──────┬─────────┘                    └───────────────┘            │
│         │                                                           │
│    ┌────┼────┬────────┬────────┐                                  │
│    ▼     ▼    ▼        ▼       ▼                                │
│  MangaCrawler DownloadService SettingsManager TitleParser      │
│                     MangaDataStore                             │
└─────────────────────────────────────────────────────────────────┘

データ永続化:
  manga.json ──▶ %LocalAppData%\AutoUpdateMangaFolder\
  settings.json ──▶ %AppData%\AutoUpdateMangaFolder\
```

### 2.2 コンポーネント一覧

#### バックエンド（C#）

| クラス | ファイル | 役割 |
|--------|----------|------|
| `ApiHandlerService` | ApiHandlerService.cs | ReactからのAPIリクエストを路由・処理 |
| `MangaCrawler` | MangaCrawler.cs | DL-RAW.SIのページを巡回して漫画データを収集 |
| `DownloadService` | DownloadService.cs | 検知された新刊ファイルをダウンロード |
| `TitleParser` | TitleParser.cs | 投稿タイトルから作品名・巻数・作者を抽出 |
| `MangaDataStore` | MangaDataStore.cs | JSONファイルへの漫画データ保存/読み込み |
| `SettingsManager` | SettingsManager.cs | アプリ設定の永続化とイベント発火 |

#### フロントエンド（React）

| コンポーネント | ファイル | 役割 |
|----------------|----------|------|
| `App` | App.tsx | ルートコンポーネント・状態管理 |
| `CrawlConsole` | CrawlConsole.tsx | クロール制御UI・ログ表示 |
| `MangaTable` | MangaTable.tsx | 漫画一覧テーブル |
| `SettingsModal` | SettingsModal.tsx | 設定ダイアログ |
| `CheckpointCard` | CheckpointCard.tsx | チェックポイント保存/復元UI |
| `FolderScanner` | FolderScanner.tsx | ローカルフォルダスキャン |
| `NewVolumeModal` | NewVolumeModal.tsx | 新刊検知モーダル |
| `DownloadLogViewer` | DownloadLogViewer.tsx | ダウンロード履歴表示 |
| `LiveLogViewer` | LiveLogViewer.tsx | リアルタイムログストリーム |
| `DlRawDatabaseTable` | DlRawDatabaseTable.tsx | DL-RAWデータベース一覧 |
| `ImportUrlsModal` | ImportUrlsModal.tsx | URLインポートダイアログ |
| `Header` | Header.tsx | アプリヘッダー・ナビゲーション |
| `ParserPlayground` | ParserPlayground.tsx | タイトルパーステストツール |
| `ScriptGeneratorModal` | ScriptGeneratorModal.tsx | スクリプト生成モーダル |

### 2.3 データフロー

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│   React UI   │    │ ApiHandler   │    │ MangaCrawler │
│ (ブラウザ)    │◀──▶│ Service      │◀──▶│              │
│             │ GET/POST │           │  Events        │
└─────────────┘         └──────┬───────┘              │
                                │                      │
                        ┌───────┼───────┐            ▼
                        ▼         ▼       ▼      ┌──────────────┐
                   Settings    Download  Title   │ MangaDataStore│
                   Manager     Service   Parser  │ (manga.json)  │
                   (settings)                      └──────────────┘
```



---

## 3. データモデル

### 3.1 MangaItem（漫画アイテム）

バックエンドとフロントエンドで共有される主要なデータ構造。

```typescript
// C# (MangaItem.cs) / TypeScript (types.ts)
interface MangaItem {
  id: number;                    // 一意識別子（C#: GetHashCode()）
  title: string;                 // クリーン化されたタイトル
  normalizedTitle: string;       // 重複排除用正規化キー
  category: string;              // 'manga' | 'magazine' | その他
  author?: string;               // 抽出された作者名
  maxVolume: number;             // 検出された最大巻数
  displayVolume: string;         // 表示用巻数文字列（例: "第19巻"）
  latestPostTitle: string;       // 最新投稿記事タイトル
  latestUrl: string;             // 最新記事URL
  firstSeenAt: string;           // 初回登録日時 (ISO 8601)
  lastUpdatedAt: string;         // 最終更新日時 (ISO 8601)
  history?: VolumeRecord[];      // 検出履歴
}
```

### 3.2 VolumeRecord（巻数履歴）

```typescript
interface VolumeRecord {
  volume: number;                // 巻数
  rawVolumeStr: string;          // 元の巻数文字列
  postTitle: string;             // 投稿タイトル
  url: string;                   // 記事URL
  foundAt: string;               // 検出日時 (ISO 8601)
  page: number;                  // 発見されたページ番号
}
```

### 3.3 AppSettings（アプリケーション設定）

```typescript
interface AppSettings {
  DownloadPath: string;           // ダウンロード先パス（デフォルト: ~/Downloads）
  MaxParallelDownloads: number;   // 同時ダウンロード数 (1-10, デフォルト: 1)
  MaxRetries: number;             // リトライ回数 (0-10, デフォルト: 3)
  RetryDelayBaseMs: number;       // リトライ待機時間 (500-30000ms, デフォルト: 2000)
  RequestTimeoutMinutes: number;  // HTTPタイムアウト (1-60分, デフォルト: 10)
  PreferredFormat: string;        // 'auto' | 'zip' | 'rar'
  BaseUrl?: string;               // クローラーベースURL（デフォルト: null）
  CrawlWaitMs: number;            // ページ間待機時間 (500-30000ms, デフォルト: 1500)
  StopOnCheckpoint: boolean;      // チェックポイントで停止するかどうか
}
```

### 3.4 Checkpoint（チェックポイント）

```typescript
interface Checkpoint {
  title: string;
  rawTitle: string;
  maxVolume: number;
  displayVolume: string;
  url: string;
  page: number;
  savedAt: string;               // ISO 8601
  category?: string;
  previous?: Checkpoint | null;
}

---

## 4. API仕様書

Reactフロントエンドは `window.fetch` のオーバーライドを通じて `/api/*` エンドポイントにアクセスします。C#側で `ApiHandlerService.HandleRequest()` がリクエストを処理します。

### 4.1 エンドポイント一覧

| メソッド | パス | 機能 | リクエストボディ |
|----------|------|------|------------------|
| GET | `/api/status` | クロール状態・統計取得 | なし |
| POST | `/api/crawl/start` | クロール開始 | CrawlSettings JSON |
| POST | `/api/crawl/stop` | クロール停止 | なし |
| POST | `/api/crawl/reset` | クロール状態リセット | なし |
| GET | `/api/manga/list` | 漫画一覧取得 | なし |
| GET | `/api/manga/:id` | 個別漫画詳細 | なし |
| POST | `/api/manga/add` | 漫画追加 | MangaItem JSON |
| POST | `/api/manga/update` | 漫画更新 | MangaItem JSON |
| DELETE | `/api/manga/:id` | 漫画削除 | なし |
| GET | `/api/settings` | 設定取得 | なし |
| POST | `/api/settings` | 設定更新 | AppSettings JSON |
| POST | `/api/settings/reset` | 設定リセット | なし |

### 4.2 リクエスト/レスポンス形式

**共通レスポンスフォーマット:**
```json
{
  "success": true,
  "data": { ... },    // または
  "error": "エラーメッセージ"
}
```

#### GET /api/status

**レスポンス例:**
```json
{
  "success": true,
  "data": {
    "isCrawling": false,
    "currentPage": 0,
    "totalPages": 100,
    "mangaCount": 5234,
    "mangaCategoryCount": 4500,
    "magazineCategoryCount": 734,
    "lastUpdated": "2026-11-06T12:00:00.0000000+09:00"
  }
}
```

#### POST /api/crawl/start

**リクエストボディ:**
```json
{
  "category": "manga",
  "startPage": 1,
  "maxPages": 50,
  "waitMs": 1500,
  "stopOnCheckpoint": false
}
```

#### POST /api/settings

**リクエストボディ:**
```json
{
  "DownloadPath": "C:\\Users\\User\\Downloads",
  "MaxParallelDownloads": 2,
  "MaxRetries": 3,
  "RetryDelayBaseMs": 2000,
  "RequestTimeoutMinutes": 10,
  "PreferredFormat": "auto",
  "BaseUrl": null,
  "CrawlWaitMs": 1500,
  "StopOnCheckpoint": false
}
```


---

## 5. 機能仕様

### 5.1 クロール機能（MangaCrawler）

#### 概要
DL-RAW.SI のカテゴリページを巡回して、漫画の最新巻情報を収集します。

#### 処理フロー
```
1. カテゴリ判定 (manga / magazine)
       │
2. ページ巡回 (startPage → maxPages)
       │
3. HTML取得・パース (HtmlAgilityPack)
       │
4. タイトル解析 (TitleParser.ParsePostTitle)
       │
5. 重複判定 (normalizedTitle で照合)
       │
6a. 新規登録 → MangaDataStore に追加
6b. 既存更新 → maxVolume が大きい場合のみ更新
       │
7. チェックポイント保存
```

#### イベント
| イベント | タイミング |
|----------|------------|
| `OnLog` | ログ出力時 (info/success/warning/error) |
| `OnStatusChange` | クロール状態変化時 |

#### 設定パラメータ
| パラメータ | デフォルト | 範囲 |
|-----------|-----------|------|
| startPage | 1 | ≥ 1 |
| maxPages | - | ≥ startPage |
| category | "manga" | "manga" \| "magazine" |
| waitMs | 1500 | 500-30000ms |

### 5.2 ダウンロード機能（DownloadService）

#### 概要
検知された新刊ファイルをローカルPCにダウンロードします。

#### 処理フロー
```
1. MangaDataStore から対象リスト取得
       │
2. セマフォ制御で並列ダウンロード
       │
3. ファイル名生成 (タイトル + 巻数 + 拡張子)
       │
4. HTTP GET → ストリームコピー
       │
5. リトライ機構 (最大 MaxRetries 回)
       │
6. 完了/失敗のログ出力
```

#### ファイル名生成ルール
```
{タイトル}[{巻数}]{拡張子}
例: "そうだ、売国しよう[19巻].zip"
```

### 5.3 タイトル解析（TitleParser）

#### 概要
DL-RAW.SI の投稿タイトルから構造化データを抽出します。

#### 正規化処理
```typescript
NormalizeTitle(title): string {
  // NFKC正規化 → 小文字変換 → 半角スペース・ダッシュ・アンダースコア除去
  // 全角括弧・記号の除去
}
```

#### カテゴリ判定ルール
| パターン | カテゴリ |
|----------|--------|
| `[一般コミック]` | 一般コミック |
| `[成年コミック]` | 成年コミック |
| `[Light Novel]` / `[ライトノベル]` | Light Novel |
| `[雑誌]` または雑誌関連キーワード | 雑誌 |
| `[同人誌]` | 同人誌 |

#### 巻数抽出パターン
- `第19巻` → maxVolume: 19, displayVolume: "第19巻"
- `15-20巻` → maxVolume: 20, displayVolume: "第20巻"
- `2024年1月号` (雑誌) → maxVolume: 1, displayVolume: "1月号"

### 5.4 設定管理（SettingsManager）

#### 概要
アプリケーション設定の永続化と変更イベントの発火を担当します。

#### アーキテクチャ
```
┌───────────────┐    ┌──────────────────┐
│ SettingsModal │──▶│ ApiHandlerService │
│ (React)       │   │ POST /api/settings│
└───────────────┘    └────────┬─────────┘
                              │ Save()
                              ▼
                       ┌───────────────┐
                       │ SettingsManager│
                       │  .Save()      │
                       │  → settings.json
                       │  → Invoke     │
                       │    SettingsChanged
                       └───────┬───────┘
                               │ Event
                               ▼
                        ┌───────────────┐
                        │ApiHandlerService│
                        │ OnSettingsUpdated│
                        │ _state 更新     │
                        └───────────────┘
```

#### イベント駆動型同期
- `SettingsManager.SettingsChanged` イベントを購読
- 設定変更時に `OnSettingsUpdated()` が呼び出され、内部 `_state` を即時更新
- アプリ再起動なしでクロールパラメータが反映される


---

## 6. UIコンポーネント

### フロントエンド構造

```
autoupdatemangafolder/
├── src/
│   ├── App.tsx                    # ルート・状態管理
│   ├── main.tsx                   # エントリーポイント
│   ├── types.ts                   # 型定義
│   ├── components/
│   │   ├── CrawlConsole.tsx       # クロール制御パネル
│   │   ├── MangaTable.tsx         # 漫画一覧テーブル
│   │   ├── SettingsModal.tsx      # 設定ダイアログ
│   │   ├── CheckpointCard.tsx     # チェックポイント管理
│   │   ├── FolderScanner.tsx      # ローカルスキャン
│   │   ├── NewVolumeModal.tsx     # 新刊検知モーダル
│   │   ├── DownloadLogViewer.tsx  # ダウンロード履歴
│   │   ├── LiveLogViewer.tsx      # リアルタイムログ
│   │   ├── DlRawDatabaseTable.tsx # DL-RAW DB一覧
│   │   ├── DlRawDetailModal.tsx   # 詳細モーダル
│   │   ├── ImportUrlsModal.tsx    # URLインポート
│   │   ├── Header.tsx             # ヘッダー
│   │   ├── ParserPlayground.tsx   # パーサテスト
│   │   └── ScriptGeneratorModal.tsx # スクリプト生成
│   └── utils/
│       ├── mangaParser.ts         # 漫画パーサー
│       ├── parser.ts              # タイトルパーサー
│       ├── fileSystem.ts          # ファイルシステム操作
│       └── cbzGenerator.ts        # CBZ生成
├── server.ts                      # Node.js サーバー（開発用）
├── package.json                   # npm依存関係
└── vite.config.ts                 # Vite設定
```


---

## 7. 実装ロードマップ

### 現在の進捗状況

| # | タスク | ステータス | 備考 |
|---|--------|-----------|------|
| 1 | C# クロール機能 | ✅ 完了 | MangaCrawler + TitleParser 動作確認済み |
| 2 | C# ダウンロード機能 | ✅ 完了 | DownloadService 実装済み |
| 3 | C# APIハンドラ | ✅ 完了 | ApiHandlerService で全エンドポイント対応 |
| 4 | C# 設定管理 | ✅ 完了 | SettingsManager + イベント機構実装済み |
| 5 | React フロントエンド | ✅ 完了 | 16コンポーネント・型定義・パーサー実装済み |
| 6 | WPF-WebView2統合 | ✅ 完了 | MainWindow.xaml.cs でfetchプロキシ実装 |
| 7 | 設定リアルタイム同期 | ⚠️ 進行中 | AppModifyingList.md 参照 |

### 未実装機能（計画）

| # | 機能 | 優先度 | 説明 |
|---|------|--------|------|
| 8 | ダウンロードAPI統合 | 高 | `/api/download/*` エンドポイント追加 |
| 9 | WebSocketログストリーム | 中 | リアルタイムログの双方向通信 |
| 10 | チェックポイントAPI | 中 | `/api/checkpoint/*` エンドポイント追加 |
| 11 | ローカルフォルダスキャン | 低 | FolderScanner のC#実装 |

---

## 付録A: ファイル構成

### C# バックエンド（DesktopApp/）

```
DesktopApp/
├── AutoUpdateMangaFolder.Desktop.csproj  # WPFプロジェクト
├── ApiHandlerService.cs                   # API路由・処理
├── AppSettings.cs                         # 設定モデル
├── DownloadService.cs                     # ダウンロード処理
├── MangaCrawler.cs                        # クロールロジック
├── MangaDataStore.cs                      # JSON永続化
├── SettingsManager.cs                     # 設定管理・イベント
├── TitleParser.cs                         # タイトル解析
├── MainWindow.xaml.cs                     # WPFメインウィンドウ
├── App.xaml.cs                            # アプリケーション初期化
└── AssemblyInfo.cs                        # アセンブリ情報
```

### テストプロジェクト（AutoUpdateMangaFolder.Tests/）

```
AutoUpdateMangaFolder.Tests/
├── AutoUpdateMangaFolder.Tests.csproj     # xUnit テストプロジェクト
└── (テストファイル)                         # 単体テスト・統合テスト
```

### React フロントエンド（autoupdatemangafolder/）

```
autoupdatemangafolder/
├── package.json                           # npm依存関係
├── vite.config.ts                         # Vite設定
├── tsconfig.json                          # TypeScript設定
├── server.ts                              # 開発用Expressサーバー
├── src/
│   ├── App.tsx                            # ルートコンポーネント
│   ├── main.tsx                           # エントリーポイント
│   ├── types.ts                           # 型定義 (19インターフェース)
│   ├── components/                        # UIコンポーネント (14ファイル)
│   └── utils/                             # ユーティリティ (4ファイル)
└── public/                                # 静的アセット
```


---

## 付録B: ビルド・実行手順

### C# バックエンド
```powershell
cd c:\source\repos\AutoUpdateMangaFolder\DesktopApp
dotnet build
dotnet run
```

### React フロントエンド（開発）
```bash
cd c:\source\repos\AutoUpdateMangaFolder\autoupdatemangafolder
npm install
npm run dev    # Vite HMRサーバー (http://localhost:5173)
```

### React フロントエンド（本番ビルド）
```bash
npm run build  # → dist/ に出力
# WPFアプリは dist/ の index.html を WebView2 で読み込む
```

### テスト実行
```powershell
cd c:\source\repos\AutoUpdateMangaFolder\AutoUpdateMangaFolder.Tests
dotnet test
```

---

## 付録C: 既知の問題・技術的負債

| # | 問題 | 影響度 | 対応予定 |
|---|------|--------|----------|
| 1 | `UpdateSettings()` で `SettingsChanged` イベント発火漏れ | 高 | AppModifyingList.md タスク1 |
| 2 | React SettingsModal がローカルstateのみ更新 | 中 | AppModifyingList.md タスク3 |
| 3 | 起動時設定フェッチ未実装 | 低 | AppModifyingList.md タスク2 |
| 4 | Node.js サーバー（server.ts）は開発用で本番不使用 | 情報 | - |

---

*文書作成日: 2026/11/06*  
*次期更新予定: 実装ロードマップ完了後*

```
