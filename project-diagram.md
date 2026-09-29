project-diagram
プロジェクト全体を読んで構成図・依存図・処理フロー図を作り、アーキテクチャの良し悪しを根拠とコメント付きで判定してレポートにする。「プロジェクトを図にして」「構成を可視化」「アーキテクチャを評価して」のときに使う。

# プロジェクト全体図 ＋ アーキテクチャ評価

コードベース全体を読み、(1) 構造を図にし、(2) アーキテクチャの良し悪しを根拠とコメント付きで判定して、1 枚のレポートページにまとめる。

## 0. 進め方の原則
- **機械で数え、人が読む。** 依存関係・循環・規模はスクリプトで数える（推測で書かない）。役割・設計意図・問題の深刻度は、実際のコードを読んで判断する。
- 評価には必ず**根拠**（ファイル名・数値・該当箇所）を付ける。根拠のない「良い／悪い」は書かない。
- 良い点も書く。悪い点だけのレビューにしない。
- タスクリストを作る：①プロジェクトの取得 ②スキャン ③主要コードの読解 ④図の作成 ⑤評価 ⑥レポート公開 ⑦検証。

## 1. プロジェクトを手元に用意する
対象がどこにあるかで分ける（分からなければ一度だけ聞く）。
- **ユーザーの PC のフォルダ**：device 系ツールで接続済みフォルダを確認し、なければ `device_request_folder_access` でそのフォルダを要求する。`device_bash` があり `python3` が使えるなら、スクリプトを PC 側に置かずに `python3 - <path> < script` のように標準入力から流して実行する（ユーザーのフォルダにファイルを書かない）。使えない場合は、ソース一式を stage してワークスペースで実行する。
- **GitHub などの URL**：`git clone --depth 1` でワークスペースに取得する。
- **zip などの添付**：ワークスペースで展開する。

## 2. スキャンを実行する
付録のスクリプトを作業ディレクトリ（scratchpad）に `scan_project.py` として保存し、実行する。

```bash
python3 scan_project.py <project_dir>                     # Markdown の概要
python3 scan_project.py <project_dir> --json > scan.json  # 詳細な数値
python3 scan_project.py <project_dir> --focus src         # 1 つのグループが大きすぎるときに掘り下げる
python3 scan_project.py <project_dir> --depth 3           # グループの粒度を手動で指定
```

出力されるもの：言語別の規模、ディレクトリツリー、マニフェスト・CI・インフラ、エントリポイント候補、外部依存、**ディレクトリ単位の import グラフ**、評価用メトリクス（ファイル／グループ単位の循環依存、相互依存、不安定度 I=Ce/(Ca+Ce)、ハブファイル、巨大ファイル、テスト比率、孤立ファイル）、Mermaid 図のたたき台。

- 1 つのグループが全体の半分以上を占めるなら `--focus` で掘り下げ、「全体図」と「コア詳細図」の 2 段にする。
- import グラフを取れるのは Python / JS / TS / Go。それ以外の言語（Java, C#, Rust など）は、`import` / `using` / `use` を grep してグループ間の依存を手作業で数え、その旨をレポートに明記する。

## 3. 主要コードを読む（ここを省略しない）
スキャン結果を地図として、次を実際に開いて読む。目安は 10〜25 ファイル。大きなファイルは要所だけ読む。
1. README / ARCHITECTURE などのドキュメント（作者が意図した設計を知る）
2. マニフェスト（何のフレームワーク上に建っているか）
3. エントリポイント候補（起動から処理の入口まで）
4. ハブファイル上位（全体が何に依存しているか）
5. 最大ファイル上位（責務が集中していないか）
6. 循環依存に含まれるファイル（どこで循環が閉じているか）
7. 各グループの代表ファイル 1 つずつ（そのグループの責務を一言で言えるか）

大規模（ソース 300 ファイル超）の場合は、グループごとに Explore エージェントを並列で動かし、「このグループの責務・主要な型／関数・他グループとの関係」を要約させてよい。評価の判断そのものは自分で行う。

## 4. 図を作る（Mermaid）
最低 3 枚。ノードの説明は、ディレクトリ名ではなく**役割**で書く（例：`src/router` → 「ルーティング（URL→ハンドラ解決）」）。
1. **全体構成図**（flowchart）：外部アクター・外部サービス・DB・主要コンポーネントの関係。インフラ／CI から分かる実行環境も含める。
2. **モジュール依存図**（flowchart）：スキャンのグループ間エッジから作る。矢印の数値で依存の強さを示す。循環・相互依存・層の逆流は赤で強調する。15 ノード以下に保ち、多い場合はまとめるか 2 段にする。
3. **主要処理フロー**（sequenceDiagram）：代表的なユースケース 1 つ（例：HTTP リクエストが処理される流れ、CLI コマンドの実行）。実際のコードを読んで、関数名が合っていることを確認して書く。
必要に応じて、クラス継承（classDiagram）、データモデル（erDiagram）、デプロイ図を追加する。

