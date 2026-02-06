---
title: "【CAE×AI】完全自律ヒーリングAIを作る(3)：自己進化とメタ認知の実装"
emoji: "🧬"
type: "tech"
topics: ["GenerativeAI", "Python", "SelfCorrection", "MetaCognition"]
published: true
---

## はじめに

[前回](ansa-god-ai-part2)までで、AIが自律的に行動し、そのすべてをATL（Autonomous Trace Logger）に記憶する仕組みができました。

最終回となる今回は、このシステムに「知能」を吹き込みます。
具体的には、以下の2つの機能を実装し、**「失敗から学び、自らコード（YAML）を書き換え、自分を疑うAI」**へと進化させます。

1.  **Skill Rebuilder**: 成功率の低いスキルを、パラメータ調整して「v2, v3...」と世代交代させる。
2.  **Meta Critic**: 「最近、同じ手ばかり使っていないか？」と自らの行動を疑う。

## フォルダ構成の追加

まずは拡張用のフォルダを作ります。PowerShellで実行してください。

```powershell
mkdir skill_rebuilder
mkdir meta
mkdir skill_registry
mkdir skills_generated
```

1. スキル世代管理データベースの作成
「どのスキルが、なぜ生まれたか」を管理するための台帳を作ります。

skill_registry/init_skill_registry.py

```Python
import sqlite3
import os

os.makedirs("skill_registry", exist_ok=True)

conn = sqlite3.connect("skill_registry/skills.db")
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS skill_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    base_skill TEXT,
    version INTEGER,
    yaml_path TEXT,
    created_reason TEXT,
    created_at TEXT
)
""")

conn.commit()
conn.close()
print("Skill Registry initialized.")
```

作成後、初期化を実行します。

```PowerShell
python skill_registry/init_skill_registry.py
```

2. 自己進化の実装（Skill Rebuilder）
ATLのログを分析し、成績の悪いスキルがあれば、パラメータ範囲を微調整した「次世代バージョン」を自動生成するコードです。

skill_rebuilder/rebuild_skills.py

```Python
import sqlite3
import json
import yaml
import os
from datetime import datetime

ATL_DB = "skill_db/skill_confidence.db"
REG_DB = "skill_registry/skills.db"
OUTPUT_DIR = "skills_generated"

os.makedirs(OUTPUT_DIR, exist_ok=True)

def rebuild():
    atl = sqlite3.connect(ATL_DB)
    reg = sqlite3.connect(REG_DB)

    atl_cur = atl.cursor()
    reg_cur = reg.cursor()

    # ATLログから全履歴を取得
    atl_cur.execute("SELECT candidates, selected FROM atl_log")
    rows = atl_cur.fetchall()

    stats = {}

    # 成功率の集計
    for candidates_json, selected in rows:
        candidates = json.loads(candidates_json)
        for c in candidates:
            name = c["skill"]
            stats.setdefault(name, {"trials":0, "selected":0})
            stats[name]["trials"] += 1
            if name == selected:
                stats[name]["selected"] += 1

    for skill, s in stats.items():
        success = s["selected"] / max(1, s["trials"])

        # 成功率が低い(40%未満)場合、進化させる
        if success < 0.4:
            # 現在の最新バージョンを確認
            reg_cur.execute(
                "SELECT MAX(version) FROM skill_history WHERE base_skill=?", 
                (skill,)
            )
            row = reg_cur.fetchone()
            next_ver = (row[0] or 1) + 1

            new_name = f"{skill}_v{next_ver}"

            # 新しいスキル定義（仮のパラメータ改善）
            new_skill = {
                "name": new_name,
                "base_skill": skill,
                "version": next_ver,
                "description": f"Auto regenerated due to low success rate ({success:.2f})",
                "params": {
                    "tolerance": [0.003, 0.02] 
                }
            }

            # YAMLファイル生成
            path = f"{OUTPUT_DIR}/{new_name}.yaml"
            with open(path, "w") as f:
                yaml.dump(new_skill, f)

            # 履歴に登録
            reg_cur.execute("""
            INSERT INTO skill_history 
            (base_skill, version, yaml_path, created_reason, created_at)
            VALUES (?, ?, ?, ?, ?)
            """, (
                skill, next_ver, path,
                f"Low success rate: {success:.2f}",
                datetime.utcnow().isoformat()
            ))

            print(f"🧬 Generated {new_name}")

    atl.close()
    reg.commit()
    reg.close()

if __name__ == "__main__":
    rebuild()
```

