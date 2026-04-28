---
name: tech-news-aggregator
description: Hacker News、Reddit、Zenn.dev、GitHub、HuggingFace、AgentSkillsHub からテック系最新情報を収集してトピックスレポートを生成するスキル。認証不要の公開APIを使用。
category: web-search
tags: [tech, news, hackernews, reddit, zenn, github, huggingface, agentskillshub, aggregator]
created: 2026-04-24
updated: 2026-04-26
---

# テックニュース集約スキル

## 概要
Hacker News、Reddit（r/MachineLearning / r/LocalLLaMA / r/programming）、Zenn.dev、
GitHub（直近1週間の急上昇リポジトリ）、HuggingFace（7日間トレンドモデル・スペース）、
AgentSkillsHub（AIエージェントスキル・MCPサーバーのTrending Top10）から
テック系最新トピックスを収集し、日本語レポートを生成して保存する。

## 前提条件
- curl、python3 が使えること（WSL環境）
- インターネット接続があること

---

## ステップ1: データ収集

**⚠️ 以下のスクリプトは内容・フォーマットを一切改変せずそのままexecute_codeで実行すること。**

    import json, subprocess, time, os
    from datetime import datetime, timedelta

    def curl(url, headers=None):
        cmd = ["curl", "-s", "--max-time", "10"]
        if headers:
            for k, v in headers.items():
                cmd += ["-H", f"{k}: {v}"]
        cmd.append(url)
        r = subprocess.run(cmd, capture_output=True, text=True)
        return r.stdout

    # Hacker News
    def get_hn_stories(count=40):
        ids = json.loads(curl("https://hacker-news.firebaseio.com/v0/topstories.json"))[:count]
        stories = []
        for sid in ids:
            try:
                d = json.loads(curl(f"https://hacker-news.firebaseio.com/v0/item/{sid}.json"))
                if d.get("title"):
                    stories.append({
                        "title": d.get("title", ""),
                        "score": d.get("score", 0),
                        "url": d.get("url", ""),
                        "comments": d.get("descendants", 0),
                        "hn_url": f"https://news.ycombinator.com/item?id={sid}"
                    })
            except:
                pass
            time.sleep(0.05)
        return sorted(stories, key=lambda x: x["score"], reverse=True)[:15]

    # Reddit (User-Agent必須 — なしだと403)
    UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"

    def get_reddit_posts(subreddit, time_filter="week", count=10):
        url = f"https://www.reddit.com/r/{subreddit}/top.json?t={time_filter}&limit={count}"
        raw = curl(url, headers={"User-Agent": UA})
        try:
            posts = []
            for child in json.loads(raw).get("data", {}).get("children", []):
                p = child.get("data", {})
                posts.append({
                    "title": p.get("title", ""),
                    "score": p.get("score", 0),
                    "comments": p.get("num_comments", 0),
                    "url": f"https://reddit.com{p.get('permalink', '')}"
                })
            return posts
        except:
            return []

    reddit_posts = {}
    for sub in ["MachineLearning", "LocalLLaMA", "programming"]:
        reddit_posts[sub] = get_reddit_posts(sub)
        time.sleep(0.5)

    # Zenn.dev
    def get_zenn_articles(count=10):
        raw = curl(f"https://zenn.dev/api/articles?order=weekly&count={count}")
        try:
            articles = []
            for a in json.loads(raw).get("articles", []):
                articles.append({
                    "title": a.get("title", ""),
                    "likes": a.get("liked_count", 0),
                    "url": f"https://zenn.dev{a.get('path', '')}",
                    "emoji": a.get("emoji", ""),
                    "published_at": a.get("published_at", "")[:10]
                })
            return sorted(articles, key=lambda x: x["likes"], reverse=True)
        except:
            return []

    # GitHub トレンド (未認証60req/時 — 1日1回なら問題なし)
    def get_github_trending(count=10):
        week_ago = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d")
        url = (f"https://api.github.com/search/repositories"
               f"?q=created:>{week_ago}&sort=stars&order=desc&per_page={count}")
        raw = curl(url, headers={"Accept": "application/vnd.github.v3+json"})
        try:
            repos = []
            for r in json.loads(raw).get("items", []):
                repos.append({
                    "name": r.get("full_name", ""),
                    "stars": r.get("stargazers_count", 0),
                    "description": (r.get("description") or "")[:80],
                    "url": r.get("html_url", ""),
                    "language": r.get("language") or ""
                })
            return repos
        except:
            return []

    # AgentSkillsHub Trending（Supabase公開APIを直接呼び出し — Playwright不要）
    ASH_URL = "https://vknzzecmzsfmohglpfgm.supabase.co/rest/v1/rpc/get_landing_data"
    ASH_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InZrbnp6ZWNtenNmbW9oZ2xwZmdtIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzI4MDQ3MzIsImV4cCI6MjA4ODM4MDczMn0.zFAGZH-lDcL-GwyMkR-9sSV8pJToVzomsJ_fuXZIoDo"

    def get_agentskillshub_trending():
        # max-time 30秒 — このRPCはサーバー負荷次第で遅い。10秒だとタイムアウトする
        for attempt in range(2):
            r = subprocess.run([
                "curl", "-s", "--max-time", "30", "-X", "POST", ASH_URL,
                "-H", f"apikey: {ASH_KEY}",
                "-H", f"Authorization: Bearer {ASH_KEY}",
                "-H", "Content-Type: application/json",
                "-d", "{}"
            ], capture_output=True, text=True)
            try:
                data = json.loads(r.stdout)
                trending = data.get("trending", [])
                if not trending:
                    time.sleep(3)
                    continue
                items = []
                for t in trending:
                    items.append({
                        "name": t.get("repo_full_name", ""),
                        "description": (t.get("description") or "")[:80],
                        "stars": t.get("stars", 0),
                        "score": t.get("score", 0),
                        "category": t.get("category", ""),
                        "url": f"https://agentskillshub.top/skill/{t.get('repo_full_name', '')}/"
                    })
                return items
            except:
                time.sleep(3)
        return []

    # HuggingFace トレンド（クライアント側でlikes降順に再ソート）
    def get_hf_trending(endpoint, count=8):
        raw = curl(f"https://huggingface.co/api/{endpoint}?sort=likes7d&direction=-1&limit={count}")
        try:
            items = []
            for item in json.loads(raw):
                # spacesエンドポイントはURLに /spaces/ プレフィックスが必要
                if endpoint == "spaces":
                    url = f"https://huggingface.co/spaces/{item.get('id', '')}"
                else:
                    url = f"https://huggingface.co/{item.get('id', '')}"
                items.append({
                    "id": item.get("id", ""),
                    "likes": item.get("likes", 0),
                    "url": url
                })
            return sorted(items, key=lambda x: x["likes"], reverse=True)
        except:
            return []

    # レポート生成（プレースホルダー付き）
    def generate_report(hn, reddit, zenn, gh, hf_m, hf_s, ash):
        today = datetime.now().strftime("%Y-%m-%d")
        lines = [
            f"# テックニュース トピックスレポート {today}", "",
            f"収集日: {today}",
            "ソース: Hacker News / Reddit / Zenn.dev / GitHub / HuggingFace / AgentSkillsHub",
            "", "---", "", "## Hacker News", ""
        ]
        for i, s in enumerate(hn[:10], 1):
            link = s["url"] or s["hn_url"]
            lines += [f"{i}. {s['title']}",
                      f"   {s['score']}pts / {s['comments']}コメント | {link}", ""]

        lines += ["---", "", "## Reddit", ""]
        for sub, posts in reddit.items():
            if posts:
                lines.append(f"### r/{sub}")
                for p in posts[:5]:
                    lines.append(f"- {p['title']} | {p['score']}pts | {p['url']}")
                lines.append("")

        lines += ["---", "", "## Zenn.dev", ""]
        for a in zenn[:8]:
            lines.append(f"- {a['emoji']} {a['title']} | いいね{a['likes']} | {a['url']}")
        lines.append("")

        lines += ["---", "", "## GitHub トレンド（直近1週間）", ""]
        for r in gh[:10]:
            lang = f" [{r['language']}]" if r["language"] else ""
            lines.append(f"- ⭐{r['stars']}{lang} {r['name']}")
            if r["description"]:
                lines.append(f"  {r['description']}")
            lines.append(f"  {r['url']}")
        lines.append("")

        lines += ["---", "", "## HuggingFace トレンド（7日間）", "", "### モデル", ""]
        for m in hf_m:
            lines.append(f"- ❤️{m['likes']} {m['id']} | {m['url']}")
        lines += ["", "### スペース", ""]
        for s in hf_s:
            lines.append(f"- ❤️{s['likes']} {s['id']} | {s['url']}")
        lines += ["---", "", "## AgentSkillsHub Trending（AIエージェントスキル Top10）", ""]
        for i, s in enumerate(ash[:10], 1):
            cat = f" [{s['category']}]" if s["category"] else ""
            lines.append(f"{i}. {s['name']}{cat} | ⭐{s['stars']} | score:{s['score']}")
            if s["description"]:
                lines.append(f"   {s['description']}")
            lines.append(f"   {s['url']}")
        lines.append("")

        lines += ["", "---", "", "## 今週のポイント（AIによる分析）", "",
                  "PLACEHOLDER_ANALYSIS", ""]
        return "\n".join(lines)

    hn = get_hn_stories()
    zenn = get_zenn_articles(10)
    gh = get_github_trending(10)
    hf_m = get_hf_trending("models", 8)
    hf_s = get_hf_trending("spaces", 5)
    ash = get_agentskillshub_trending()
    report = generate_report(hn, reddit_posts, zenn, gh, hf_m, hf_s, ash)

    workspace = os.path.expanduser("~/hermes-workspace")
    os.makedirs(workspace, exist_ok=True)
    today = datetime.now().strftime("%Y-%m-%d")
    filepath = f"{workspace}/tech_topics_{today}.md"
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(report)
    print(f"ステップ1完了: {filepath}")
    print(f"HN:{len(hn)}件 Reddit:{sum(len(v) for v in reddit_posts.values())}件 "
          f"Zenn:{len(zenn)}件 GitHub:{len(gh)}件 HF_M:{len(hf_m)}件 HF_S:{len(hf_s)}件 ASH:{len(ash)}件")