## 5. アーキテクチャを評価する
### 評価軸と判定基準
各軸を **◎ 良い / ○ おおむね良い / △ 要改善 / × 問題あり** で判定し、「根拠」「コメント」「改善案」を書く。数値の目安は出発点にすぎないので、プロジェクトの種類・規模・目的（ライブラリ／アプリ／PoC）に合わせて調整し、調整した理由も書く。

| # | 評価軸 | 主に見るもの | 目安 |
|---|---|---|---|
| 1 | 責務分割・凝集度 | グループの責務を一言で言えるか、ディレクトリ構成と実際の役割が一致しているか | 一言で言えない／`utils`・`common`・`helpers` が肥大化 → △ |
| 2 | 依存の方向・レイヤリング | 下位層（utils, domain, core）が上位層（UI, controller, adapter）を import していないか。不安定度 I：安定させたい中核は I が 0 に近く、末端（adapter, CLI, app）は 1 に近いのが健全 | 下位→上位の逆流が 1 本でもあれば指摘する。本数が多ければ × |
| 3 | 循環依存 | ファイル単位・グループ単位の循環、相互依存 | グループ間の循環 → △〜×。`__init__` による再公開だけの循環は軽微として扱う |
| 4 | 結合度・ハブ | 被参照が突出したファイル、import 数の多いファイル | 被参照が多いだけなら問題ない（安定した型定義など）。**被参照も参照も多いファイル**は変更の影響が大きい → △ |
| 5 | サイズ・複雑さ | 500 行超・1000 行超のファイル、巨大な関数やクラス | 1000 行超があれば中身を読み、docstring の割合と責務の混在を確認して判断する |
| 6 | テスト | テスト／本番のファイル数と行数の比、テストが層ごとに揃っているか、CI でテストを実行しているか | 比率 0.3 未満、または CI でテストが走っていない → △ |
| 7 | 外部依存 | 依存の数と種類、同じ用途のライブラリの重複、フレームワークがドメイン層に漏れていないか | 重複や、コア層でのフレームワーク直接依存 → △ |
| 8 | 運用・開発体験 | README、設定の外出し（env）、Docker / IaC、CI、lint / format の設定 | 起動方法が README から分からない → △ |

### 総合判定
- 総合を **A（健全）/ B（おおむね健全・部分的な改善で十分）/ C（構造的な改善が必要）/ D（大幅な再設計を検討）** で付け、3 行以内で理由を書く。
- 「優先度付きの改善提案」を最大 5 つ挙げる。それぞれに：何を・なぜ・効果・手間（小／中／大）・対象ファイル。
- 「このプロジェクトの良い設計」を最低 2 つ挙げる。

### 判断の注意
- 静的解析は正規表現ベースなので、動的 import、DI コンテナ、プラグイン登録、リフレクションによる依存は拾えない。見つけたら本文で補足する。
- スクリプトは型だけの import（TS の `import type`、Python の `TYPE_CHECKING` ブロック）と、docstring・コメント内のコード例を除外して循環を数え、型専用の本数は別に出す。関数内の遅延 import は実行時依存として数えるので、循環を指摘する前に該当箇所を読み、どこで循環が閉じているか（遅延 import・再エクスポートなど）を確認して書く。
- TS の `import { type X }` のような、名前単位の型 import は実行時依存として数えられる。循環にそれが含まれていないか確認する。
- `examples/`、`benchmarks/`、`scripts/` は評価の中心から外し、本体（多くは `src/` やパッケージ名のディレクトリ）を中心に評価する。
- 評価は断定しすぎない。設計上のトレードオフだと読み取れる場合は、そう書く。

## 6. レポートを公開する
`artifact-design` スキルを読み込み、自己完結した 1 枚の HTML ページとして Artifact で公開する（ユーザーが Markdown や別の形式を指定した場合はそれに従う）。
構成：
1. 概要（何のプロジェクトか、技術スタック、規模を数字で示す）
2. 総合判定（A〜D ＋ 3 行の理由）と、評価軸ごとの判定一覧
3. 図（全体構成・モジュール依存・主要フロー、ほか）
4. 評価の詳細（軸ごとに、判定・根拠・コメント・改善案）
5. 優先度付きの改善提案と、良い設計
6. 付録：主要メトリクス、分析の範囲と限界

