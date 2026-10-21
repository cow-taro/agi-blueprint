# 外部 AI エージェント管理ルールの調査（採用候補の棚卸し）

作成日: 2026-10-16。対象: agi-blueprint に取り入れられそうな、外部で評価・採用されている AI エージェント管理ルール。

## 0. 調査の前提と限界（必読）

- 本調査は **AI モデルの訓練時知識に基づく**。本ワークスペースから外部ネットワークへはアクセスしていない。各仕組みの実装詳細・対応ツール・名称は変動するため、**採用前に必ず公式ドキュメントで実在と現行仕様を確認する**（doc-first の精神）。
- 確度の表記: ◎＝仕組みの存在と骨子に高い確信あり / ○＝存在は確実だが細部に不確実さあり。
- 本文中の「本テンプレート」は agi-blueprint を指す。

## 1. 実在する主要な仕組み（フレームワーク別・実名）

### Cursor Rules（確度 ◎）

`.cursor/rules/` 配下の `.mdc` ファイルで規約を管理。適用方式が 4 種類に分かれているのが特徴:

| 適用方式 | 挙動 |
|---|---|
| Always Apply | 常時ロード（本テンプレートの rules 相当） |
| Auto Attached | glob に一致するファイルを触るときだけ自動ロード |
| Agent Requested | フロントマッターの description を AI が読み、必要と判断したときだけロード（本テンプレートの skills 相当） |
| Manual | 明示的に参照されたときだけ |

### AGENTS.md（確度 ◎）

リポジトリルートに置く `AGENTS.md` という共通指示ファイルの標準。OpenAI Codex が採用し、他の主要ツールも対応が進んでいる（対応ツール一覧は要確認）。リポジトリルート＋サブディレクトリへの階層配置ができ、モノリポ／マルチパッケージで指示を分割できる。

### Claude Code（Anthropic 公式・確度 ◎）

- `CLAUDE.md`: リポジトリルート・サブディレクトリ・ホーム (`~/.claude/CLAUDE.md`) の階層配置。`@パス` による他ファイルの取り込み (import) に対応。
- プランモード: ソースを変更させず計画だけ作らせるモード（本テンプレートの計画フェーズ相当）。
- 権限機構: ツールごとの allow / deny リストと権限モード（計画のみ・確認あり・編集自動承認・全許可）。
- サブエージェント (`.claude/agents/`): 独自のコンテキストと指示を持つ専門エージェントに調査を分離し、結論だけ返させる。
- 公式ベストプラクティス: 「CLAUDE.md は簡潔に保つ（コンテキストの全トークンが判断に影響する）」「運用しながら反復チューニングする」「PRD を TODO.md へ展開し P1/P2/P3 の優先度とチェックボックスで進捗管理する」「並列タスクには git worktree を使う」等。

### Cline / Roo Code コミュニティの Memory Bank パターン（確度 ◎・名称は ○）

`memory-bank/` ディレクトリに構造化 md を置き、セッション開始時に読み、作業後に更新する儀式 (ritual) を回すパターン。知られている構成例: プロジェクト概要・システム構造・技術文脈・**現在の作業文脈 (activeContext)**・**進捗 (progress)** の各 md。

### GitHub Spec Kit（確度 ◎）

「constitutional workflow」と呼ぶ spec 駆動開発の公式フレームワーク。**constitution.md（プロジェクトの憲章＝不変原則）** を核に、specify → plan → tasks → implement の各段階の成果物 md を生成する。憲章は AI の判断の上位制約として機能する。

### AWS Kiro（確度 ◎）

spec 駆動 IDE。requirements.md（EARS 形式）→ design.md → tasks.md を段階的に生成し、実装は tasks に縛られる。

### Aider（確度 ◎）

- `CONVENTIONS.md`: プロジェクト規約を読ませる仕組み。
- **repo map**: ツリーシッターで解析した「ディレクトリ＋主要シンボル」の索引を、トークン予算に収まるよう圧縮して渡す。生コード全文ではなく**索引で案内させる**発想。

### llms.txt 提案（Answer.AI・確度 ◎）

サイト/ドキュメントの先頭に、H1 ＋ 要約 blockquote ＋ ファイル一覧（1 行説明つきリンク）を置き、LLM に効率的に読ませる標準形式の提案。

### Anthropic の context engineering 記事（確度 ◎）

「最小限の高シグナルなコンテキスト」「just-in-time な取得（必要なときに必要な分だけ）」「長時間タスクでの compaction と構造化メモへの書き出し (agentic memory)」「独立した調査はサブエージェントへ分離」等の原則を公式に提示。

### Roo Code（確度 ◎・本プロジェクトで使用中）

`.roo/rules/`（常時ロード）＋ modes（モード別指示）＋ `.roo/skills/`（frontmatter の description で AI が判断してロード）＋ orchestrator モード（タスク分解）。本テンプレートの直接の母体。

### その他（確度 ○・参考）

- Windsurf: `.windsurfrules` による規約ファイル。
- GitHub Copilot: `.github/copilot-instructions.md`。
- BMAD-METHOD: 役割別エージェント（PM・アーキテクト・開発・QA）とプランゲート・Definition of Done チェックリストで回す人気フレームワーク。
- SuperClaude: ペルソナとトークン効率化を売りにするフレームワーク（詳細は要確認）。

## 2. 本テンプレートが既に等価以上で持つもの

