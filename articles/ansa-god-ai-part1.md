---
title: "【CAE×AI】完全自律ヒーリングAIを作る(1)：神の神経系「ATL」という構想"
emoji: "🧠"
type: "tech"
topics: ["CAE", "Python", "Automation", "AI", "Architecture"]
published: true
---

* [第1回：構想と環境構築編](ansa-god-ai-part1)本記事
* [第2回：コアロジック実装編](ansa-god-ai-part2)
* [第3回：自己進化・メタ認知編](ansa-god-ai-part3)

## はじめに：CAE自動化の「壁」を越える

CAEのプリ処理、特にジオメトリの修正（ヒーリング）は、熟練のエンジニアでも手こずる作業です。これを自動化しようとしたとき、従来のスクリプトでは「想定外の形状」で必ず停止してしまいます。

そこで今回は、**「失敗を資産に変え、自ら成長する完全自律AI（God AI）」**を、低スペックなWindows PC上でゼロから構築するプロジェクトを始動します。

第1回は、このAIの核となる**「ATL（Autonomous Trace Logger）」**という思想と、開発環境のセットアップを行います。

## 「神AI」のアーキテクチャ

単なる自動化ツールと、今回作成する「自律AI」の違いは、**中枢神経（ATL）**を持っているかどうかです。

### 従来の自動化
* 状態 → ルール → 実行
* 失敗したら終わり。なぜ失敗したか記録されない。

### 今回構築する「完全自律ANSAヒーリングAI」
AI自身が「考えたが選ばなかった選択肢」や「失敗した理由」をログとして残し、それを元に自らのスキルを書き換えていく構造です。

```text
┌─────────────────────────┐
│        God Core         │
│  (Orchestrator Brain)   │
└─────────┬───────────────┘
          ↓
┌─────────────────────────┐
│   Healing Planner       │  ← 複数案生成（迷い）
└─────────┬───────────────┘
          ↓
┌─────────────────────────┐
│   Skill Executor        │  ← ANSA操作（手足）
└─────────┬───────────────┘
          ↓
┌─────────────────────────┐
│   Geometry Evaluator    │  ← 成否・品質評価（目）
└─────────┬───────────────┘
          ↓
┌─────────────────────────┐
│ ★ ATL (Self Trace Log)  │  ← 中枢神経（経験の蓄積）
└─────────┬───────────────┘
          ↓
┌─────────────────────────┐
│ Skill Confidence DB     │  ← 成長するデータベース
└─────────────────────────┘
```

この**ATL（Autonomous Trace Logger）**こそが、AIにおける「意識」の原型となります。

技術スタックと要件
ハイスペックなマシンは不要です。以下の環境で動作する設計にします。

OS: Windows 10 / 11

言語: Python 3.10

LLM: OpenAI API (または Azure OpenAI) ※今回はロジック中心のためモックで動作可能

DB: SQLite (軽量・構成不要)

UI: Streamlit (神の視点ツール)

ハンズオン：環境構築
それでは、実際に構築を始めましょう。 Windowsの PowerShell を開き、以下のコマンドを順番に実行（コピペ）してください。これでプロジェクトの「器」が完成します。

Step 1: Python環境の準備
まずはPython 3.10をインストールし、仮想環境を作成します。

```PowerShell
# Pythonのインストール（未インストールの場合）
winget install Python.Python.3.10

# 仮想環境の作成と有効化
python -m venv venv
.\venv\Scripts\activate
```

Step 2: ライブラリのインストール
必要最小限のライブラリを入れます。

```PowerShell
pip install pyyaml streamlit requests sqlite-utils
```

Step 3: ディレクトリ構造の作成
「神の器」となるフォルダ構造を一気に作ります。

```PowerShell
mkdir ansa_god_ai
cd ansa_god_ai
mkdir bootstrap core atl skills skill_db ui mcp
```
これで準備は整いました。現在のフォルダ構成は以下のようになっているはずです。

```Plaintext
ansa_god_ai/
├─ bootstrap/   (DB初期化用)
├─ core/        (脳機能: Plan, Eval)
├─ atl/         (中枢神経: Log)
├─ skills/      (スキルの定義ファイル)
├─ skill_db/    (記憶用DB)
├─ ui/          (可視化用)
└─ mcp/         (外部ツール操作用)
```

次回予告
次回は、いよいよPythonコードを実装していきます。 中枢神経である「ATL」を実装し、AIが自律的にプランを立て、実行し、評価してログに残す「自律ループ」を回すところまで進めます。

お楽しみに。