図は `<pre class="mermaid">` に Mermaid をそのまま書く（Artifact は Mermaid をネイティブに描画するので、ライブラリは読み込まない）。ノードのラベルに括弧・スラッシュ・`<`・日本語の記号などを含めるときは `"..."` で囲む。赤で強調するときは `classDef bad stroke:#c2410c,stroke-width:3px` と `linkStyle <番号> stroke:#c2410c,stroke-width:3px` を使う。
評価軸の判定は、色だけに頼らず ◎○△× の記号と文字（「要改善」など）を併記する。

## 7. 検証してから渡す
- 公開前に、全ての Mermaid 図の構文を確認する。ArtifactCheck などのプレビュー手段があればそれを使う。なければ Python の Playwright でページを開き、mermaid.min.js を `add_script_tag` で注入して、図ごとに `mermaid.parse()` を実行する（ワークスペースで npm や CDN に届かない場合は、`git clone --depth 1 https://github.com/badboy/mdbook-mermaid` の `src/bin/assets/mermaid.min.js` が使える）。スクリーンショットを 1 回撮って目視でも確認する。
- 評価で挙げたファイル名・数値が、スキャン結果や実際のコードと一致しているか照合する。
- 最後の返信は短く：総合判定、気になった点トップ 3、次にやるとよいこと 1 つ。

## 付録：scan_project.py
以下を作業ディレクトリに `scan_project.py` として保存して使う（Python 3.10 以上、標準ライブラリのみ）。

