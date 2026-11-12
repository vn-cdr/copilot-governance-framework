# GitHub Copilot AIガバナンス設計書（GHEC・約5,000ユーザー／約5,000リポジトリ）

ファイル名案：`copilot-ai-governance-ghec.md`（作成基準日：2026年11月11日）

**結論：5,000人規模のGHECでCopilotを安全に展開する最適解は、「Enterprise（AI Controls）で既定値を締め、managed-settings.json・エージェントファイアウォール・ルールセット・予算で“技術的に強制”し、利用者には少数の“標準”への同意だけを求める」ガードレール型ガバナンスです。** これはGartnerが2026年のEnterprise AI Coding Agents Magic Quadrantで示した「エージェント導入はツール購入ではなく運用モデルの決定であり、生成・コミット・CI/CDの各段階でゼロトラスト型の統制と自動検証が前提」という立場と一致し、申請・例外処理を最小化したいという運用要件とも両立します。\[1\]

## TL;DR

- **最優先は“既定値の固定”**：2026年10月22日以降、未設定のGA機能・MCPポリシー・Copilot code reviewは「Default policy for new features」に従って自動的に有効化され得るため、Enterpriseレベルで既定を「Disabled」にし、承認済み機能・モデルだけを明示的に有効化する。モデルも「default availability for released models」を無効にして手動承認制にする。
- **エージェント系は“別の統制面”として扱う**：コンテンツ除外はIDEのEdit/Agentモード、Copilot CLI、クラウドエージェントでは効かず、パブリックコード一致ブロックもクラウドエージェントには適用されない。代わりに managed-settings.json（bypass無効化・MCP許可リスト）、組織レベルのエージェントファイアウォール、ルールセット（必須レビュー・CODEOWNERS）、ワークフロー承認で境界を作る。
- **GHASなしでも“最低限の防御線”は組める**：Dependabot alerts／security updates／version updates、依存関係グラフ、ルールセット、無料のsecret risk assessment、クラウドエージェント組込みのCodeQL・シークレット検査は追加ライセンス不要。ただしプライベートリポジトリのpush protection・secret scanning・code scanningは有償（Secret Protection $19、Code Security $30／アクティブコミッター／月）なので、少なくともSecret Protectionは高リスク領域から優先導入を推奨する。

---

## 1. エグゼクティブサマリー

本書は、GitHub Enterprise Cloud（GHEC、GHESは対象外）上で約5,000人の開発者が約5,000のプライベートリポジトリに対してGitHub Copilotを利用する前提で、Gartnerの公開情報を軸にAIガバナンスを設計し、NIST・OWASP・ISOとの対応、具体的なGHEC設定、整備すべき文書体系までを一貫して示すものです。

設計思想は次の3点です。

1. **「ブロックではなく安全な既定路」**：Gartnerは2026年4月28日付のプレスリリース「Gartner Identifies Six Steps to Manage AI Agent Sprawl」で、Senior Director AnalystのMax Gossの発言として「多くの組織がAIエージェントの利用を禁止・制限するが、それは長期的な解決策ではない。承認されたツールで仕事ができなければ、従業員は組織の統制を迂回してシャドーAIを使い始める可能性が高く、はるかに大きなリスクになる」と警告している。したがって承認済みCopilotを“最も使いやすい道”として提供し、他の生成AIの迂回利用を減らす。
2. **「技術的強制＞手続き的統制」**：5,000人規模で申請ベースの統制は破綻する。設定で強制できるもの（モデル、MCP、ファイアウォール、ブランチ保護、予算）は強制し、文書は“ユーザー責任の標準”として短く保つ。
3. **「エージェントは別リスク」**：補完・チャットと、エージェントモード／CLI／クラウドエージェントでは効く統制が異なる。段階導入で後者を遅らせ、ガードレールを整えてから開放する。

## 2. 前提と対象範囲

| 項目 | 前提 |
|---|---|
| プラットフォーム | GitHub Enterprise Cloud（GHE.comデータレジデンシーを含み得る）。GHESは対象外 |
| 規模 | 約5,000ユーザー、約5,000リポジトリ |
| リポジトリ可視性 | 数名の管理者を除き一般ユーザーはprivateのみ作成・利用可 |
| Copilotプラン | Copilot Business または Copilot Enterprise |
| GHAS | Secret Protection／Code Securityは未導入の可能性あり → 「あり／なし」両方を併記 |
| 承認済み生成AI | 限定的。Copilot以外のAIツールは原則対象外 |
| 運用方針 | 申請・管理負荷を抑え、ユーザー責任の「標準」中心 |

## 3. Gartnerベストプラクティスの要点

Gartnerの本文レポート（有料）は直接確認できないため、以下は**Gartner公式プレスリリース、Gartner Peer Insights、レポートを引用した報道・ベンダー公開記事**からの整理です。出典の確度を「A：Gartner公式」「B：報道によるレポート引用」「C：ベンダー経由の二次情報」で示し、Gartnerの記述と本書の解釈・推測を区別します。

### 3.1 市場認識の変化（アシスタント → エージェント）

- **（B）** Gartnerは2026年5月20日に初の「Magic Quadrant for Enterprise AI Coding Agents」を公表し、Anthropic、Cursor、GitHub、OpenAIをLeaderに位置付けた。Virtualization Review（David Ramel、2026年6月5日）によれば、ChallengerはAWS、Google、Alibaba Cloud、Cognition、VisionaryはTabnineのみ、Niche PlayersはAtlassian、BytePlus、JetBrainsで、2025年のAI Code Assistants MQでLeaderだったAWSとGoogleはChallengerに移った。従来の「AI Code Assistants」MQから市場定義そのものを変更している。
- **（B）** 同レポートはエンタープライズAIコーディングエージェントの必須機能として、自律タスク実行、反復的検証と自己修正、人間による監督・トレーサビリティ・監査性、利用分析、エンタープライズ統制とデータ保護、そしてネイティブMCPサポートを挙げ、ベースモデルが顧客コードで学習されない保証も要件としている。\[1\]
- **（A）** Gartner Peer Insightsのカテゴリ定義でも、必須機能として「エージェントが生成した変更に対する人間のレビューと承認の仕組み、エージェントの行動・起動者・判断・ツール使用の詳細ログ」が明記されている。\[2\]

### 3.2 ガバナンスに関する推奨（本書の設計原則に直結）

| # | Gartnerの推奨・見解 | 確度 | 本書での具体化 |
|---|---|---|---|
| G1 | エージェントの選定はツール購入ではなく「運用モデルの決定」。高品質なコンテキスト供給、生成・コミット・CI/CDでのゼロトラスト統制、出力の自動検証ができる領域に限って導入すべき\[1\] | B | 段階導入（§6）、ルールセット・ワークフロー承認（§8） |
| G2 | エンジニアリング・アーキテクチャ・セキュリティ・法務の横断タスクフォースで評価・パイロット・統治する\[3\] | B（2025 MQ報道） | AIガバナンス委員会とRACI（§6） |
| G3 | 若手のスキル・経験パラドックスに対応し、AIが基礎能力を代替しないよう構造化されたアップスキリングを行う\[3\] | B | 教育資料・レビュー基準（§7） |
| G4 | 生産性は記述行数・受入率・タスク単位指標だけで測らず、デリバリーフロー、品質、開発者体験、ビジネス価値といったチーム単位の成果と結び付ける\[1\] | B | KPI設計（§9） |
| G5 | 2028年までにAIコーディングのコストが平均的な開発者給与を上回ると予測し、コストガバナンス、ワークロードのスロットリング、利用制御が必要\[1\] | B | 予算・ULB設計（§8.6） |
| G6 | AIエージェントのスプロール対策6ステップ：エージェントのガバナンスとポリシー（誰が作り共有できるか、許可されるコネクタ）、集中的なエージェント台帳、AI情報ガバナンス等。2026年4月28日付Gartnerプレスリリースによれば、適切なエージェントガバナンスがあると考える組織は13%のみで、2028年までに平均的なグローバルFortune 500企業は15万以上のエージェントを使用すると予測 | A（2026年4月28日） | MCP許可リスト、カスタムエージェント管理、台帳化 |
| G7 | 禁止・制限は長期解ではなく、シャドーAIを誘発する | A | 「安全な既定路」方針（§1） |
| G8 | ガバナンス委員会の設置、AI TRiSM（プロンプトフィルタリング・リダクション等）への予算配分\[4\] | A（2025年2月） | 委員会設置、DLP/監査ログ連携 |
| G9 | Gartnerの2025年7月1日付プレスリリース「Top Strategic Trends in Software Engineering for 2025 and Beyond」（VP Analyst Joachim Herschmann）によれば、2028年までにエンタープライズのソフトウェアエンジニアの90%がAIコードアシスタントを使う（2024年初頭は14%未満）。\[5\]リスク管理のための強力なガードレールの実装を推奨\[5\] | A（2025年7月） | 全社展開を前提とした設計 |
| G10 | エージェントを日常のIDEワークフローに組み込み、厳格な人間による検証を徹底する。構造化されたプロンプトエンジニアリング教育を行い、過度の依存を防ぐ\[2\] | A（Peer Insightsのピア・レッスン） | 開発者向けガイド（§7） |

