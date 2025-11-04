# ソースコード詳細解説 (詳細版)

このドキュメントは `src/ollama_deep_researcher` ディレクトリ内のPythonコードについて、各関数の内部ロジック、データの流れ、モジュール間の具体的な連携を詳細に解説します。

## 1. 中核概念: 状態機械としてのリサーチエージェント

このアプリケーションの心臓部は、`graph.py` で定義された**状態機械（State Machine）**です。これは `LangGraph` フレームワークによって構築されています。

- **状態 (State):** `state.py` の `SummaryState` クラスが「状態」を定義します。これは、リサーチプロセス全体で引き回される情報の入れ物です。最初はほぼ空ですが、各ステップ（ノード）を経るごとに情報が追加・更新されていきます。
- **ノード (Node):** グラフの各ステップです。実体は `graph.py` にある関数（例: `generate_query`, `web_research`）です。各ノードは、現在の「状態」を引数として受け取り、特定の処理を実行し、更新された「状態」の一部を返します。
- **エッジ (Edge):** ノード間の繋がりです。あるノードの処理が終わった後、次にどのノードに進むかを定義します。

## 2. `SummaryState` 詳細: リサーチ旅行のパスポート

`state.py` で定義される `SummaryState` は、リサーチの全工程で使われる「パスポート」のようなものです。各ノード（関所）で新しい情報（スタンプ）が押されていきます。

```python
# state.py
@dataclass(kw_only=True)
class SummaryState:
    research_topic: str = field(default=None)       # ユーザーが指定したリサーチ主題
    search_query: str = field(default=None)         # LLMが生成した、または考察した検索クエリ
    web_research_results: Annotated[list, operator.add] = field(default_factory=list) # Web検索結果の生データ（要約済み）のリスト
    sources_gathered: Annotated[list, operator.add] = field(default_factory=list)     # 見つかった情報源（URLとタイトル）のリスト
    research_loop_count: int = field(default=0)     # リサーチの繰り返し回数
    running_summary: str = field(default=None)      # 各ステップで更新されていく要約文
```

`Annotated[list, operator.add]` は、このフィールドがリストであり、新しい値が追加される際には既存のリストに追記(`+`)されることを示します。

## 3. 実行フローのステップ・バイ・ステップ追跡

ユーザーがリサーチを開始すると、`SummaryState` オブジェクトが作成され、以下の旅が始まります。

### ステップ 1: `START` → `generate_query` ノード

1.  **入力:** `state.research_topic` (例: "人工知能の最新動向")
2.  **処理内容 (`generate_query` 関数内):**
    a.  `prompts.py` の `get_current_date()` で現在日付を取得します。
    b.  `prompts.py` の `query_writer_instructions` プロンプトテンプレートに、日付と `research_topic` を埋め込み、LLMへの指示を作成します。
    c.  `configuration.py` から設定を読み込み、`use_tool_calling` が `False` の場合、`json_mode_query_instructions` をプロンプトに追加します。これはLLMにJSON形式で出力するよう強制するための指示です。
    d.  `get_llm(configurable)` を呼び出し、設定に基づいたLLMクライアント（`ChatOllama` または `ChatLMStudio`）を取得します。この際、JSONモードが指定されていれば、LLMクライアントもJSONを出力する設定で初期化されます。
    e.  LLMに最終的なプロンプトを渡して実行 (`llm.invoke`) します。
    f.  LLMからの応答（JSON形式の文字列）を `json.loads()` でパースし、`"query"` キーの値を取得します。これが次のステップで使う検索クエリとなります。
    g.  LLMの応答が不正な形式だった場合などのために、フォールバックとして単純なクエリ (例: `f"Tell me more about {state.research_topic}"`) を用意しておき、失敗時にそれを使います。
3.  **出力:** `{"search_query": "(LLMが生成したクエリ)"}` という辞書を返します。LangGraphはこれを現在の `state` にマージし、`state.search_query` が更新されます。

### ステップ 2: `generate_query` → `web_research` ノード

1.  **入力:** `state.search_query`, `state.research_loop_count`
2.  **処理内容 (`web_research` 関数内):**
    a.  `configuration.py` から設定を読み込み、使用する検索API (`search_api`) を決定します (例: `duckduckgo`)。
    b.  決定されたAPIに対応する `utils.py` 内の関数（例: `duckduckgo_search`）を、`state.search_query` を引数として呼び出します。
    c.  `duckduckgo_search` 関数は、`duckduckgo-search` ライブラリを使ってWeb検索を実行し、結果（タイトル、URL、スニペット）のリストを返します。
    d.  `deduplicate_and_format_sources` 関数を使い、検索結果の重複をURLベースで除去し、LLMが読みやすいように整形された文字列 (`search_str`) を作成します。
    e.  `format_sources` 関数で、引用元リストとして表示するためのシンプルな文字列（タイトルとURLのリスト）も作成します。