```python
#!/usr/bin/env python3
"""scan_project.py - プロジェクトを走査し、図とアーキテクチャ評価の材料を出力する（標準ライブラリのみ）。

出力: 構造ツリー / 言語別規模 / マニフェスト・CI・エントリポイント / 外部依存 /
      内部 import グラフ（Python・JS/TS・Go）をディレクトリ単位に集約 /
      評価用メトリクス（循環依存・不安定度・ハブ・巨大ファイル・テスト比率）/ Mermaid たたき台

使い方: python3 scan_project.py <dir> [--focus SUBDIR] [--depth N] [--json] [--max-files N]
"""
import argparse, json, os, re, sys
from collections import Counter, defaultdict
from pathlib import Path, PurePosixPath

IGNORE_DIRS = {".git", ".hg", ".svn", "node_modules", "__pycache__", ".venv", "venv", "env",
    ".tox", ".mypy_cache", ".pytest_cache", ".ruff_cache", "dist", "build", "out", "target",
    ".next", ".nuxt", ".svelte-kit", ".turbo", ".cache", "coverage", ".idea", ".vscode",
    "vendor", "Pods", ".gradle", ".terraform", "site-packages", ".dart_tool", "htmlcov"}
KEEP_DOT = {".github", ".gitlab", ".circleci"}
LANG = {".py": "Python", ".js": "JavaScript", ".jsx": "JavaScript", ".mjs": "JavaScript",
    ".cjs": "JavaScript", ".ts": "TypeScript", ".tsx": "TypeScript", ".mts": "TypeScript",
    ".go": "Go", ".rs": "Rust", ".java": "Java", ".kt": "Kotlin", ".rb": "Ruby", ".php": "PHP",
    ".cs": "C#", ".cpp": "C++", ".cc": "C++", ".c": "C", ".h": "C/C++", ".hpp": "C++",
    ".swift": "Swift", ".dart": "Dart", ".vue": "Vue", ".svelte": "Svelte", ".scala": "Scala",
    ".sql": "SQL", ".sh": "Shell", ".ex": "Elixir", ".lua": "Lua", ".r": "R"}
MANIFESTS = {"package.json", "pyproject.toml", "requirements.txt", "setup.py", "setup.cfg",
    "Pipfile", "go.mod", "Cargo.toml", "pom.xml", "build.gradle", "build.gradle.kts",
    "Gemfile", "composer.json", "pubspec.yaml", "tsconfig.json", "pnpm-workspace.yaml",
    "turbo.json", "nx.json", "lerna.json", "deno.json"}
INFRA = {"Dockerfile", "docker-compose.yml", "docker-compose.yaml", "compose.yaml", "compose.yml",
    "Makefile", "Procfile", "serverless.yml", "vercel.json", "netlify.toml", "fly.toml",
    "app.yaml", "schema.prisma", "alembic.ini"}
ENTRY_RE = re.compile(r"^(main|app|server|index|cli|manage|wsgi|asgi|__main__|App|Program)\.(py|js|mjs|ts|tsx|jsx|go|rs|java|kt|rb|php|cs)$")
DOC_RE = re.compile(r"^(README|ARCHITECTURE|CONTRIBUTING|DESIGN)(\..*)?$", re.I)
TEST_RE = re.compile(r"(^|/)(tests?|__tests__|spec|specs)(/|$)|(^test_.*\.py$)|(_test\.(py|go)$)|(\.(test|spec)\.[jt]sx?$)")
JS_EXTS = [".ts", ".tsx", ".js", ".jsx", ".mjs", ".cjs", ".mts", ".vue", ".svelte"]
MAX_BYTES = 1_500_000

PY_RE = re.compile(r"^[ \t]*(?:from[ \t]+(\.*[\w\.]*)[ \t]+import[ \t]+([\w\*, \t\(\)]+)|import[ \t]+([\w\.]+(?:[ \t]*,[ \t]*[\w\.]+)*))", re.M)
JS_RE = re.compile(r"""(?:import\s+(?:type\s+)?(?:[\w*{}\s,$]+\s+from\s+)?|export\s+[\w*{}\s,$]+\s+from\s+|require\(\s*|import\(\s*)['"]([^'"]+)['"]""")
GO_BLOCK_RE = re.compile(r"^import\s*\((.*?)\)", re.S | re.M)
GO_LINE_RE = re.compile(r'^import\s+(?:\w+\s+)?"([^"]+)"', re.M)
TC_RE = re.compile(r"^(\s*)if\s+(?:not\s+)?(?:t\.|typing\.|_?t\.)?TYPE_CHECKING\b")
TS_TYPE_RE = re.compile(r"""^\s*(?:import|export)\s+type\s[^;]*?from\s+['"][^'"]+['"];?""", re.M)
PY_DOC_RE = re.compile(r'(\"\"\"|\'\'\')[\s\S]*?\1')
JS_COMMENT_RE = re.compile(r"/\*[\s\S]*?\*/|^\s*//.*$", re.M)


def walk(root, max_files):
    files = []
    for dp, dns, fns in os.walk(root):
        dns[:] = sorted(d for d in dns if d not in IGNORE_DIRS and not d.endswith(".egg-info")
                        and not (d.startswith(".") and d not in KEEP_DOT))
        for f in sorted(fns):
            files.append(PurePosixPath(Path(dp, f).relative_to(root).as_posix()))
            if len(files) >= max_files:
                return files, True
    return files, False


def read(root, rel):
    p = root / rel
    try:
        return None if p.stat().st_size > MAX_BYTES else p.read_text(encoding="utf-8", errors="ignore")
    except OSError:
        return None


def nlines(root, rel):
    try:
        with open(root / rel, "rb") as fh:
            return fh.read(MAX_BYTES * 4).count(b"\n") + 1
    except OSError:
        return 0


def tree(files, max_depth=3, max_children=25):
    t = {}
    for f in files:
        node = t
        for part in f.parts[:-1]:
            node = node.setdefault(part + "/", {})
        node.setdefault("", []).append(f.name)
    def total(n):
        return len(n.get("", [])) + sum(total(v) for k, v in n.items() if k)
    out = []
    def rec(n, pre, d):
        dirs = sorted(k for k in n if k)
        for k in dirs[:max_children]:
            out.append(f"{pre}{k}  ({total(n[k])} files)")
            if d < max_depth:
                rec(n[k], pre + "  ", d + 1)
        if len(dirs) > max_children:
            out.append(f"{pre}... (+{len(dirs) - max_children} dirs)")
        if d <= 1:
            fs = sorted(n.get("", []))
            out.extend(pre + f for f in fs[:15])
            if len(fs) > 15:
                out.append(f"{pre}... (+{len(fs) - 15} files)")
    rec(t, "", 1)
    return out


def manifest_deps(root, rel):
    text, name, deps = read(root, rel) or "", rel.name, []
    try:
        if name == "package.json":
            d = json.loads(text)
            for k in ("dependencies", "devDependencies", "peerDependencies"):
                deps += list((d.get(k) or {}).keys())
        elif name == "requirements.txt":
            deps += [re.split(r"[<>=!~\[; ]", l.split("#")[0].strip())[0]
                     for l in text.splitlines() if l.strip() and not l.strip().startswith(("#", "-"))]
        elif name == "pyproject.toml":
            m = re.search(r"^dependencies\s*=\s*\[(.*?)\]", text, re.S | re.M)
            if m:
                deps += [re.split(r"[<>=!~\[; ]", s)[0] for s in re.findall(r"['\"]([^'\"]+)['\"]", m.group(1))]
            m = re.search(r"^\[tool\.poetry\.dependencies\](.*?)(?:^\[|\Z)", text, re.S | re.M)
            if m:
                deps += [x for x in re.findall(r"^([\w\-\.]+)\s*=", m.group(1), re.M) if x != "python"]
        elif name == "go.mod":
            deps += re.findall(r"^\s*([\w\.\-/]+\.[\w\.\-/]+)\s+v\d", text, re.M)
        elif name == "Cargo.toml":
            m = re.search(r"^\[dependencies\](.*?)(?:^\[|\Z)", text, re.S | re.M)
            if m:
                deps += re.findall(r"^([\w\-]+)\s*=", m.group(1), re.M)
        elif name == "Gemfile":
            deps += re.findall(r"^\s*gem\s+['\"]([^'\"]+)['\"]", text, re.M)
    except (ValueError, AttributeError):
        pass
    return sorted({d for d in deps if d})


class Resolver:
    def __init__(self, files, root):
        self.fileset = set(files)
        self.py_abs = defaultdict(list)      # import 可能な名前 -> files
        self.py_full = {}                    # リポジトリ相対の dotted -> file（相対 import 用）
        pkg_dirs = {str(f.parent) for f in files if f.name == "__init__.py"}
        for f in files:
            if f.suffix != ".py":
                continue
            parts = list(f.with_suffix("").parts)
            if parts[-1] == "__init__":
                parts = parts[:-1]
            if not parts:
                continue
            self.py_full[".".join(parts)] = f
            # パッケージ境界（__init__.py のあるディレクトリ）をさかのぼり、import 可能な名前を求める
            dirs = list(f.parent.parts)
            i = len(dirs)
            while i > 0 and "/".join(dirs[:i]) in pkg_dirs:
                i -= 1
            self.py_abs[".".join(parts[i:])].append(f)
        self.go_mod = None
        if (root / "go.mod").exists():
            m = re.search(r"^module\s+(\S+)", (root / "go.mod").read_text(errors="ignore"), re.M)
            self.go_mod = m.group(1) if m else None
        self.go_dirs = defaultdict(list)
        for f in files:
            if f.suffix == ".go":
                self.go_dirs[str(f.parent)].append(f)
        self.alias_roots = [p for p in ("src", "app", ".") if (root / p).is_dir()]

    def _py_lookup(self, dotted):
        c = self.py_abs.get(dotted)
        return sorted(c, key=lambda f: len(f.parts))[0] if c else None

    def resolve_py(self, src, module, names):
        if module.startswith("."):
            level = len(module) - len(module.lstrip("."))
            base = src.parent
            for _ in range(level - 1):
                base = base.parent
            bp = [p for p in base.parts if p not in ("", ".")]
            rest = module.lstrip(".")
            if rest:
                full = ".".join(bp + rest.split("."))
                hits = [self.py_full[full + "." + n] for n in names if full + "." + n in self.py_full]
                return (hits or ([self.py_full[full]] if full in self.py_full else [])), None
            hits = [self.py_full[".".join(bp + [n])] for n in names if ".".join(bp + [n]) in self.py_full]
            if not hits and ".".join(bp) in self.py_full:
                hits = [self.py_full[".".join(bp)]]
            return hits, None
        parts = module.split(".")
        for n in names:
            h = self._py_lookup(".".join(parts + [n]))
            if h:
                return [h], None
        for i in range(len(parts), 0, -1):
            h = self._py_lookup(".".join(parts[:i]))
            if h:
                return [h], None
        return [], parts[0]

    def _js_try(self, base):
        if base in self.fileset:
            return base
        for ext in JS_EXTS:
            for p in (PurePosixPath(str(base) + ext), base / ("index" + ext)):
                if p in self.fileset:
                    return p
        if base.suffix in (".js", ".jsx", ".mjs"):
            for ext in (".ts", ".tsx", ".mts"):
                p = PurePosixPath(str(base.with_suffix("")) + ext)
                if p in self.fileset:
                    return p
        return None

    def resolve_js(self, src, spec):
        if spec.startswith("."):
            h = self._js_try(PurePosixPath(os.path.normpath((src.parent / spec).as_posix())))
            return ([h] if h else []), None
        for alias in ("@/", "~/", "#/"):
            if spec.startswith(alias):
                for r in self.alias_roots:
                    h = self._js_try(PurePosixPath(os.path.normpath(f"{r}/{spec[len(alias):]}")))
                    if h:
                        return [h], None
                return [], None
        if spec.startswith(("node:", "http:", "https:", "$", "virtual:")):
            return [], None
        return [], ("/".join(spec.split("/")[:2]) if spec.startswith("@") else spec.split("/")[0])

    def resolve_go(self, spec):
        if self.go_mod and (spec == self.go_mod or spec.startswith(self.go_mod + "/")):
            fs = self.go_dirs.get(spec[len(self.go_mod):].lstrip("/") or ".")
            return ([fs[0]] if fs else []), None
        return [], (spec if "." in spec.split("/")[0] else None)


def split_type_only(text, ext):
    """(実行時コード, 型専用コード) に分ける。Python の TYPE_CHECKING ブロックと TS の import type。
    docstring・コメント内のサンプルコードは事前に除去する。"""
    if ext == ".py":
        text = PY_DOC_RE.sub("", text)
        run, typ, indent = [], [], None
        for line in text.splitlines():
            if indent is not None:
                cur = len(line) - len(line.lstrip())
                if not line.strip() or cur > indent:
                    typ.append(line); continue
                indent = None
            m = TC_RE.match(line)
            if m:
                indent = len(m.group(1)); typ.append(line); continue
            run.append(line)
        return "\n".join(run), "\n".join(typ)
    if ext in JS_EXTS:
        text = JS_COMMENT_RE.sub("", text)
        typ = "\n".join(TS_TYPE_RE.findall(text))
        return TS_TYPE_RE.sub("", text), typ
    return text, ""


def _parse(f, text, r):
    ext, found = f.suffix, []
    if ext == ".py":
        for m in PY_RE.finditer(text):
            if m.group(1) is not None:
                names = [n.split(" as ")[0].strip() for n in re.sub(r"[()]", "", m.group(2)).split(",")]
                found.append(r.resolve_py(f, m.group(1), [n for n in names if n and n != "*"]))
            else:
                for mod in m.group(3).split(","):
                    found.append(r.resolve_py(f, mod.strip().split(" as ")[0], []))
    elif ext == ".go":
        specs = GO_LINE_RE.findall(text)
        for b in GO_BLOCK_RE.findall(text):
            specs += re.findall(r'"([^"]+)"', b)
        found = [r.resolve_go(s) for s in specs]
    else:
        found = [r.resolve_js(f, s) for s in JS_RE.findall(text)]
    return found


def extract_edges(root, files, r):
    """戻り値: (実行時エッジ, 型専用エッジ, 外部 import カウント)"""
    edges, type_edges, external = set(), set(), Counter()
    stdlib = set(getattr(sys, "stdlib_module_names", ()))
    for f in files:
        if f.suffix not in (".py", ".go") and f.suffix not in JS_EXTS:
            continue
        text = read(root, f)
        if not text:
            continue
        run_text, type_text = split_type_only(text, f.suffix)
        for target, t in ((edges, run_text), (type_edges, type_text)):
            if not t:
                continue
            for hits, ext_name in _parse(f, t, r):
                target.update((f, h) for h in hits if h != f)
                if target is edges and ext_name and ext_name not in stdlib:
                    external[ext_name] += 1
    return edges, type_edges - edges, external


def group_of(f, depth):
    return "/".join(f.parts[:-1][:depth]) or "(root)"


def choose_depth(src, lo=5, hi=30):
    """グループ数が lo〜hi に収まり、かつ最大グループが全体の半分以下になる最も浅い深さ。"""
    best = 1
    for d in range(1, 7):
        c = Counter(group_of(f, d) for f in src)
        if len(c) > hi:
            break
        best = d
        if len(c) >= lo and c.most_common(1)[0][1] <= 0.5 * len(src):
            break
    return best


def sccs(nodes, adj):
    """Tarjan（反復版）。サイズ2以上の強連結成分 = 循環依存。"""
    idx, low, on, stack, out, counter = {}, {}, set(), [], [], [0]
    for s in nodes:
        if s in idx:
            continue
        work = [(s, iter(adj.get(s, ())))]
        idx[s] = low[s] = counter[0]; counter[0] += 1; stack.append(s); on.add(s)
        while work:
            v, it = work[-1]
            nxt = next(it, None)
            if nxt is not None:
                if nxt not in idx:
                    idx[nxt] = low[nxt] = counter[0]; counter[0] += 1
                    stack.append(nxt); on.add(nxt)
                    work.append((nxt, iter(adj.get(nxt, ()))))
                elif nxt in on:
                    low[v] = min(low[v], idx[nxt])
            else:
                work.pop()
                if work:
                    low[work[-1][0]] = min(low[work[-1][0]], low[v])
                if low[v] == idx[v]:
                    comp = []
                    while True:
                        w = stack.pop(); on.discard(w); comp.append(w)
                        if w == v:
                            break
                    if len(comp) > 1:
                        out.append(sorted(comp))
    return sorted(out, key=len, reverse=True)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("root")
    ap.add_argument("--depth", type=int, default=0)
    ap.add_argument("--json", action="store_true")
    ap.add_argument("--max-files", type=int, default=30000)
    ap.add_argument("--focus", default="", help="このディレクトリ配下だけをグループ化して掘り下げる（例: src）")
    a = ap.parse_args()
    root = Path(a.root).resolve()
    if not root.is_dir():
        sys.exit(f"not a directory: {root}")

    files, truncated = walk(root, a.max_files)
    lf, ll, src, sizes = Counter(), Counter(), [], {}
    for f in files:
        lang = LANG.get(f.suffix.lower())
        if lang:
            n = nlines(root, f)
            lf[lang] += 1; ll[lang] += n; src.append(f); sizes[f] = n
    tests = [f for f in src if TEST_RE.search(str(f))]
    test_set = set(tests)
    prod = [f for f in src if f not in test_set]

    manifests = [f for f in files if f.name in MANIFESTS or f.suffix in (".csproj", ".sln")]
    infra = [str(f) for f in files if f.name in INFRA or f.name.startswith("Dockerfile")
             or f.suffix == ".tf" or f.parts[0] in KEEP_DOT]
    docs = [str(f) for f in files if DOC_RE.match(f.name) and len(f.parts) <= 3]
    entries = [str(f) for f in files if ENTRY_RE.match(f.name) and len(f.parts) <= 4
               and not TEST_RE.search(str(f)) and not {"examples", "docs"} & set(f.parts)]

    r = Resolver(files, root)
    edges, type_edges, external = extract_edges(root, files, r)
    prod_set = set(prod)
    if a.focus:
        fp = a.focus.strip("/") + "/"
        prod_set = {f for f in prod_set if str(f).startswith(fp)}
        prod = [f for f in prod if f in prod_set]
    pedges = {(x, y) for x, y in edges if x in prod_set and y in prod_set}
    ptype = {(x, y) for x, y in type_edges if x in prod_set and y in prod_set}
    all_adj = defaultdict(set)
    for x, y in pedges | ptype:
        all_adj[x].add(y)
    cycles_incl_types = sccs(sorted(prod_set, key=str), all_adj)

    depth = a.depth or choose_depth(prod or src)
    gsize = Counter(group_of(f, depth) for f in prod)
    gedges = Counter((group_of(x, depth), group_of(y, depth)) for x, y in pedges
                     if group_of(x, depth) != group_of(y, depth))

    # 評価メトリクス
    ca, ce = Counter(), Counter()   # 被依存(afferent) / 依存(efferent) グループ数
    gadj = defaultdict(set)
    for (x, y) in gedges:
        ce[x] += 1; ca[y] += 1; gadj[x].add(y)
    instability = {g: round(ce[g] / (ca[g] + ce[g]), 2) for g in gsize if ca[g] + ce[g]}
    fadj = defaultdict(set)
    for x, y in pedges:
        fadj[x].add(y)
    file_cycles = sccs(sorted(prod_set, key=str), fadj)
    group_cycles = sccs(sorted(gsize), gadj)
    mutual = sorted({tuple(sorted((x, y))) for (x, y) in gedges if (y, x) in gedges})
    fan_in = Counter(y for _, y in pedges)
    fan_out = Counter(x for x, _ in pedges)
    big = sorted(((n, f) for f, n in sizes.items() if f in prod_set), reverse=True)
    orphans = [str(f) for f in prod if fan_in[f] == 0 and fan_out[f] == 0
               and f.suffix in ([".py", ".go"] + JS_EXTS) and f.name not in ("__init__.py",)]
    graph_supported = sum(1 for f in prod if f.suffix in [".py", ".go"] + JS_EXTS)
    prod_lines, test_lines = sum(sizes[f] for f in prod), sum(sizes[f] for f in tests)

    ids = {}
    def nid(g):
        return ids.setdefault(g, f"n{len(ids)}")
    mm = ["flowchart LR"]
    for g, n in gsize.most_common(40):
        mm.append(f'  {nid(g)}["{g}<br/>{n} files"]')
    for (x, y), c in gedges.most_common(80):
        if x in ids and y in ids:
            mm.append(f"  {ids[x]} -->|{c}| {ids[y]}")

    res = {
        "root": root.name, "focus": a.focus or None, "total_files": len(files), "truncated": truncated,
        "languages": [{"language": k, "files": v, "lines": ll[k]} for k, v in lf.most_common()],
        "docs": docs[:20], "manifests": [str(m) for m in manifests[:50]], "infra_ci": infra[:50],
        "entry_point_candidates": entries[:40],
        "declared_dependencies": {str(m): manifest_deps(root, m)[:60] for m in manifests[:20]},
        "external_imports_top": external.most_common(40),
        "group_depth": depth, "groups": gsize.most_common(60),
        "group_edges": [{"from": x, "to": y, "imports": c} for (x, y), c in gedges.most_common(150)],
        "metrics": {
            "source_files": len(src), "prod_files": len(prod), "test_files": len(tests),
            "prod_lines": prod_lines, "test_lines": test_lines,
            "test_to_prod_line_ratio": round(test_lines / prod_lines, 2) if prod_lines else None,
            "graph_supported_prod_files": graph_supported,
            "internal_file_edges_runtime": len(pedges),
            "internal_file_edges_type_only": len(ptype),
            "file_cycle_count_including_type_only_imports": len(cycles_incl_types),
            "largest_cycle_including_type_only": len(cycles_incl_types[0]) if cycles_incl_types else 0,
            "file_cycles": [[str(x) for x in c] for c in file_cycles[:15]],
            "file_cycle_count": len(file_cycles),
            "group_cycles": group_cycles[:10],
            "mutual_group_dependencies": mutual[:30],
            "group_instability_I=Ce/(Ca+Ce)": dict(sorted(instability.items(), key=lambda kv: kv[1])),
            "group_fan": {g: {"depends_on": ce[g], "depended_by": ca[g]} for g in gsize},
            "hub_files_most_imported": [(str(f), c) for f, c in fan_in.most_common(15)],
            "files_importing_most": [(str(f), c) for f, c in fan_out.most_common(15)],
            "largest_prod_files": [(str(f), n) for n, f in big[:15]],
            "files_over_500_lines": sum(1 for n, _ in big if n > 500),
            "files_over_1000_lines": sum(1 for n, _ in big if n > 1000),
            "isolated_files_sample": orphans[:20], "isolated_file_count": len(orphans),
        },
        "mermaid_draft": "\n".join(mm),
        "tree": tree(files),
    }
    if a.json:
        print(json.dumps(res, ensure_ascii=False, indent=2)); return

    m, o, fence = res["metrics"], [], "`" * 3
    o.append(f"# Project scan: {res['root']}\n- files scanned: {len(files)}" + (" (TRUNCATED)" if truncated else ""))
    o.append("\n## Languages")
    o += [f"- {l['language']}: {l['files']} files / {l['lines']} lines" for l in res["languages"]]
    for title, key in (("Docs", "docs"), ("Manifests", "manifests"), ("Infra / CI", "infra_ci"),
                       ("Entry point candidates", "entry_point_candidates")):
        o.append(f"\n## {title}"); o += [f"- {x}" for x in res[key]] or ["- (none)"]
    o.append("\n## Declared dependencies")
    o += [f"- {k}: {', '.join(v)}" for k, v in res["declared_dependencies"].items() if v] or ["- (none)"]
    o.append("\n## External imports (most used)\n" + (", ".join(f"{n}({c})" for n, c in res["external_imports_top"]) or "(none)"))
    o.append(f"\n## Module groups (prod code, depth={depth}{', focus=' + a.focus if a.focus else ''})")
    o += [f"- {g}: {n} files | depends_on={ce[g]} depended_by={ca[g]} I={instability.get(g, '-')}" for g, n in res["groups"]]
    o.append("\n## Group import edges (from -> to : count)")
    o += [f"- {e['from']} -> {e['to']} : {e['imports']}" for e in res["group_edges"][:60]]
    o.append("\n## Architecture metrics")
    o.append(f"- prod files {m['prod_files']} ({m['prod_lines']} lines) / test files {m['test_files']} ({m['test_lines']} lines) / test:prod line ratio {m['test_to_prod_line_ratio']}")
    o.append(f"- import-graph coverage: {m['graph_supported_prod_files']}/{m['prod_files']} prod files parsed (Python/JS/TS/Go only)")
    o.append(f"- internal file edges: runtime {m['internal_file_edges_runtime']} / type-only {m['internal_file_edges_type_only']} (type-only = TYPE_CHECKING / import type, excluded below)")
    o.append(f"- file-level cycles (runtime): {m['file_cycle_count']}  [if type-only imports included: {m['file_cycle_count_including_type_only_imports']}, largest {m['largest_cycle_including_type_only']} files]")
    o += [f"  - cycle({len(c)}): {' -> '.join(c[:8])}{' ...' if len(c) > 8 else ''}" for c in m["file_cycles"][:8]]
    o.append(f"- group-level cycles: {len(group_cycles)}")
    o += [f"  - {' <-> '.join(c)}" for c in group_cycles[:5]]
    o.append(f"- mutual group deps: {', '.join(f'{x}<->{y}' for x, y in mutual[:15]) or 'none'}")
    o.append(f"- files >500 lines: {m['files_over_500_lines']}, >1000 lines: {m['files_over_1000_lines']}")
    o.append(f"- isolated files (no internal imports in/out): {m['isolated_file_count']}")
    o.append("\n## Hub files (most imported)"); o += [f"- {f} ({c})" for f, c in m["hub_files_most_imported"]]
    o.append("\n## Files importing the most"); o += [f"- {f} ({c})" for f, c in m["files_importing_most"]]
    o.append("\n## Largest prod files"); o += [f"- {f} ({n})" for f, n in m["largest_prod_files"]]
    o.append(f"\n## Mermaid draft\n{fence}mermaid\n" + res["mermaid_draft"] + f"\n{fence}")
    o.append(f"\n## Tree\n{fence}"); o += res["tree"]; o.append(fence)
    print("\n".join(o))


if __name__ == "__main__":
    main()
```