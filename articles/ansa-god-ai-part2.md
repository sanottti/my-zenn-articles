---
title: "【CAE×AI】完全自律ヒーリングAIを作る(2)：中枢神経「ATL」と自律ループの実装"
emoji: "⚙️"
type: "tech"
topics: ["Python", "SQLite", "SystemDesign", "Automation"]
published: true
---

* [第1回：構想と環境構築編](ansa-god-ai-part1)
* [第2回：コアロジック実装編](ansa-god-ai-part2)本記事
* [第3回：自己進化・メタ認知編](ansa-god-ai-part3)
 
## はじめに

[前回](ansa-god-ai-part1)は、「完全自律ANSAヒーリングAI」の構想とディレクトリ作成を行いました。
今回は、AIの**中枢神経であるATL（Autonomous Trace Logger）**と、思考のループ（Plan → Do → Check）を実装し、実際に動かしてみます。

すべてのコードは、前回の記事で作成した `ansa_god_ai` フォルダ内で作成してください。

## 1. データベースの初期化 (Bootstrap)

まずは、AIの記憶領域となるSQLiteデータベースを用意します。

**`bootstrap/init_db.py`**
```python
import sqlite3
import os

# ディレクトリがない場合に備えて作成
os.makedirs("skill_db", exist_ok=True)

conn = sqlite3.connect("skill_db/skill_confidence.db")
cur = conn.cursor()

# ATLログ（思考と行動の履歴）
cur.execute("""
CREATE TABLE IF NOT EXISTS atl_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT,
    state_id TEXT,
    candidates TEXT,
    selected TEXT,
    result TEXT
)
""")

# スキル信頼度（成功率の記憶）
cur.execute("""
CREATE TABLE IF NOT EXISTS skill_confidence (
    skill TEXT PRIMARY KEY,
    success REAL,
    trials INTEGER
)
""")

conn.commit()
conn.close()
print("Database initialized.")
```

作成したら、一度だけ実行してDBファイルを生成します。

```PowerShell
python bootstrap/init_db.py
```

2. ATL（中枢神経）の実装
これが本システムの核です。AIが行ったことだけでなく、「候補に挙がったが選ばなかったプラン」も含めて記録します。

atl/self_trace_logger.py

```Python
import json
import sqlite3
from datetime import datetime

DB = "skill_db/skill_confidence.db"

def log_trace(state_id, candidates, selected, result):
    conn = sqlite3.connect(DB)
    cur = conn.cursor()
    cur.execute("""
    INSERT INTO atl_log
    (timestamp, state_id, candidates, selected, result)
    VALUES (?, ?, ?, ?, ?)
    """, (
        datetime.utcnow().isoformat(),
        state_id,
        json.dumps(candidates),
        selected,
        json.dumps(result)
    ))
    conn.commit()
    conn.close()
    print(f"[ATL] Trace logged for state: {state_id}")
```

3. 初期スキルの用意
AIが使う「技」を定義します。今回はYAML形式で管理します。

skills/heal_faces.yaml

```YAML
name: heal_faces
description: Heal small disconnected faces
params:
  tolerance: [0.01, 0.05]
```

※同様に merge_edges.yaml などの名前で空ファイルでも良いので作っておくと動作確認がスムーズです。

4. 思考エンジンの実装
Planner（計画立案）
複数のスキルロードし、ランダムなパラメータで「候補（Plan）」を生成します。

core/planner.py

```Python
import yaml
import random
import os
import glob

def load_skills():
    skills = []
    # skillsフォルダ内の全yamlを読み込む
    yaml_files = glob.glob("skills/*.yaml")
    for f in yaml_files:
        try:
            with open(f) as fp:
                skills.append(yaml.safe_load(fp))
        except Exception as e:
            print(f"Error loading {f}: {e}")
    return skills

def propose(state):
    plans = []
    for s in load_skills():
        # パラメータをランダムに決定（初期段階の探索）
        plans.append({
            "skill": s["name"],
            "params": {k: random.choice(v) for k,v in s["params"].items()},
            "expected_success": random.random() # 本来は学習モデルから予測
        })
    return plans
```

Executor（実行）
実際のCAEソフト（ANSAなど）を叩く部分です。今回はモック（模擬）として実装します。

mcp/ansa_client.py

```Python
def execute(skill, params):
    # 実際はここでMCP経由でANSA APIを叩く
    print(f"Executing {skill} with {params}...")
    # 模擬的な結果を返す
    return {"status": "ok", "fixed_errors": 5}
```

Evaluator（評価）
実行結果が良いものだったか判断します。

core/evaluator.py

```Python
def evaluate(result):
    # 修正数が多いほどスコアが高いとする簡易ロジック
    return {
        "quality_delta": result.get("fixed_errors", 0) / 10.0
    }
```

5. 自律ループの起動 (Orchestrator)
全てのモジュールを統合し、自律的に動くスクリプトを作成します。

run_autonomous.py

```Python
from core.planner import propose
from mcp.ansa_client import execute
from core.evaluator import evaluate
from atl.self_trace_logger import log_trace
import uuid

def main():
    # 1. 状態IDの発行（UUID）
    state_id = str(uuid.uuid4())
    print(f"--- Start Cycle: {state_id} ---")

    # 2. プランニング
    plans = propose(state_id)
    if not plans:
        print("No skills found.")
        return

    # 3. 意思決定（期待値が最も高いものを選ぶ）
    selected = max(plans, key=lambda x: x["expected_success"])
    print(f"Selected Strategy: {selected['skill']}")

    # 4. 実行
    result = execute(selected["skill"], selected["params"])

    # 5. 評価
    evaluation = evaluate(result)

    # 6. ATLへの記録（中枢神経への刻印）
    log_trace(
        state_id,
        plans,
        selected["skill"],
        evaluation
    )

    print("Autonomous healing cycle completed.")

if __name__ == "__main__":
    main()
```

動作確認
以下のコマンドを実行してみましょう。

```PowerShell
python run_autonomous.py
```

エラーなく Autonomous healing cycle completed. と表示されれば成功です。 これで、AIは「計画し、実行し、評価し、記憶する」という基本ループを手に入れました。

次回予告
次回は完結編。 蓄積されたATLのログを元に**「AIが自分でスキルを改造する（自己進化）」機能と、「自分を疑うメタ評価エージェント」**を実装し、神AIを完成させます。