### 3.3 期待値の調整（生産性・コスト）

- **（B）** GitHub公式ブログ（2026年MQ発表記事）が引用するGartnerの予測では、2028年までに非同期AIコーディングエージェントのワークフローがソフトウェアエンジニアリングチームの生産性を30〜50%向上させ、2025年のAIコードアシスタントによる0〜20%の向上を上回るとされる。
- **（B）** Visual Studio Magazine（2025年9月17日）が引用した2025年Magic Quadrant for AI Code Assistantsによれば、エンジニアリングスタッフの42%がAIによる生産性向上は1〜10%にとどまると回答し、さらに12%は向上なしと回答している。

**本書の解釈**：Gartnerの論点は一貫して「価値はあるが、ガバナンス・検証・コスト管理が伴わなければ手戻りとリスクが増える」です。したがってKPIは導入率より“品質を落とさずにフローが改善したか”を主軸に置きます。

## 4. 参照標準とCopilotリスクのマッピング

### 4.1 標準の位置付け

| 標準 | Copilotガバナンスでの使い方 |
|---|---|
| NIST AI RMF 1.0（AI 100-1） | Govern／Map／Measure／Manageの4機能で管理サイクルを構成\[6\] |
| NIST AI 600-1（生成AIプロファイル、2024年7月26日） | 12の生成AIリスク（情報セキュリティ、データプライバシー、知的財産、作話、価値連鎖・コンポーネント統合、人間とAIの構成等）をリスク台帳の分類軸に使う\[7\]\[8\] |
| NIST SP 800-218（SSDF 1.1） | PO／PS／PW／RVの実践を「AI生成コードを含むSDLC」に適用（中核） |
| NIST SP 800-218A | 本来はAIモデルの開発者・統合者向け。Copilot“利用者”には類推適用とし、主にサプライヤー評価の観点で参照\[9\] |
| OWASP Top 10 for LLM Applications 2025 | LLM01〜LLM10でCopilotの脅威を列挙（利用者視点に読み替え） |
| OWASP Top 10 for Agentic Applications 2026（ASI01〜ASI10、2025年12月9日公表） | エージェントモード／CLI／クラウドエージェント／MCPの脅威分析\[10\]\[11\] |
| ISO/IEC 42001:2023 | 附属書Aの管理策領域（A.2 AI方針、A.3 内部組織、A.4 資源、A.5 影響評価、A.6 ライフサイクル、A.7 データ、A.9 AIシステムの利用、A.10 第三者関係）を文書体系の骨格に |
| ISO/IEC 23894:2023 | AIリスクマネジメント（ISO 31000準拠）のプロセスをリスク評価手順書に反映 |
| ISO/IEC 27001:2022 | 附属書A 5.10（許容される利用）、5.19／5.23（供給者・クラウド）、6.3（意識向上）、8.4（ソースコードへのアクセス）、8.12（データ漏えい防止）、8.15／8.16（ログ・監視）、8.25／8.28（セキュア開発・コーディング）、8.32（変更管理）を既存ISMSへ統合 |

### 4.2 リスク×標準マッピング表

| ID | Copilot利用リスク | NIST AI 600-1 | AI RMF | OWASP LLM 2025 | OWASP Agentic 2026 | SSDF | ISO/IEC 42001 | ISO/IEC 27001:2022 |
|---|---|---|---|---|---|---|---|---|
| R1 | 機密情報・シークレットのプロンプト経由漏えい | データプライバシー、情報セキュリティ | MAP 4／MANAGE 2 | LLM02 | ASI03 | PO.5、PS.1 | A.7、A.10 | 5.10、8.12 |
| R2 | 脆弱なコードの生成・混入 | 情報セキュリティ、作話 | MEASURE 2 | LLM05、LLM09 | ASI05 | PW.7、PW.8、RV.1 | A.6 | 8.25、8.28 |
| R3 | ライセンス・著作権（パブリックコード一致） | 知的財産 | GOVERN 1／MAP 4 | — | — | PW.4 | A.10 | 5.32 |
| R4 | プロンプトインジェクション（リポジトリ内容・Issue・PRコメント・Web・MCP結果経由） | 情報セキュリティ | MEASURE 2.7 | LLM01 | ASI01、ASI06 | PW.1 | A.6 | 8.28 |
| R5 | エージェントの過剰な権限・自律実行 | 人間とAIの構成 | GOVERN 3／MANAGE 2 | LLM06 | ASI02、ASI03、ASI10 | PO.5 | A.6、A.9 | 8.2、8.4 |
| R6 | MCPサーバー・プラグイン経由のデータ流出／サプライチェーン | 価値連鎖・コンポーネント統合 | GOVERN 6 | LLM03 | ASI04、ASI07 | PW.4、PO.1 | A.10 | 5.19、5.23 |
| R7 | サードパーティモデル利用（保持・学習・所在地） | データプライバシー、価値連鎖 | GOVERN 6 | LLM03 | ASI04 | PO.1 | A.10 | 5.23 |
| R8 | コスト暴走（AI Credits・高価格モデル・エージェント多用） | — | MANAGE 4 | LLM10 | ASI08 | — | A.4 | 5.1 |
| R9 | シャドーAI（未承認ツール・個人アカウント） | 価値連鎖 | GOVERN 1.6 | — | ASI10 | PO.1 | A.2、A.9 | 5.10、6.3 |
| R10 | 過信・スキル劣化・レビュー形骸化 | 人間とAIの構成、作話 | GOVERN 2／3 | LLM09 | ASI09 | PO.2 | A.4、A.9 | 6.3 |
| R11 | 監査証跡不足（誰が・どのエージェントで・何をしたか不明） | — | GOVERN 4／MEASURE 3 | — | ASI10 | PO.4 | A.6 | 8.15、8.16 |

## 5. リスクシナリオと対策（リスク→管理策→GitHub設定→ガイドライン条項）