3.  **出力:**
    - `sources_gathered`: 引用元リストの文字列が `state.sources_gathered` に追加されます。
    - `research_loop_count`: `state.research_loop_count` が `+1` されます。
    - `web_research_results`: 整形された検索結果 `search_str` が `state.web_research_results` に追加されます。

### ステップ 3: `web_research` → `summarize_sources` ノード

1.  **入力:** `state.running_summary` (初回はNone), `state.web_research_results` の最新の要素, `state.research_topic`
2.  **処理内容 (`summarize_sources` 関数内):**
    a.  `state.running_summary` が存在するかどうかで、LLMへのプロンプトを分岐させます。
        - **初回 (要約が存在しない場合):** 「このコンテキストを使って、トピックに関する要約を作成してください」という指示を生成します。
        - **2回目以降 (要約が存在する場合):** 「既存の要約を、この新しいコンテキストで更新・拡張してください」という指示を生成します。
    b.  `prompts.py` の `summarizer_instructions` をシステムプロンプトとして使用します。これは高品質な要約を生成するための詳細なガイドラインです。
    c.  LLMクライアントを取得し、システムプロンプトと上記の指示を渡して実行します。
    d.  LLMが生成した要約文を取得し、`strip_thinking_tokens` で余分なタグを除去します。
3.  **出力:** `{"running_summary": "(LLMが生成/更新した要約文)"}` を返し、`state.running_summary` が更新されます。

### ステップ 4: `summarize_sources` → `reflect_on_summary` ノード

1.  **入力:** `state.running_summary`, `state.research_topic`
2.  **処理内容 (`reflect_on_summary` 関数内):**
    a.  `prompts.py` の `reflection_instructions` をベースに、「専門家として現在の要約を分析し、知識のギャップと次の調査項目を特定せよ」というプロンプトを構築します。
    b.  `generate_query` と同様に、`use_tool_calling` の設定に応じてJSONモードの指示を追加します。
    c.  LLMに現在の要約 (`state.running_summary`) を渡し、「この情報に欠けている点は何か？それを埋めるための次の検索クエリは何か？」と問いかけます。
    d.  LLMからの応答（JSON形式）をパースし、`"follow_up_query"` キーの値を取得します。
3.  **出力:** `{"search_query": "(LLMが生成した次の検索クエリ)"}` を返し、`state.search_query` が新しいクエリで上書きされます。

### ステップ 5: `reflect_on_summary` → `route_research` (条件分岐エッジ)

このステップはノードではなく、次にどこへ進むかを決める**ルーター**です。

1.  **入力:** `state.research_loop_count`
2.  **処理内容 (`route_research` 関数内):**
    a.  `configuration.py` から `max_web_research_loops` の設定値を取得します。
    b.  `state.research_loop_count` が `max_web_research_loops` 以下であれば、`"web_research"` という文字列を返します。
    c.  そうでなければ、`"finalize_summary"` という文字列を返します。
3.  **出力:** 次に進むべきノードの名前 (`web_research` または `finalize_summary`)。

-   `web_research` に進む場合、**ステップ2**に戻り、新しい検索クエリで再びWeb検索からループが始まります。
-   `finalize_summary` に進む場合、ループは終了し、最終ステップに進みます。

### ステップ 6: `finalize_summary` ノード → `END`

1.  **入力:** `state.running_summary`, `state.sources_gathered`
2.  **処理内容 (`finalize_summary` 関数内):**
    a.  `state.sources_gathered` リストに蓄積された引用元情報（URLとタイトル）の重複を最終的に除去します。
    b.  最終的な要約文 (`state.running_summary`) と、整形された引用元リストを結合し、一つの完成したレポートを作成します。
    (例: `## Summary
(最終要約文)

### Sources:
* title1 : url1
* title2 : url2`)
3.  **出力:** `{"running_summary": "(完成した最終レポート)"}` を返し、`state.running_summary` が最終成果物で更新されます。グラフは `END` に到達し、処理を終了します。

## 4. 補助モジュールの深掘り

- **`utils.py`**: 検索APIを叩くだけでなく、`fetch_raw_content` でページのHTMLを取得して `markdownify` でMarkdownに変換するなど、LLMが扱いやすい形にデータを前処理する重要な役割を担っています。
- **`lmstudio.py`**: `ChatOpenAI` を継承し、`_generate` メソッドをオーバーライドしています。`format="json"` が指定された場合、リクエストパラメータに `"response_format": {"type": "json_object"}` を追加し、LM StudioにJSON出力を強制します。さらに、LM Studioの出力が純粋なJSONでない場合（例: テキストが前後に付加されている）、正規表現や文字列検索でJSON部分だけを抽出する後処理も行なっており、堅牢性を高めています。

この詳細な解説で、コードの内部で何が起こっているか、より明確にご理解いただけたのであれば幸いです。