3. メタ認知の実装（Meta Critic）
「神が自分を疑う」機能です。直近の行動に偏りがないかを監視します。

meta/meta_critic.py

```Python
import sqlite3
from collections import Counter

DB = "skill_db/skill_confidence.db"

def critique():
    conn = sqlite3.connect(DB)
    cur = conn.cursor()

    # 直近30回の行動を取得
    cur.execute("SELECT selected FROM atl_log ORDER BY id DESC LIMIT 30")
    rows = [r[0] for r in cur.fetchall()]
    conn.close()

    if not rows:
        return []

    count = Counter(rows)
    total = len(rows)
    critiques = []

    for skill, c in count.items():
        ratio = c / total
        # 特定のスキルばかり選んでいたら警告
        if ratio > 0.5:
            critiques.append({
                "skill": skill,
                "reason": "overuse",
                "message": f"{skill} was selected in {ratio:.0%} of recent runs, indicating exploration bias"
            })
    
    return critiques
```

4. 全システムの統合
最後に、run_autonomous.py をアップデートして、メタ認知の声をログに残せるようにします。

atl/self_trace_logger.py に以下の関数を追記してください。

```Python
def log_doubt(skill, reason, message):
    conn = sqlite3.connect(DB)
    cur = conn.cursor()
    cur.execute("""
    INSERT INTO atl_log
    (timestamp, state_id, candidates, selected, result)
    VALUES (?, ?, ?, ?, ?)
    """, (
        datetime.utcnow().isoformat(),
        "META",
        "[]",
        skill,
        json.dumps({"doubt": reason, "message": message})
    ))
    conn.commit()
    conn.close()
    print(f"[ATL] Metacognition Logged: {message}")
```

そして、run_autonomous.py を書き換えます。

```Python
from core.planner import propose
from mcp.ansa_client import execute
from core.evaluator import evaluate
from atl.self_trace_logger import log_trace, log_doubt
from meta.meta_critic import critique
import uuid

def main():
    state_id = str(uuid.uuid4())
    
    # 1. プランニング & 実行
    plans = propose(state_id)
    if not plans: return
    selected = max(plans, key=lambda x: x["expected_success"])
    result = execute(selected["skill"], selected["params"])
    evaluation = evaluate(result)
    
    # 2. 通常ログ記録
    log_trace(state_id, plans, selected["skill"], evaluation)

    # 3. ★神の自己懐疑（メタ認知）
    for c in critique():
        log_doubt(c["skill"], c["reason"], c["message"])

    print("🤖 Autonomous cycle with self-doubt completed.")

if __name__ == "__main__":
    main()
```

最終確認
以下のコマンドを実行して、AIが動き出すのを確認してください。

自律ループを何度か回す： python run_autonomous.py

自己進化（スキル再生成）を実行： python skill_rebuilder/skill_rebuilder.py

もし成績が悪ければ、skills_generated フォルダに heal_faces_v2.yaml が自動生成されます。これが「AIが自分で作ったスキル」です。

まとめ：CAE × AGI の境界線へ
全3回で構築したのは、単なる自動化スクリプトではありません。 **「失敗を記憶し（ATL）、自らのコードを書き換え（Rebuilder）、自らの行動を疑う（Meta Critic）」**という、生命的な特徴を持ったシステムです。

このアーキテクチャは、CAEに限らず、あらゆるエンジニアリングの自動化に応用可能です。ぜひ、あなたの手でこの「神AI」を育ててみてください。