| ID | 具体シナリオ | 管理策 | GitHub設定（GHASなし） | GHASありの追加 | ガイドライン条項 |
|---|---|---|---|---|---|
| R1 | .envや設定ファイル中のAPIキーをチャットに貼る／エージェントが読み取る | 機密データ分類、コンテンツ除外、エージェントの読み取り禁止ルール | コンテンツ除外（補完・チャット向け）、managed-settings.jsonの`permissions.deny`で`Read(./.env)`等、クラウドエージェントはcopilot環境に登録したシークレットのみ渡す、無料のsecret risk assessmentで現状把握\[12\]\[13\] | Secret Protection：push protection、secret scanning、AI検出シークレット、委任バイパス\[14\] | 利用ガイドライン「入力禁止情報」条項 |
| R2 | 生成コードにSQLインジェクション等が混入 | 人間レビュー必須、CI検証 | ルールセットで必須レビュー・必須ステータスチェック・CODEOWNERS、Copilot code review、クラウドエージェント組込みのCodeQL・シークレット検査（GHAS不要）\[15\] | Code Security：CodeQLコードスキャン、Copilot Autofix、dependency review\[16\] | レビューガイド「AI生成コードの受入基準」 |
| R3 | GPLコードと一致する提案を取り込む | パブリックコード一致のブロック、コード参照の確認 | 「Suggestions matching public code」＝Blocked（IDE補完・チャット向け）。クラウドエージェントには効かないため、エージェントPRはレビューで確認\[12\]\[17\] | — | 利用ガイドライン「ライセンス」条項 |
| R4 | Issue本文やPRコメントに仕込まれた指示でエージェントが外部送信 | 入力を信頼しない、外部通信制限 | クラウドエージェントのファイアウォール（組織で強制）、カスタム許可リストは組織のみ、MCP許可リスト\[18\] | Code Security（エージェント出力の追加検査） | 開発者ガイド「信頼できない入力」条項 |
| R5 | エージェントモードで確認なしにシェル実行・破壊的操作 | 最小権限、承認ゲート | `permissions.disableBypassPermissionsMode: "disable"`、`permissions.deny`に破壊的コマンド、クラウドエージェントは書込み権限者のみ起動・単一ブランチ（copilot/）のみpush・ワークフローは承認制\[19\]\[20\]\[21\] | — | 利用ガイドライン「エージェント利用」条項 |
| R6 | 野良MCPサーバーがソースを外部へ送る | MCP許可リスト、登録審査 | 「MCP servers in Copilot」有効＋managed-settings.jsonの`allowedMcpServers`／`deniedMcpServers`、必要ならRegistry only\[22\] | — | MCP登録標準 |
| R7 | 保持条件が異なるモデル（例：Anthropicが安全分類器目的でデータを保持するClaude Fable 5、オープンウェイトモデル）を無自覚に使う\[17\]\[23\] | モデル承認制 | 「default availability for released models」を無効、モデルごとに明示承認 | — | 承認モデル一覧 |
| R8 | 一部ユーザーが高価格モデルとエージェントで月数十万円消費 | 予算・ULB | ユニバーサルULB、コストセンターULB、「Stop usage when budget limit is reached」有効化\[24\] | — | 利用ガイドライン「コスト」条項 |
| R9 | 個人のCopilot Pro／他社AI IDEで社内コードを扱う | 承認ツールの明示、EMUによる分離 | EMU、IdP連携、IP許可リスト、PAT・OAuthアプリポリシー | — | AI利用ポリシー「承認ツール」条項 |
| R10 | AI生成PRを読まずに承認 | 依頼者とは別の承認者、教育 | クラウドエージェントPRは依頼者が承認不可（標準仕様）、ルールセットで承認者数\[20\]\[25\] | — | レビューガイド |
| R11 | 事故時にエージェント操作を追えない | 監査ログ保全 | 監査ログストリーミング、エージェント関連監査イベント、利用状況メトリクスAPI | セキュリティ概要 | 管理者運用手順書 |

## 6. ガバナンス体制・RACI・段階導入ロードマップ

### 6.1 体制

- **AIガバナンス委員会**（Gartner G2／G8に対応、ISO/IEC 42001 A.3）：CISO（議長）、開発本部、アーキテクチャ、法務・知財、調達、人事・教育。四半期開催、モデル・機能の承認判断を担う。
- **AIマネージャー**（GitHub上の実務管理者）：エンタープライズカスタムロールに「Manage enterprise AI controls」「Read enterprise audit logs」「View Enterprise Copilot Metrics」を付与し、Enterprise Ownerを増やさずにAI管理を委任する（GitHub公式の推奨構成）。\[26\]
- **Copilotチャンピオン**：各事業部の開発者50〜100名程度（約1〜2%）。一次問合せ・教育・フィードバック収集。

### 6.2 RACI

| 活動 | 委員会 | AIマネージャー | Enterprise Owner | Org Owner | リポジトリ管理者 | 開発者 | セキュリティ | 法務 |
|---|---|---|---|---|---|---|---|---|
| AI利用ポリシー策定・改定 | A | R | C | I | I | I | C | C |
| モデル・機能の承認 | A | R | C | I | I | I | C | C |
| AI Controls設定・managed-settings.json管理 | I | R/A | C | I | — | — | C | — |
| MCPサーバー許可リスト審査 | I | R | — | C | — | 申請 | A | C |
| ライセンス割当（IdPグループ） | I | A | R | C | — | — | — | — |
| ルールセット・ブランチ保護 | I | C | A | R | R | — | C | — |
| AI生成コードのレビュー | — | — | — | — | A | R | C | — |
| 予算・ULB設定 | A | R | C | I | — | — | — | — |
| インシデント対応 | I | R | C | C | C | 報告 | A | C |
| 教育・同意取得 | A | R | — | C | — | R | C | C |

（R=実行、A=説明責任、C=協議、I=報告）

### 6.3 段階導入ロードマップ

| フェーズ | 期間目安 | 対象 | 開放する機能 | 出口条件 |
|---|---|---|---|---|
| 0：基盤整備 | 1〜2か月 | 管理者 | EMU/IdP、AI Controlsの既定値、予算、監査ログストリーミング、ルールセット、managed-settings.json | 設定チェックリスト（§8）完了 |
| 1：パイロット | 2〜3か月 | 約200〜300名（複数事業部・若手/ベテラン混在） | IDE補完、IDEチャット（Ask/Edit）、github.comチャット、Copilot code review | 品質指標の悪化なし、インシデントゼロ、コスト実績 |
| 2：拡大 | 3〜4か月 | 約2,000名 | ＋エージェントモード（bypass無効）、Copilot CLI（パイロット群のみ） | KPI達成、教育完了率95%以上 |
| 3：全社 | 以降 | 約5,000名 | ＋承認済みMCP、クラウドエージェントは対象リポジトリをカスタムプロパティで限定 | 四半期レビューで継続判断 |
| 4：エージェント拡張 | 適宜 | 選定チーム | 自動化（automations）、カスタムエージェント、サードパーティエージェント | 個別リスク評価 |

## 7. 作成すべきドキュメント一覧（目次案・主要条項案）

文書は**「1ポリシー＋3標準＋4手順＋2補助」の10本**に限定し、開発者が読む必要があるのは①②③の3本（合計A4で10ページ以内）に抑えます。