| 外部の仕組み | 本テンプレートの対応 | 判定 |
|---|---|---|
| Claude Code プランモード / Cursor Plan モード | `rules/01-workflow-gates.md` の「1. フェーズフロー」「5. モード選択とリスク」 | 等価（ツール非依存の形で明文化済み） |
| Spec Kit の constitution.md | `rules/00-governance-core.md` の「0. 最上位目的（羅針盤）」 | 等価 |
| Anthropic の「最小限の高シグナル・just-in-time 取得」 | `rules/00-governance-core.md` の「1. 思想: 知識量から論理性へ」「4. 生コード垂れ流しの禁止と等値一致ダイレクトロード」 | 等価以上（参照の一意性まで踏み込んでいる） |
| Memory Bank の activeContext / progress | `templates/work-log-skeleton.md` の「現在状態」とタイムライン | ほぼ等価（更新儀式も「更新ルール」に明記済み） |
| Agent Requested rules / Roo skills | `skills/` の frontmatter description 方式 | 等価 |
| Kiro / Spec Kit の段階的成果物 md | plan md ＋ `templates/desired-red-gate-checklist.md` ＋ テスター用 md | 等価（テスト先行ゲートは本テンプレートが上） |
| CLAUDE.md の簡潔性原則 | `rules/00-governance-core.md` の「6. 繰り返す失敗のエスカレーション」（常時 rules は最小限） | 等価（v1.1.0 で追加） |

## 3. 採用候補（R1〜R8・導入先つき）

| ID | 候補 | 出典の仕組み | 導入先 | 効果 |
|---|---|---|---|---|
| R1 | **パススコープ規約**: 対象パスの glob（例: `frontend/**`）に一致するファイルを触るときだけ自動適用される規約ファイル | Cursor Auto Attached / AGENTS.md のサブディレクトリ階層 | `SETUP-CHECKLIST.md` B 章へ新項目（例: B7「パススコープ rules」） | 常時コンテキストを増やさずに領域別規約（フロント・インフラ等）を運用できる |
| R2 | **セッション開始時の儀式**: 作業開始時に、該当タスクの work log と前提ファイルを必ず読む | Memory Bank の「start every task by reading memory bank」 | `rules/01-workflow-gates.md` の「2. doc-first」へ一文 | セッションをまたいだ文脈喪失・ハルシネーションを防ぐ |
| R3 | **権限マトリクス**: エージェントが絶対実行禁止のコマンド群と承認必須操作を明文化 | Claude Code の allow / deny リスト | `SETUP-CHECKLIST.md` B 章へ新項目（例: B8「権限マトリクス」） | 「人が GUI で行う操作」「破壊的コマンド」の禁止が設定値として一元化される |
| R4 | **repo map（コード索引 md）**: ディレクトリ構成＋主要シンボルの一覧 md を正本として持たせ、生コードの代わりに索引で案内する | Aider の repo map | `rules/00-governance-core.md` の「4. 生コード垂れ流しの禁止と等値一致ダイレクトロード」へ補完条文 | 大規模コードベースで「どこに何があるか」の特定をフルスキャンなしで済ませる |
| R5 | **llms.txt 形式の索引**: README 各ファイルに 1 行要約（blockquote 形式）を持たせ、機械可読の入口とする | llms.txt 提案 | トップ `README.md`（既に役割表があるため、要約行の形式統一程度で済む） | エージェントが初回に読む入口の効率が上がる |
| R6 | **TODO 展開儀式**: 実装開始時、承認済み plan md のタスクを優先度つき TODO に展開し、完了ごとにチェックする | Claude Code ベストプラクティスの PRD → TODO.md（P1/P2/P3） | `rules/01-workflow-gates.md` の「1. フェーズフロー」へ一文 | 長い実装の途中脱線・作業漏れを防ぐ |
| R7 | **羅針盤変更の人間承認の明示**: 憲章（羅針盤）の変更は AI からは提案のみで、必ず人間承認を要する | Spec Kit の constitution 運用 | `rules/00-governance-core.md` の「0. 最上位目的（羅針盤）」へ一文 | 最上位目的が AI の便宜で書き換えられる事故を防ぐ |
| R8 | サブエージェント・git worktree による並列分離 | Claude Code サブエージェント / ベストプラクティス | 将来候補（導入先: `rules/01-workflow-gates.md` か新規 skill） | 独立した調査を別コンテキストで回し、結論だけ主コンテキストへ戻す。ツール依存が強い見送り可 |

## 4. 見送り推奨と理由

- **BMAD-METHOD の役割別エージェント編成**: 本テンプレートの「計画と実行の分離＋承認ゲート」で同等の統制が得られ、役割を増やすと常時コンテキストと複雑性が増すため。
- **SuperClaude 等のペルソナ系**: ペルソナの性情はトークンを消費し、決定論的統治の思想と相性が悪いため。
- **Cursor 依存の .mdc 形式そのもの**: ツール非依存のプレーン md（本テンプレートの方式）の方が移植性が高いため。

## 5. 判定記録（2026-10-16）

- ユーザー判断により、**R1〜R8 は現時点で不採用**。本テンプレートの既存機構（rules / skills / templates ＋ Roo Code / Zoo Code の機能）で同等以上が達成できるため。
- 本 survey は調査記録として残す。将来の再検討時は次節の手順に従う。

## 6. 採用手順の推奨

1. 採用する候補（R1〜R8）を決める。
2. **各候補の出典ツールの公式ドキュメントで現行仕様を確認する**（本調査は訓練時知識ベースのため）。
3. `SETUP-CHECKLIST.md` 該当章へ項目を追加し、`skills/setup-wizard/SKILL.md` の質問に組み込む。
4. 日本語版（正本）へ先に入れ、英語版へ同一内容を翻訳反映する（トップ `README.md` の「保守ルール」）。
5. トップ `README.md` 最下部の改訂メタデータへ追記する。