---

## ステップ2: AI分析の生成とファイル追記

**⚠️ ステップ1が成功したことを確認してから実行すること。**

ステップ1の出力データをすべて参照し、以下の基準で「今週のポイント」を3〜5点作成する：

- 複数ソース（HN・Reddit・Zenn・GitHub・HuggingFace・AgentSkillsHub）にまたがる共通テーマを1点以上
- スコア・いいね数・スター数などの数値を根拠として引用すること
- 日本語テックコミュニティ（Zenn）とグローバル（HN/Reddit）の視点の違いがあれば言及
- 各ポイントは具体的な記事名・リポジトリ名・モデル名を含めること（抽象的な言及は不可）

分析が完成したら、以下のスクリプトの `analysis_points` リストを実際の内容に差し替えてexecute_codeで実行する：

    import os
    from datetime import datetime

    # ↓ ここを実際の分析内容（3〜5点）に差し替える
    analysis_points = [
        "ポイント1: （具体的な内容）",
        "ポイント2: （具体的な内容）",
        "ポイント3: （具体的な内容）",
    ]

    workspace = os.path.expanduser("~/hermes-workspace")
    today = datetime.now().strftime("%Y-%m-%d")
    filepath = f"{workspace}/tech_topics_{today}.md"

    with open(filepath, "r", encoding="utf-8") as f:
        content = f.read()

    analysis_text = "\n".join(f"- {p}" for p in analysis_points)
    new_content = content.replace("PLACEHOLDER_ANALYSIS", analysis_text)

    with open(filepath, "w", encoding="utf-8") as f:
        f.write(new_content)
    print(f"ステップ2完了: 分析{len(analysis_points)}点をファイルに書き込みました")
    print(filepath)

---

## 実行トリガー例

- 「テックニュースまとめて」
- 「今週のHacker Newsどうだった？」
- 「AIの最新動向を教えて」
- 「テックトピックスを集めて」
- 「GitHubで今週何が話題？」
- 「HuggingFaceのトレンドモデルは？」
- 「最新のテック情報を集めてレポートにして」

## 出力先
~/hermes-workspace/tech_topics_YYYY-MM-DD.md

## 既知の制限
- Reddit は環境によって稀にブロックされることがある（その場合はスキップして他ソースで生成）
- HN のitem fetchは1件ずつのため数秒かかる
- GitHub API は未認証で60req/時（1日1回利用なら問題なし）
- X (Twitter) / Note.com は公開APIなしのため対象外
- AgentSkillsHub は Supabase anon key を使用（公開キー、有効期限 2088年）

## 関連スキル
- youtube-fetch-transcript: 動画トランスクリプト取得
- youtube-shorts: ショート動画アイデア生成