| # | 文書 | 読者 | 性格 | 対応標準 | 主要条項・目次案 |
|---|---|---|---|---|---|
| ① | 生成AI利用ポリシー（全社・Copilot章） | 全員 | ポリシー（同意必須） | ISO/IEC 42001 A.2、27001 5.10 | 適用範囲／承認済みAIツールは会社付与のCopilot（EMU）のみ、個人契約のCopilot Free/Pro/Pro+/Maxや未承認AI IDE・エージェントでの業務コード利用禁止／AI生成物の責任は利用者とレビュアー／入力禁止情報の分類／監視と記録の明示／違反時の扱い・年1回改定 |
| ② | GitHub Copilot利用ガイドライン（標準） | 開発者 | 標準（同意必須） | 42001 A.9、SSDF PO.2 | フェーズ別の使用可能機能／承認モデル一覧（既定は「Auto」）／入力禁止情報／エージェント標準（bypass禁止、破壊的操作は人が確認）／カスタム指示（`.github/copilot-instructions.md`、`AGENTS.md`、組織指示）にシークレット・内部URLを書かない／コード参照表示時は不採用か法務確認／個人予算と到達時の挙動（補完は継続、チャット・エージェントは停止）／報告先\[24\] |
| ③ | AI生成コード・プロンプト安全ガイド／レビュー基準 | 開発者・レビュアー | 標準 | SSDF PW.7、OWASP LLM01/05 | AI生成PRも同一基準でレビュー、クラウドエージェントPRは依頼者以外が承認／`.github/workflows/`変更は「Approve and run workflows」前に差分を読む／Issue・PRコメント・Web・MCP結果は信頼できない入力／追加依存パッケージの実在性確認（パッケージ幻覚対策）／テスト・静的解析の通過を受入条件\[19\]\[27\] |
| ④ | MCP・プラグイン・カスタムエージェント登録標準 | 開発者・AIマネージャー | 標準 | OWASP ASI04、42001 A.10 | 許可リスト方式、申請はフォーム1枚（提供元・通信先・扱うデータ・認証方式）／リモートはURL固定、ローカルはコマンド固定／審査基準（社外送信、OAuthスコープ、保守主体、バージョン固定）／年1回棚卸し |
| ⑤ | Copilot管理者運用手順書 | AIマネージャー・Owner | 手順 | 27001 8.15/8.32 | 設定ベースライン（§8）とIaC（managed-settings.jsonはPRレビュー必須）／IdP連動のライセンス割当・90日非アクティブで剥奪／新機能・新モデル評価（Changelog監視→リスク評価→パイロット→承認）／予算監視／四半期ドリフト検知 |
| ⑥ | 例外管理手順 | 管理者 | 手順 | 42001 A.6 | 例外は期限付き・自動失効（個別ULBの有効期限機能を使用）／例外種別は(a)予算引き上げ、(b)MCP追加、(c)クラウドエージェント対象リポジトリ追加の3種のみ、他は委員会判断\[24\] |
| ⑦ | AI関連インシデント対応手順 | セキュリティ | 手順 | AI RMF MANAGE 4、27001 5.24-5.28 | シークレット入力・コミット（即時ローテーション→監査ログ確認）／プロンプトインジェクション疑い（ULBを$0にして課金機能を即時ブロック、セッションログ保全、MCP見直し）／ライセンス違反疑い（該当コード除去、法務連携）／ベンダー起因（モデル無効化）\[24\] |
| ⑧ | 教育プログラム | 全員 | 補助 | 27001 6.3、Gartner G3/G10 | 全員必須30分＋理解度テスト80%以上で合格者にのみライセンス付与（同意取得を兼ねる）／レビュアー向け60分／若手向けにAIなしで基礎課題を解くトラック |
| ⑨ | KPI定義書・ダッシュボード仕様 | 委員会 | 補助 | AI RMF MEASURE | §9参照 |
| ⑩ | AIリスク台帳・影響評価 | 委員会 | 手順 | ISO/IEC 23894、42001 A.5 | §4.2を初版とし、年1回と重大変更時に更新 |

## 8. 必要なGitHub設定一覧

凡例：**GHAS**欄は「不要」＝追加ライセンス不要、「SP」＝Secret Protection必要、「CS」＝Code Security必要。設定名はUI表記に準拠しつつ、2026年に「AI Controls」へ再編されたため表記揺れがあり得ます。

### 8.1 Enterprise：アイデンティティ・アクセス

| 設定項目 | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Enterprise Managed Users（EMU） | 採用（新規構築時。既存の場合は移行計画を検討） | 管理対象アカウントは企業内のprivate/internalリポジトリにしか貢献できず、外部へのフォークも不可。\[28\]個人アカウント混在によるシャドーAI・流出を構造的に防ぐ | 27001 5.15/5.16、42001 A.9 | 不要 |
| SAML/OIDC SSO＋SCIM | OIDC（Entra ID）＋条件付きアクセス、IdPグループでCopilotライセンスを管理 | 退職者の即時無効化、申請レスのライセンス管理 | 27001 5.18 | 不要 |
| IP許可リスト | 社内・VPN・承認SaaSのみ | 端末外からのアクセス制限 | 27001 8.20 | 不要 |
| Personal access tokens | classic PATは制限、fine-grained PATは承認制・最長有効期限を設定 | エージェント・MCPに渡るトークンの最小化 | OWASP ASI03 | 不要 |
| OAuth app／GitHub Appポリシー | OAuthアプリアクセス制限を有効、Appの導入はOwner承認 | サードパーティ連携の統制 | 27001 5.19 | 不要 |
| 監査ログストリーミング | SIEMへストリーミング（Git イベント含む） | エージェント操作の追跡 | AI RMF GOVERN 4、27001 8.15 | 不要 |
| AIマネージャー用カスタムロール | Manage enterprise AI controls／Read enterprise audit logs／View Enterprise Copilot Metrics\[26\] | Owner権限の過剰付与回避 | 42001 A.3 | 不要 |

### 8.2 Enterprise：リポジトリ管理ポリシー

| 設定項目 | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Repository creation | 「Members can create repositories」で**Privateのみ**チェック（EMUではpublicは選択肢自体がない）\[29\]\[30\] | 前提条件の強制 | 27001 8.4 | 不要 |
| Block the creation of user namespace repositories（EMU） | 有効 | 個人名前空間へのコード持ち出し防止 | 27001 8.12 | 不要 |
| Repository visibility change | Enterprise Ownerのみ\[30\] | private以外への変更防止 | 27001 8.4 | 不要 |
| Repository forking | 同一組織内のみ、または禁止 | コピー拡散の防止 | 27001 8.12 | 不要 |
| Repository deletion and transfer | Ownerのみ | 証跡保全 | 27001 8.13 | 不要 |
| Base permissions | No permission（チーム単位で付与） | 最小権限 | 27001 5.15 | 不要 |
| カスタムプロパティ | `data_classification`、`copilot_agent_allowed`、`criticality`を必須プロパティとして定義 | ルールセット・クラウドエージェント対象選定に使う\[29\] | 42001 A.7 | 不要 |

### 8.3 Enterprise／Organization：ルールセット（ブランチ保護）

| 設定項目 | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Enterpriseルールセット（全リポジトリのデフォルトブランチ） | Restrict deletions、Block force pushes、Require a pull request before merging（承認1名以上、Dismiss stale approvals、Require review from Code Owners）、Require status checks to pass | AI生成コードを含むすべての変更に人間レビューとCIを強制 | SSDF PW.7、27001 8.32、OWASP LLM06 | 不要（private＋GHECで利用可） |
| 高重要度リポジトリ（`criticality=high`） | 承認2名、Require conversation resolution、Require signed commits | 重要資産の強化 | SSDF PS.1 | 不要 |
| CODEOWNERS | `.github/workflows/`、`.github/copilot-instructions.md`、`AGENTS.md`、`.github/agents/`、`copilot-setup-steps.yml`にプラットフォームチームを指定 | エージェントの挙動を変えるファイルの改ざん防止 | OWASP ASI04/ASI06 | 不要 |
| Copilot自動化（shared automation）PRの追加承認 | 既定（追加承認あり）を維持\[31\] | 自動化起点の変更に人間の目を追加 | OWASP ASI09 | 不要 |
| Copilot code reviewの自動レビュー（ルールセット） | フェーズ1から有効化を検討（AI Credits＋Actions分数を消費） | 人間レビューの補助 | SSDF PW.7 | 不要 |

### 8.4 Enterprise：AI Controls（Copilotポリシー）

| 設定項目（AI Controls内） | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Default policy for new features | **Disabled**（2026年10月22日以降、未設定のGA機能はこの値に従う）\[32\] | 新機能の自動有効化を防ぎ、承認制にする。プレビュー機能は引き続きオプトイン\[32\] | 42001 A.6、AI RMF GOVERN 1 | 不要 |
| default availability for released models | **Disabled**（手動承認） | 新モデルの自動開放を防ぐ。オープンウェイトモデルやデータ保持契約外モデルは元々自動有効化対象外\[23\] | OWASP LLM03、42001 A.10 | 不要 |
| 個別モデルポリシー | 承認済みモデルのみEnabled。高価格モデルは「Optional」としてエンタープライズチーム単位で付与（プレビュー）\[33\] | コスト・データ取扱いの統制 | AI RMF MANAGE | 不要 |
| Suggestions matching public code | **Blocked** | GitHub Copilot Business製品ページのFAQによれば、フィルタは65レキシーム以上（平均150文字）の一致を検査し、ブロック有効時に一致が検出されると提案はユーザーに表示されない。IP補償の条件の一つ | NIST AI 600-1 知的財産 | 不要 |
| Copilot CLI | フェーズ2でパイロット群のみ（Let organizations decide→対象Orgのみ）\[34\] | シェル実行を伴うためガードレール整備後 | OWASP ASI02/ASI05 | 不要 |
| GitHub Copilot app | Copilot CLIと同時期まで「Disabled everywhere」 | 2026年7月から専用ポリシー化。\[35\]エージェントセッションを伴う | OWASP ASI02 | 不要 |
| Copilot in the cloud（クラウドエージェント） | フェーズ3で対象Orgのみ有効、さらに組織側で対象リポジトリを限定\[36\] | 自律エージェントの段階導入 | OWASP ASI03/ASI10 | 不要 |
| サードパーティコーディングエージェント／Agent apps | Disabled（承認済み生成AIが限定的なため） | 未承認AIの経路を閉じる | 42001 A.10 | 不要 |
| Copilot code review | Enabled（予算設定後） | レビュー支援。Actions分数も消費する点に注意\[37\] | SSDF PW.7 | 不要 |
| MCP servers in Copilot | Enabled＋許可リスト（下記） | 無効化は利便性を下げシャドー化を招くため、許可リスト方式を推奨（GitHub公式推奨）\[38\] | OWASP ASI04 | 不要 |
| Restrict MCP access to registry servers | 原則Allow all＋managed-settings.jsonの許可リストを採用。自前レジストリを運用する場合のみRegistry only | レジストリ方式はパブリックプレビューで推奨手段ではない。\[39\]Registry onlyではCopilot connectorsが使えなくなる\[40\] | OWASP ASI04 | 不要 |
| Editor preview features／プレビュー機能 | Disabled（パイロット群のみ評価） | プレビューはプレリリース条項 | 42001 A.6 | 不要 |
| Copilot Memory、Copilot Spaces、Web検索、Connectors | 個別評価。初期はDisabled | 情報の持続・外部送信の評価が必要 | OWASP ASI06 | 不要 |
| AI credits paid usage | Enabled（既定）だが必ず予算と併用\[41\] | 無効だと共有プール枯渇時に停止。\[24\]業務継続と費用上限のバランス | — | 不要 |

### 8.5 Enterprise managed settings（`.github-private`リポジトリの`copilot/managed-settings.json`）

GitHub公式は、MCP許可リストの推奨手段としてこのファイルを挙げています。設定はCopilotライセンスを受けた全ユーザーに、`.github-private`へのアクセス有無にかかわらず適用されます。公式は可視性をinternalにして全員が内容を読めるようにすることを推奨していますが、本環境の「一般ユーザーはprivateのみ」という前提を守るなら、private（AIマネージャーのみ編集可）とし、内容を利用ガイドラインに転記して公開する運用を推奨します。\[36\]\[38\]\[42\]

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "deny": [
      "Read(./.env)",
      "Shell(git push --force *)",
      "Shell(rm -rf *)",
      "Shell(terraform destroy *)"
    ]
  },
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.internal.example.co.jp/*" }
  ],
  "deniedMcpServers": [],
  "forceRemoteSettingsRefresh": true,
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "ORG/copilot-plugins" }
  ]
}
```

**仕様上の注意点**：

- `allowedMcpServers`が未設定ならすべての非既定サーバーが許可され、空配列ならすべてブロックされる。`deniedMcpServers`は常に優先する。複数ソースの許可リストは積集合になり、JSONが不正だと空の許可リストとして扱われる（フェイルクローズ）。\[43\]\[44\]\[45\]
- `allowedMcpServers`／`deniedMcpServers`は、リポジトリのカスタムエージェントプロファイルで定義されたMCPサーバーには適用されない。対策として、組織のファイアウォールで起動に必要なドメインを許可しない。\[46\]
- キーごとに対応クライアント（Copilot CLI、VS Code、Copilot app、JetBrains、クラウドエージェント）が異なる。GitHubは2026年7月にクラウドエージェントとCopilot appを対象に加えたと発表しているが、二次情報では未対応とする表もあるため、公式リファレンスの対応表を導入時に必ず確認すること。\[39\]\[45\]\[47\]
- MCPポリシーはCopilot Business／Enterpriseのシートにのみ適用され、Cursor・Claude等のサードパーティホストは統制できない。だからこそ「承認ツール以外の禁止」をポリシー①で明文化する。\[48\]\[49\]\[50\]

### 8.6 Enterprise：課金・予算（GitHub AI Credits）

2026年6月1日から全CopilotプランがGitHub AI Creditsによる使用量課金に移行しました。Copilot Businessは$19/ユーザー/月で1,900クレジット、Copilot Enterpriseは$39で3,900クレジットを含み、Enterprise内でプールされます。\[24\]\[37\]\[51\]

| 設定項目 | 推奨値 | 理由 | 根拠 | GHAS |
|---|---|---|---|---|
| Universal user-level budget | 例：含有額の1.5倍程度（Business想定で$30） | ULBは常にハードストップ。1人の暴走を防ぐ\[24\] | Gartner G5、OWASP LLM10 | 不要 |
| Cost center user-level budget | 事業部ごとに設定（重度利用チームは高め） | 数千件の個別予算を作らずに差を付ける\[24\] | — | 不要 |
| Individual user-level budget | 例外時のみ、有効期限付き | 例外の自動失効\[24\] | — | 不要 |
| Cost center／Organization／Enterprise budget | 設定し、**「Stop usage when budget limit is reached」を必ず有効化**（既定オフ） | 既定のままでは上限超過後も課金が続く\[24\] | — | 不要 |
| ライセンス付与経路 | 1ユーザー1組織（またはエンタープライズ直接・コストセンター） | 複数組織から付与すると、請求組織が毎月ランダムに選ばれ予算統制が崩れる\[24\] | — | 不要 |
| 予算見積り | エンタープライズ予算はライセンス料を上限に含まない点を前提に設計\[24\] | 例：400ライセンスのBusinessは月$7,600。$5,000の強制予算を設定しても合計は$12,600まで達し得る\[24\] | — | 不要 |

**本書の推奨**：5,000名でCopilot Businessの場合、ライセンス料は月約$95,000、含有クレジットは月約950万（$95,000相当）です。まずパイロットで1人あたり消費の分布を取り、ULBを中央値の2〜3倍に置くのが現実的です。2026年10月発表の「GitHub EnterpriseにCopilotアクセスを含める」モデルは含有クレジットがなく初回利用から従量課金になるため、利用量の実測なしに移行するのは避けるべきです。\[24\]\[52\]

### 8.7 Organization：Copilot・クラウドエージェント

| 設定項目 | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Copilotライセンス割当 | IdP同期チームに割当（セルフサーブ申請は不要） | 運用負荷低減 | — | 不要 |
| コンテンツ除外（Organization） | `**/.env*`、`**/secrets/**`、`**/*.pem`、`**/*.key`、顧客データサンプル等 | 補完・チャットの文脈から除外（エージェント系には効かない）\[53\]\[54\] | OWASP LLM02 | 不要 |
| 組織カスタム指示 | セキュアコーディング規約の要点、禁止事項（ハードコード禁止等） | 全リポジトリへ一貫した指示。現時点ではクラウドエージェントとcode reviewに適用\[55\] | SSDF PW.5 | 不要 |
| ファイアウォール：Enable firewall | **Enabled**（リポジトリ判断に委ねない）\[56\] | 外部通信の制限によるデータ流出・プロンプトインジェクション対策\[18\] | OWASP ASI01/ASI02 | 不要 |
| Recommended allowlist | Enabled（パッケージレジストリ等）\[34\] | 開発に必要な通信のみ | — | 不要 |
| Organization custom allowlist | 社内パッケージレジストリ、社内ドキュメントのみ\[18\] | 例外を組織で一元管理 | — | 不要 |
| Allow repository custom rules | **Disabled** | リポジトリ管理者による許可リスト拡張を禁止\[57\] | OWASP ASI04 | 不要 |
| クラウドエージェントのワークフロー承認 | 既定（承認必須）を維持し、自動実行を許可しない\[19\]\[27\] | エージェントが書いたワークフローが秘密情報にアクセスするのを防ぐ\[27\] | OWASP ASI05 | 不要 |
| copilot環境のシークレット | 原則登録しない。必要時は読み取り専用・短命トークン | エージェントはcopilot環境に登録したシークレット・変数のみ取得\[12\]\[25\] | OWASP ASI03 | 不要 |
| クラウドエージェント組込みのセキュリティ検証 | 有効（既定）。無効化は禁止 | CodeQL・シークレットスキャン・依存関係分析・Copilot code reviewによるセルフチェック。GHASライセンス不要\[12\]\[15\]\[20\]\[58\]\[59\] | SSDF PW.7/PW.8 | 不要 |
| 対象リポジトリ | カスタムプロパティ`copilot_agent_allowed=true`のみ（REST APIで選択）\[36\] | 段階導入・高機密リポジトリの除外 | 42001 A.6 | 不要 |

### 8.8 Organization／Enterprise：GitHub Actions・リポジトリテンプレート

| 設定項目 | 推奨値 | 理由 | 根拠標準 | GHAS |
|---|---|---|---|---|
| Allowed actions and reusable workflows | 「Allow enterprise, and select non-enterprise」＋GitHub作成・認証済み作成者のみ＋許可リスト | サプライチェーン対策 | SSDF PW.4、OWASP LLM03 | 不要 |
| アクションのSHAピン留め | 要求（ポリシーで強制可能な場合は強制）※UI名称は要確認 | タグ改ざん対策 | SSDF PS.2 | 不要 |
| Default workflow permissions | Read repository contents（読み取りのみ） | GITHUB_TOKENの最小権限 | OWASP ASI03 | 不要 |
| Allow GitHub Actions to create and approve pull requests | 無効 | 自動承認の防止 | SSDF PW.7 | 不要 |
| OIDC | クラウド認証は長期シークレットではなくOIDCを使用 | シークレット削減 | 27001 8.24 | 不要 |
| リポジトリテンプレート | `.github/copilot-instructions.md`（規約・テスト方法・シークレット記載禁止）、`AGENTS.md`、最小限の`copilot-setup-steps.yml`（ファイアウォール適用外）、`CODEOWNERS`\[46\]\[60\] | 全リポジトリへの一貫配布 | SSDF PW.5、OWASP ASI05/ASI06 | 不要 |

### 8.9 セキュリティ機能：GHASあり／なしの比較

| 機能 | GHASなし（プライベート） | GHASあり | 備考 |
|---|---|---|---|
| Dependency graph・SBOMエクスポート | 利用可 | 利用可 | 全プランで無料\[61\] |
| Dependabot alerts／security updates／version updates | 利用可 | 利用可 | GitHub提供の既定自動トリアージルールも無料\[61\] |
| リポジトリルールセット・Artifact attestations | 利用可（GHEC） | 利用可 | — |
| Secret risk assessment | 無料（全リポジトリ・アーカイブ含む、ポイントインタイム）\[62\]\[63\]\[64\] | 利用可 | 現状把握に最適\[64\] |
| Code security risk assessment | 無料（最大20リポジトリ、90日ごとに再実行可）\[65\] | 利用可 | — |
| Secret scanning alerts・push protection（private） | **不可**\[64\]\[66\] | SP | ユーザー向けpush protectionは**publicリポジトリ向けのみ**でprivateは守らない\[61\]\[67\] |
| AI検出シークレット・カスタムパターン・委任バイパス | 不可 | SP | — |
| CodeQLコードスキャン・Copilot Autofix | 不可 | CS | — |
| Dependency review action・カスタム自動トリアージルール | 不可 | CS | — |
| クラウドエージェント組込みのCodeQL／シークレット検査 | **利用可**（エージェント生成コードのみ） | 利用可 | GHASライセンス不要 |

**GHASなしの代替策**：(1) pre-commitフック＋OSSのシークレット検出ツールを開発端末に配布、(2) CIでOSSのSASTやシークレット検出を必須ステータスチェックとして実行、(3) 無料のsecret risk assessmentを四半期ごとに実行、(4) 予算が付けば、`data_classification=confidential`以上のリポジトリから優先してSecret Protectionを有効化する（アクティブコミッター課金のため、対象を絞れば費用を抑えられる）。**本書の推奨**：Copilotによりコード生成量が増えるとシークレット混入の機会も増えるため、Secret\[68\] Protectionは「あれば良い」ではなく第一優先の追加投資と位置付けます。

## 9. モニタリング・KPI・監査

### 9.1 データソース

- **Copilot usage metrics（2026年2月27日GA）**：エンタープライズ／組織のダッシュボードとREST API。2026年中に、チーム単位（user-teamsレポート、API限定）、リポジトリ単位（クラウドエージェントPRの作成・マージ数、code reviewの提案数）、ユーザー別AI Credits消費量（`ai_credits_used`）、クライアント別利用（`totals_by_client`）、サードパーティエージェント別利用、code reviewの対応率が順次追加された。\[69\]\[70\]\[71\]\[72\]\[73\]\[74\]\[75\]
- **監査ログ**：エージェント関連の監査イベント、ポリシー変更、ライセンス変更をSIEMへストリーミング。
- **課金API・Budgets**：コストセンター別消費、予算到達アラート。

### 9.2 KPI（Gartner G4に従い成果指標を主軸に）

| 区分 | KPI | 目標例 | データ源 |
|---|---|---|---|
| 成果（主） | PRリードタイム（作成→マージ）の中央値 | 導入前比で改善 | GitHub API |
| 成果（主） | 変更失敗率／本番障害の起因比率 | 悪化しない | インシデント管理 |
| 品質 | Dependabot・CIで検出された脆弱性の流入数（1,000PRあたり） | 悪化しない | Dependabot・CI |
| 品質 | Copilot code review指摘の対応率 | 傾向監視 | usage metrics API\[74\] |
| 開発者体験 | 四半期サーベイ（満足度・認知負荷） | 改善 | サーベイ |
| 採用（補助） | 週次アクティブ率（ライセンス保有者比） | 70%以上 | metrics |
| 統制 | 教育修了率／ポリシー同意率 | 100%（ライセンス付与条件） | LMS |
| 統制 | managed-settings.json・ポリシーのドリフト件数 | 0 | 設定棚卸し |
| コスト | 1人あたりAI Credits消費の分布、ULB到達者数 | 予算内 | 課金API |
| コスト | 非アクティブライセンス（30日）比率 | 5%未満 | metrics |

受入率や生成行数はダッシュボードに表示されますが、Gartnerの警告どおり単独の評価指標にはしません。\[1\]

### 9.3 監査・レビューサイクルと自動化

| 頻度 | 内容 | 担当 |
|---|---|---|
| 週次 | Changelog確認（新機能・新モデル・既定値変更） | AIマネージャー |
| 月次 | コスト・ULB到達者・非アクティブライセンスの棚卸し | AIマネージャー |
| 四半期 | AI Controls・managed-settings.json・ルールセットの設定棚卸し、secret risk assessment、KPIレビュー、委員会 | AIマネージャー・セキュリティ・委員会 |
| 年次 | AIリスク台帳・影響評価の更新（ISO/IEC 23894）、ポリシー改定、内部監査（ISO/IEC 42001・27001） | 委員会・内部監査 |

- **設定のコード化**：Copilotモデルポリシー（2026年10月GAのREST API：enterprise、enterprise teams、agent apps、organizationの各レベルでGET/PATCH）、予算（REST\[76\] API、`expires_at`対応）、クラウドエージェント対象リポジトリ（REST\[24\] API）、ルールセット・リポジトリ設定（Terraform\[36\] GitHubプロバイダーやsafe-settings）をGitで管理し、PRレビューを経て適用する。週次ジョブでAPIから現状を取得し、宣言値との差分を通知する。
- **ライセンス自動回収**：metrics APIで30日以上非アクティブのユーザーに通知、60〜90日で自動剥奪。

## 10. データ取扱い（法務・調達向けの確認事項）

- GitHubはCopilot Business／EnterpriseのデータをAIモデルの学習に使用しないとTrust Centerで明言しており、これはData Protection Agreementで顧客の許可なく禁止されている。\[17\]
- Trust Centerの既定設定では、IDEのチャット・コード補完・Copilot CLIの入出力は既定で保持されず、その他のCopilot利用では入出力が最大28日間保持され、ユーザーエンゲージメントデータは2年間保持される。\[17\]\[77\]
- **要確認**：GitHubは2026年8月28日のChangelogで、github.com上のCopilotがエージェントセッション体験へ完全移行し、チャットデータの保持が28日からアカウント存続期間へ変わると予告している（2026年9月28日以降）。\[78\]\[79\]Trust Centerの最新記載を契約前に必ず確認し、利用ガイドラインで「github.comチャットへの入力は長期保持され得る」と周知すること。
- 一部モデルは保持条件が異なる（例：Claude Fable 5使用時はAnthropicが安全分類器の運用のためにプロンプトと出力を保持する）。\[17\]モデル承認時のチェック項目に含める。
- パブリックコード一致フィルタ（Block）またはコード参照（Allow時の出典表示）のいずれかがIP補償の条件となる。ただしこのフィルタはCopilot coding agent（クラウドエージェント）には適用されない。\[12\]\[17\]

## 11. Caveats（留意事項）

- **Gartner情報の確度**：Gartnerの有料レポート本文は未確認です。§3の（B）は報道経由の引用であり、社内稟議でGartnerを根拠にする場合は原典を確認してください。表中の「本書での具体化」は筆者の解釈です。
- **GitHub仕様の変動**：2026年は既定値の変更、課金体系の変更（6月1日にpremium requestsからAI Creditsへ移行）、\[80\]管理画面の「AI Controls」への再編が相次ぎました。特にコンテンツ除外のエージェント対応、managed-settings.jsonのクライアント別対応、データ保持期間は変更される可能性が高く、四半期ごとの棚卸しを前提にしてください。
- **SP 800-218Aと標準マッピング**：SP 800-218AはAIモデルの開発者・統合者向けであり、Copilotの“利用者”への直接の要求事項ではありません。§4.2のマッピングも設計上の対応付けで、\[9\]認証審査での適合を保証するものではありません。

## 出典

1. [AI Firms Push Cloud Giants from 'Leaders' Quadrant in Gartner AI Coding Report -- Virtualization Review](https://virtualizationreview.com/articles/2026/06/05/ai-firms-push-cloud-giants-from-leaders-quadrant-in-gartner-ai-coding-report.aspx)
2. [Best Enterprise AI Coding Agents Reviews 2026](https://www.gartner.com/reviews/market/ai-code-assistants)
3. [GitHub Tops AI Coding Assistants Report, with Microsoft-Related 'Cautions' -- Visual Studio Magazine](https://visualstudiomagazine.com/articles/2025/09/17/report-github-tops-ai-coding-assistants-with-microsoft-related-cautions.aspx)
4. [Gartner Predicts 40% of AI Data Breaches Will Arise from Cross-Border GenAI Misuse by 2027](https://www.gartner.com/en/newsroom/press-releases/2025-02-17-gartner-predicts-forty-percent-of-ai-data-breaches-will-arise-from-cross-border-genai-misuse-by-2027)
5. [Gartner Identifies the Top Strategic Trends in Software Engineering for 2025 and Beyond](https://www.gartner.com/en/newsroom/press-releases/2025-07-01-gartner-identifies-the-top-strategic-trends-in-software-engineering-for-2025-and-beyond)
6. [Bridging AI Risk Frameworks: Reconciling ISO/IEC 42001, the NIST AI Risk Management Framework, and the EU AI Act into a Uni ed Governance Taxonomy](https://arxiv.org/pdf/2608.07515)
7. [Generative AI Risk Management: Complete Guide to NIST AI 600-1 Profile](https://www.libertify.com/interactive-library/nist-ai-600-1-generative-ai-profile/)
8. [NIST AI 600-1: The 12 Generative AI Risk Categories](https://ai-alert.org/posts/nist-generative-ai-profile-600-1-explained/)
9. [NIST SSDF for AppSec Teams: What SP 800-218 Actually Requires - OX Security](https://www.ox.security/academy/governance-compliance/nist-ssdf-for-appsec-teams-what-sp-800-218-actually-requires/)
10. [OWASP Top 10 for Agentic Applications (2026) — Modulos Governance Guide](https://docs.modulos.ai/frameworks/owasp-top-10-agentic)
11. [OWASP Agentic Top 10 for 2026 Explained With a Control for Each Risk](https://futureagi.com/blog/owasp-agentic-top-10-2026/)
12. [Application card: GitHub Copilot Agents - GitHub Docs](https://docs.github.com/en/copilot/responsible-use/agents)
13. [GitHub Copilot CLI policy hooks and managed settings for enterprises · Failproof AI](https://befailproof.ai/ai-coding-agent-admin/copilot-cli-policy-hooks-example/)
14. [GitHub Pricing 2027: Plans, AI Costs and Free Calculator](https://projectmanagers.net/github-pricing/)
15. [Risks and mitigations for GitHub Copilot cloud agent - GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/risks-and-mitigations)
16. [Evolving GitHub Advanced Security: Greater flexibility, easier to access · GitHub](https://resources.github.com/evolving-github-advanced-security/)
17. [Trust Center - github.com](https://copilot.github.trust.page/faq?s=b9buqrq7o9ssfk3ta50x6)
18. [Organization firewall settings for Copilot cloud agent - GitHub Changelog](https://github.blog/changelog/2026-04-03-organization-firewall-settings-for-copilot-cloud-agent/)
19. [Is GitHub Copilot's coding agent safe to assign your issues?](https://greenlitbooks.com/field-notes/is-github-copilot-coding-agent-safe)
20. [AI Coding Agent Approval Workflows: What, Where and Who Approves](https://safeguard.sh/resources/blog/ai-coding-agent-approval-workflows)
21. [Enterprise managed settings reference - GitHub Enterprise Cloud Docs](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-managed-settings-reference)
22. [GitHub MCP allowlists, 6 August 2026: what platform teams must change](https://ecorpit.com/copilot-mcp-allowlist-enterprise-managed-settings-agent-host-policy-2026/)
23. [Default model enablement for Copilot Business and Enterprise - GitHub Changelog](https://github.blog/changelog/2026-07-29-default-model-enablement-for-copilot-business-and-enterprise/)
24. [Budgets for usage-based billing - GitHub Docs](https://docs.github.com/copilot/concepts/billing/budgets-for-usage-based-billing)
25. [Using GitHub Copilot Coding Agent for DevOps Automation - DEV Community](https://dev.to/pwd9000/using-github-copilot-coding-agent-for-devops-automation-3f43)
26. [Establishing AI managers in your enterprise - GitHub Docs](https://docs.github.com/en/copilot/tutorials/roll-out-at-scale/govern-at-scale/establish-ai-managers)
27. [GitHub Copilot Cloud Agent — AI Stack Current](https://www.aistackcurrent.com/agents/github-copilot-cloud-agent/)
28. [Abilities and restrictions of managed user accounts - GitHub Enterprise Cloud Docs](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-iam/understanding-iam-for-enterprises/abilities-and-restrictions-of-managed-user-accounts)
29. [GitHub Learn: Your Personalized Learning Experience](https://learn.github.com/product-guides/github-enterprise/get-started/configure-enterprise-settings-and-policies)
30. [enforcing repository management policies in your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-repository-management-policies-in-your-enterprise)
31. [Copilot automations - GitHub Docs](https://docs.github.com/en/copilot/concepts/agent-capabilities/agent-powered-workflows-and-operations/about-automations)
32. [Default Enablement of Copilot features for Copilot Business and Enterprise - GitHub Changelog](https://github.blog/changelog/2026-09-24-default-enablement-of-copilot-features-for-copilot-business-and-enterprise/)
33. [Enterprise teams model policy targeting in public preview - GitHub Changelog](https://github.blog/changelog/2026-07-31-enterprise-teams-model-policy-targeting-in-public-preview/)
34. [Governance for GitHub, Copilot CLI and Agents: Keeping Control When the Code Writes Itself · Azure Greg](https://gregorsuttie.com/2026/11/03/github-copilot-agents-governance-keeping-control/)
35. [Manage GitHub Copilot app access with a dedicated policy - GitHub Changelog](https://github.blog/changelog/2026-07-27-manage-github-copilot-app-access-with-a-dedicated-policy/)
36. [Managing GitHub Copilot in the cloud for your enterprise or organization - GitHub Docs](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-agents/manage-copilot-in-the-cloud)
37. [Updates to GitHub Copilot billing and plans - GitHub Changelog](https://github.blog/changelog/2026-06-01-updates-to-github-copilot-billing-and-plans/)
38. [MCP server usage in your company - GitHub Docs](https://docs.github.com/en/copilot/concepts/enterprise/mcp-management)
39. [Where the GitHub Copilot extension points break governance](https://devopsjournal.io/blog/2026/05/01/Copilot-extension-governance-concerns)
40. [Restrict MCP server access to a custom registry - GitHub Docs](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-mcp-usage/configure-mcp-server-access)
41. [Setting up GitHub Copilot for your enterprise - GitHub Docs](https://docs.github.com/en/copilot/how-tos/copilot-on-github/set-up-copilot/enable-copilot/set-up-for-enterprise)
42. [Getting started with enterprise-managed settings - GitHub Docs](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/get-started)
43. [Enterprise managed settings - GitHub Docs](https://docs.github.com/en/copilot/reference/enterprise-administrators/enterprise-managed-settings)
44. [docs/content/copilot/reference/copilot-cli-reference/cli-config-dir-reference.md at main · github/docs · GitHub](https://github.com/github/docs/blob/main/content/copilot/reference/copilot-cli-reference/cli-config-dir-reference.md)
45. [\`managed-settings.json\` — The Core Enterprise Lockdown File · Locking Down GitHub Copilot](https://beardofedu.github.io/copilot-governance-training/sections/managed-settings/)
46. [Giving GitHub Copilot cloud sessions access to resources in your organization - GitHub Docs](https://docs.github.com/en/copilot/tutorials/cloud-agent/give-access-to-resources)
47. [Enterprise managed settings in the GitHub Copilot app and Copilot cloud agent - GitHub Changelog](https://github.blog/changelog/2026-07-27-enterprise-managed-settings-now-apply-to-the-github-copilot-app/)
48. [github-mcp-server/docs/policies-and-governance.md at main · github/github-mcp-server · GitHub](https://github.com/github/github-mcp-server/blob/main/docs/policies-and-governance.md)
49. [How to control MCP servers in GitHub Copilot](https://nirmata.com/2026/10/25/how-to-control-mcp-servers-in-github-copilot/)
50. [Managing policies and features for GitHub Copilot in your enterprise - GitHub Enterprise Cloud Docs](https://docs.github.com/enterprise-cloud@latest/copilot/managing-copilot/managing-copilot-for-your-enterprise/managing-policies-and-features-for-copilot-in-your-enterprise)
51. [GitHub Copilot is moving to usage-based billing - The GitHub Blog](https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/)
52. [GitHub Enterprise now includes access to GitHub Copilot - GitHub Changelog](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/)
53. [GitHub Copilot Privacy: Safeguards and Troubleshooting Guide](https://www.datacamp.com/tutorial/github-copilot-privacy-and-troubleshooting)
54. [GitHub Copilot Content Exclusion: What Developers Need to Know for Agent Workflows](https://devactivity.com/insights/navigating-github-copilot-s-content-exclusion-a-key-insight-for-developer-activities/)
55. [Governing agents in GitHub Enterprise](https://learn.github.com/well-architected/library/governance/recommendations/governing-agents/)
56. [Governing Claude Code, Cursor and Copilot with one policy](https://nirmata.com/2026/10/31/governing-claude-code-cursor-and-copilot-with-one-policy/)
57. [Customizing or disabling the firewall for GitHub Copilot - GitHub Docs](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/customize-the-agent-firewall)
58. [GitHub Copilot Agent Security Risks](https://airq.adversa.ai/agent/github-copilot)
59. [Configure Copilot coding agent's validation tools - GitHub Changelog](https://github.blog/changelog/2026-03-18-configure-copilot-coding-agents-validation-tools/)
60. [GitHub Copilot Security Guide: Policies, Agents and Controls](https://nhimg.org/github-copilot-security-guide)
61. [GitHub security features - GitHub Enterprise Cloud Docs](https://docs.github.com/en/enterprise-cloud@latest/code-security/getting-started/github-security-features)
62. [The secret risk assessment is generally available - GitHub Changelog](https://github.blog/changelog/2025-08-26-the-secret-risk-assessment-is-generally-available/)
63. [GitHub Secret Scanning 2026: New Patterns, Push Protection](https://www.buildmvpfast.com/blog/github-secret-scanning-pattern-updates-devops-2026)
64. [GitHub Licensing & Pricing (2026)](https://www.releaseteam.com/github-licensing-comparison/)
65. [Code security risk assessment - GitHub Docs](https://docs.github.com/en/code-security/concepts/code-scanning/risk-assessment)
66. [GitHub Secret Scanning 2026: Free for Public Repos](https://appsecsanta.com/github-secret-scanning)
67. [Secret scanning - GitHub Docs](https://docs.github.com/code-security/secret-scanning/about-secret-scanning)
68. [GitHub Pricing 2026 — Plans & Cost Breakdown - PricePulse](https://www.getpricepulse.com/companies/github-pricing.html)
69. [Copilot metrics is now generally available - GitHub Changelog](https://github.blog/changelog/2026-02-27-copilot-metrics-is-now-generally-available/)
70. [Repository-level GitHub Copilot usage metrics generally available - GitHub Changelog](https://github.blog/changelog/2026-07-17-repository-level-github-copilot-usage-metrics-generally-available/)
71. [Team-level Copilot usage metrics now available via API - GitHub Changelog](https://github.blog/changelog/2026-05-14-team-level-copilot-usage-metrics-now-available-via-api/)
72. [Usage metrics API adds usage from tools like OpenCode - GitHub Changelog](https://github.blog/changelog/2026-10-23-usage-metrics-api-adds-usage-from-tools-like-opencode/)
73. [Copilot usage metrics API adds agent app activity - GitHub Changelog](https://github.blog/changelog/2026-08-07-copilot-usage-metrics-api-adds-agent-app-activity/)
74. [Usage metrics API adds code review details and addressed rate - GitHub Changelog](https://github.blog/changelog/2026-11-05-usage-metrics-api-adds-code-review-details-and-addressed-rate/)
75. [AI credits consumed per user now in the Copilot usage metrics API - GitHub Changelog](https://github.blog/changelog/2026-06-19-ai-credits-consumed-per-user-now-in-the-copilot-usage-metrics-api/)
76. [Copilot model policy API for enterprise, orgs, and teams - GitHub Changelog](https://github.blog/changelog/2026-10-19-copilot-model-policy-api-for-enterprise-orgs-and-teams/)
77. [GitHub Copilot Security: Risks, Privacy and Plans (2026)](https://axonbuild.com/blog/github-copilot-security/)
78. [Upcoming changes to GitHub Copilot policies and billing - GitHub Changelog](https://github.blog/changelog/2026-08-28-upcoming-changes-to-github-copilot-policies-and-billing/)
79. [Copilot Chats Face Account-Lifetime Retention - Quasa.io](https://quasa.io/insights/copilot-chats-may-persist-for-your-account-s-lifetime-audit-the-policy-now)
80. [Devs Sound Off on Usage-Based Copilot Pricing Change: 'You Will Get Less, but Pay the Same Price' -- Visual Studio Magazine](https://visualstudiomagazine.com/articles/2026/04/27/devs-sound-off-on-usage-based-copilot-pricing-change-you-will-get-less-but-pay-the-same-price.aspx)
