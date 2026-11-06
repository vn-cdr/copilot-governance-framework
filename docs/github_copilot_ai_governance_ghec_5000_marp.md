---
description: GHEC 5,000ユーザー・5,000リポジトリ規模のAIガバナンス設計
marp: true
paginate: true
size: "16:9"
style: |
  section { font-family: "Aptos", "Noto Sans JP", sans-serif; color:
  #172033; background: #ffffff; padding: 54px; font-size: 25px; } h1 {
  color: #12345a; font-size: 2.0em; } h2 { color: #12345a; font-size:
  1.45em; border-bottom: 3px solid #2f6f9f; padding-bottom: 8px; } h3 {
  color: #24577d; } strong { color: #174f78; } table { font-size:
  0.72em; } th { background: #e8f0f7; color: #12345a; } blockquote {
  border-left: 5px solid #2f6f9f; color: #34445a; } footer { color:
  #64748b; font-size: 13px; }
theme: default
title: GitHub Copilot AI Governance
---

```{=html}
<!-- _class: lead -->
```
# GitHub Copilot

# AIガバナンス設計

## GHEC｜5,000ユーザー・5,000リポジトリ

**企業利用における統制・セキュリティ・運用設計**

2026年11月

------------------------------------------------------------------------

# 1. エグゼクティブサマリー

## Copilotを「AI機能」ではなく「企業統制対象」として管理する

-   **利用者**：誰が、どの機能を利用できるか
-   **データ**：何をAIに入力・参照させてよいか
-   **権限**：Agent・MCPに何を実行させるか
-   **成果物**：AI生成コードをどう検証・承認するか
-   **監査**：利用・設定変更・リスクをどう把握するか

> ポリシーだけでなく、GHECの技術設定・開発プロセス・人間の責任を組み合わせる。

------------------------------------------------------------------------

# 2. 前提条件と設計スコープ

  項目           前提
  -------------- ----------------------------------------------
  GitHub環境     GitHub Enterprise Cloud（GHEC）
  対象規模       約5,000ユーザー／5,000リポジトリ
  GHES           対象外
  一般ユーザー   Private Repositoryのみ利用可能
  管理者         数名に限定
  Copilot        Business／Enterpriseを想定。契約機能は要確認

### 設計上の重要事項

-   Privateであることは、機密情報を無条件に入力できることを意味しない。
-   Repo作成制限と既存RepoのVisibility変更制限は別々に実装する。
-   機能・モデル・クライアントごとの対応差を確認する。

------------------------------------------------------------------------

# 3. ガバナンス設計の基本原則

1.  **Default deny**：新機能・Preview・Agent・MCPは明示承認後に許可
2.  **Central control**：全社共通の最低基準はEnterpriseで強制
3.  **Least privilege**：利用者・Agent・MCPの権限を最小化
4.  **Human
    accountability**：生成物の品質・安全性・リリース責任は人間が持つ
5.  **Defense in
    depth**：Copilotだけに依存せず、CI/CD・セキュリティ検査を併用
6.  **Continuous
    governance**：利用・コスト・リスク・設定ドリフトを継続監視

------------------------------------------------------------------------

# 4. Gartner公開知見から導く方向性

## Agentの拡散とリスクを組織的に管理する

Gartnerの公開情報から、本設計に取り込む考え方：

-   AI／Agent利用ポリシーの確立
-   Agent・拡張機能の中央インベントリ
-   ID・権限・ライフサイクル管理
-   AIが扱う情報のガバナンス
-   Agent動作の監視と是正
-   利用者教育と責任あるAI利用の文化

### Copilotへの適用

機能・モデル・MCP・Plugin・Agent・利用者・データ・生成物を台帳化し、許可から廃止まで管理する。

`<small>`{=html}注：Gartner有料レポート本文ではなく、公開調査概要・公開記事・プレスリリースを参照。`</small>`{=html}

------------------------------------------------------------------------

# 5. 標準・フレームワークの位置付け

  標準                     本設計での役割
  ------------------------ ------------------------------
  NIST AI RMF              Govern／Map／Measure／Manage
  NIST AI 600-1            生成AI固有リスクの把握
  OWASP GenAI LLM Top 10   LLM・Agent脅威の分類
  ISO/IEC 42001            AIマネジメントシステム
  ISO/IEC 23894            AIリスク管理
  ISO/IEC 38507            経営・監督の責任
  ISO/IEC 27001/27002      情報セキュリティ統制
  ISO/IEC 42005            AIシステム影響評価
  NIST SSDF／OWASP SAMM    セキュア開発プロセス

------------------------------------------------------------------------

# 6. NIST AI RMFに基づく管理サイクル

  ---------------------------------------------------------------------------------------------------
  機能                    Copilotで実施すること                          成果物
  ----------------------- ---------------------------------------------- ----------------------------
  Govern                  方針、責任、教育、例外、監査                   AI利用規程

  Map                     ユースケース、データ、Agent、MCP把握           AI利用台帳・データフロー

  Measure                 品質、セキュリティ、プライバシー、コスト測定   KPI／KRI

  Manage                  リスク対応、機能制限、是正、停止               対応計画・インシデント記録
  ---------------------------------------------------------------------------------------------------

**ポイント：**
導入時の審査だけでなく、継続的な評価・改善サイクルを構築する。

------------------------------------------------------------------------

# 7. OWASP GenAIリスクと対策

  ----------------------------------------------------------------------------------
  リスク                  Copilotでの想定                    主な対策
  ----------------------- ---------------------------------- -----------------------
  Prompt Injection        Issue／README等の悪意ある指示      Agent権限・承認制御

  Sensitive Information   ソース・秘密情報の漏えい           データ分類・Content
  Disclosure                                                 Exclusion

  Excessive Agency        Agentの過剰な自律性                最小権限・MCP allowlist

  Supply Chain            悪意ある拡張・Action               提供元審査・SHA pin

  Misinformation          誤った実装・API                    テスト・レビュー

  Unbounded Consumption   Agentの過剰実行                    利用・コスト監視

  Improper Output         生成コマンド・コードの無検証実行   人間レビュー・CI
  Handling                                                   
  ----------------------------------------------------------------------------------

`<small>`{=html}OWASP GenAI LLM Top 10
2026を脅威分類として利用。各項目の適用性は機能ごとに評価。`</small>`{=html}

------------------------------------------------------------------------

# 8. ガバナンス対象の全体像

## 管理対象を8つの単位で把握する

1.  **利用者**：社員、委託先、管理者、ライセンス保有者
2.  **機能**：Chat、Completion、Agent、CLI、Cloud Agent、Code Review
3.  **モデル**：利用可能モデル、Preview、切替ポリシー
4.  **データ**：ソース、Issue、PR、設計情報、秘密情報
5.  **拡張**：MCP、Plugin、Marketplace、GitHub App
6.  **権限**：GitHub権限、Token、Agentツール権限
7.  **成果物**：生成コード、Workflow、テスト、PR
8.  **証跡**：設定変更、利用状況、Agent activity、例外、事故

------------------------------------------------------------------------

# 9. リスクレベルと利用区分

  区分       例                               利用方針
  ---------- -------------------------------- ------------------------
  Low        一般的なコード説明、テスト案     標準利用可
  Medium     Private Repoのコード補完・説明   標準統制下で利用可
  High       認証・暗号・決済・個人情報処理   追加審査・強化レビュー
  Critical   本番直接操作、秘密鍵アクセス     原則禁止・個別例外のみ

### 評価軸

情報漏えい／セキュリティ品質／Agent自律性／サプライチェーン／正確性／知的財産／プライバシー／コスト／コンプライアンス／統制逸脱

------------------------------------------------------------------------

# 10. GHECガバナンスの全体アーキテクチャ

``` text
┌────────────────────────────────────────┐
│ Enterprise Governance                 │
│ AI Policy / AI Controls / Repo Policy │
└────────────────────────────────────────┘
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
┌─────────────────┐   ┌─────────────────────┐
│ Managed Settings│   │ Organization        │
│ .github-private │   │ AI Policy / Seats   │
└─────────────────┘   └─────────────────────┘
       └────────────┬────────────┘
                    ▼
┌────────────────────────────────────────┐
│ Repository / SDLC                      │
│ Rulesets / Code Security / Actions      │
│ Instructions / DevSecOps Template       │
└────────────────────────────────────────┘
                    ▼
┌────────────────────────────────────────┐
│ Developer / Copilot / Human Review     │
└────────────────────────────────────────┘
                    ▼
┌────────────────────────────────────────┐
│ Audit / SIEM / KPI / Continuous Review │
└────────────────────────────────────────┘
```

------------------------------------------------------------------------

# 11. Enterprise／Organization／Repositoryの役割

  レイヤー       主な責任
  -------------- --------------------------------------------------
  Enterprise     全社最低基準、AI機能・モデル、共通ポリシー、監査
  Organization   部門別制限、Seat、対象Repo、例外
  Repository     データ分類、Instructions、Rulesets適用、CI/CD
  利用者         入力判断、生成物検証、レビュー、規程遵守

### 設計原則

**Enterpriseで全社基準を強制し、Organizationで追加制限し、Repositoryで開発プロセスに落とし込む。**

------------------------------------------------------------------------

# 12. Enterprise AI Controls

## 全社共通の利用可否を制御

  設定対象            推奨方針
  ------------------- ----------------------------
  Copilot利用         承認ユーザーのみ
  モデル              承認済みモデルのみ
  新機能              Disabled／管理者承認
  Agent               Pilot Organizationから開始
  Cloud Agent         限定Orgで検証
  Code Review         Pilot後に拡大
  CLI／App            クライアント別に許可
  MCP                 原則禁止、allowlist方式
  Content Exclusion   全社共通の機密パスを設定

新機能のデフォルトを明示し、未評価の機能が意図せず有効にならないようにする。

------------------------------------------------------------------------

# 13. Enterprise Managed Settings

## `.github-private`による中央管理

``` text
.github-private/
└── copilot/
    └── managed-settings.json
```

### 管理候補

-   Agentの権限モード
-   Bypass／Assisted permissionsの制御
-   MCP allowlist／denylist
-   Marketplaceの許可
-   Computer Use等の機能制御
-   モデル指定（必要に応じて）

### 運用上の注意

-   対応クライアント・キー・スキーマを確認
-   `.github-private`への書き込みを限定
-   PRレビュー・監査を必須化
-   空の許可リスト等の実際の挙動を検証
-   UIのEnterprise Policyと併用

------------------------------------------------------------------------

# 14. MCP・Plugin・Marketplace統制

## 基本方針：Default deny／Allowlist

  対象          初期方針
  ------------- ------------------------------
  MCP           原則利用不可
  社内MCP       セキュリティ審査後に限定許可
  外部MCP       原則禁止・個別審査
  Plugin        承認済みのみ
  Marketplace   承認済みのみ

### 審査観点

-   読取・書込・削除・外部送信権限
-   OAuth scope／Tokenの最小化
-   データ送信先・保存先
-   Prompt Injection時の影響
-   監査ログ・保守責任者
-   廃止・失効手順

------------------------------------------------------------------------

# 15. Private Repository onlyの実装

## Repo作成とVisibility変更を別々に統制

1.  EnterpriseのRepository management policy
2.  OrganizationのMember privileges
3.  Private以外の作成を禁止
4.  既存RepoのPublic化を別途禁止
5.  Organization外へのコード複製を禁止
6.  外部コラボレーター・App・Tokenを棚卸し
7.  Enterprise配下外の個人所有Repoを確認

> Repo作成時のPrivate-only制限だけでは、既存RepoのPublic化を防げない。

------------------------------------------------------------------------

# 16. 管理者権限と職務分掌

  ロール                   主な責務
  ------------------------ ------------------------------------
  Enterprise Owner         全社設定・緊急対応
  AI Governance Manager    AI Controls・モデル・MCP審査
  Organization Owner       Org・ユーザー・Repo管理
  Security Administrator   Code Security・監査・脆弱性
  Repository Admin         Repo設定・Rulesets
  Developer                Copilot利用・生成物検証
  Reviewer／Code Owner     設計・コード・セキュリティレビュー
  Auditor                  設定・ログの参照

管理者を少数に限定しつつ、申請・承認・実行・監査を可能な範囲で分離する。

------------------------------------------------------------------------

# 17. AI生成コードの開発プロセス

``` text
要件・設計
   ↓
Copilotによる調査・設計補助
   ↓
開発者による生成コード確認・修正
   ↓
ローカルテスト／静的解析
   ↓
Pull Request
   ↓
人間レビュー／Code Owner
   ↓
CI/CD・セキュリティ検査
   ↓
マージ・リリース承認
   ↓
運用監視
```

**Copilotは開発者を補助するが、設計・品質・リリースの責任は人間が持つ。**

------------------------------------------------------------------------

# 18. GitHubネイティブ設定とテンプレートの役割分担

  統制                   実装先
  ---------------------- -----------------------------------
  Copilot機能・モデル    Enterprise／Org AI Controls
  MCP／Plugin            Managed Settings／AI Controls
  Private-only           Enterprise／Org Repository Policy
  main保護・レビュー     Rulesets
  Secret Scanning        GitHub Code Security
  CodeQL                 Code Security Configuration
  Dependabot             GitHub標準
  Dependency Review      PR／Actions
  Container／IaC Scan    DevSecOps Template
  DAST                   CI/CD
  Copilot Instructions   Org／Repository
  利用者行動規範         ガイドライン

### 原則

**強制統制＝GitHubネイティブ設定\
検査・実行ロジック＝DevSecOpsテンプレート\
開発時の補助＝Instructions**

------------------------------------------------------------------------

# 19. Copilot Instructionsの3層モデル

  --------------------------------------------------------------------------------------------
  層                      配置                                役割
  ----------------------- ----------------------------------- --------------------------------
  Enterprise              Managed Settings                    クライアント側の強制可能な設定

  Organization            Custom Instructions                 全社・部門共通の開発原則

  Repository              `.github/copilot-instructions.md`   技術スタック・固有ルール
  --------------------------------------------------------------------------------------------

### 指示例

-   不明な仕様・APIを推測で断定しない
-   秘密情報・個人情報を出力しない
-   テストと検証観点を提示する
-   認証・認可・暗号の変更リスクを明記する
-   依存追加時はライセンス・脆弱性を確認する
-   人間レビュー・セキュリティ検査を省略しない

**Instructionsは強制的なセキュリティ境界ではない。**

------------------------------------------------------------------------

# 20. 利用者ガイドライン

## 入力してはいけない情報

-   パスワード、API Key、Access Token、秘密鍵
-   本番環境の接続情報・資格情報
-   不要な個人情報・顧客情報・決済情報
-   AI利用が禁止された契約・規制対象情報
-   許諾が確認できない他社・顧客情報
-   Content Exclusion対象情報

## 出力に対する確認義務

仕様整合性／テスト／認証・認可／暗号／依存関係／ライセンス／不要な変更／秘密情報混入

------------------------------------------------------------------------

# 21. Agent利用時の追加ルール

-   操作対象・変更範囲を確認する
-   ファイル削除・外部通信・権限変更を慎重に扱う
-   未承認MCP／Pluginを接続しない
-   承認ダイアログを内容確認なしに許可しない
-   Bypass／Allow allを使用しない
-   差分を確認し、PRでレビューを受ける
-   本番環境への直接操作・デプロイをしない

------------------------------------------------------------------------

# 22. 監査・ログ・モニタリング

## GitHub Audit LogとSIEM

監視対象： - Copilotポリシー・設定変更 - Seatの付与・剥奪 -
GitHub上のAgent activity - MCP許可リスト変更 - Repo Visibility変更 -
管理者権限変更

### 重要な制約

GitHub Enterprise audit
logは、ローカルIDEで入力されたすべてのプロンプト・会話内容を記録するものではない。

-   監査ログをSIEMへStreaming
-   保存期間・アラートを設計
-   プロンプト全文収集は目的・必要性・プライバシーを評価

------------------------------------------------------------------------

# 23. KPI／KRI

  分類           指標
  -------------- ------------------------------
  利用           Seat利用率・アクティブ利用者
  生産性         PRリードタイム・手戻り率
  品質           CI失敗率・欠陥流出
  セキュリティ   Secret検知・CodeQL検知
  Agent          承認済みMCP比率・Agent活動
  統制           Policy drift・例外期限超過
  コスト         1 Seat当たり費用・機能別費用

コード生成量だけで効果を評価しない。品質・手戻り・開発者体験と組み合わせる。

------------------------------------------------------------------------

# 24. インシデント・例外管理

## 想定インシデント

-   秘密情報・個人情報の入力
-   AI生成コードへの重大脆弱性混入
-   Agentによる意図しない変更・外部送信
-   未承認MCP／Plugin利用
-   Public Repoへの誤公開
-   Copilot設定の不正変更

## 初動

1.  影響範囲を特定
2.  必要に応じ機能・Agent・MCP・Tokenを停止
3.  Repo・PR・Actions・監査ログを保全
4.  Secretをローテーション・失効
5.  生成コード・依存関係を検査
6.  Security／AI Governance／法務へ連携
7.  再発防止策を標準へ反映

------------------------------------------------------------------------

# 25. 作成すべきドキュメント

  No.   ドキュメント                       優先度
  ----- ---------------------------------- -----------------
  D01   AI利用基本方針                     必須
  D02   Copilot利用ガイドライン            必須
  D03   Copilot Enterprise設定標準         必須
  D04   Managed Settings運用設計           必須
  D05   Organization／Repository統制標準   必須
  D06   AIリスク評価・審査基準             必須
  D07   MCP／Plugin審査基準                Agent利用時必須
  D08   Copilot Instructions標準           推奨
  D09   DevSecOpsテンプレート標準          必須
  D10   AI生成コードレビュー基準           必須
  D11   監査・モニタリング設計             必須
  D12   インシデント対応手順               必須
  D13   例外申請・承認手順                 必須
  D14   管理者運用Runbook                  必須
  D15   利用者教育資料                     推奨

------------------------------------------------------------------------

# 26. 導入ロードマップ

  -----------------------------------------------------------------------------------------
  Phase                   期間                    主な作業
  ----------------------- ----------------------- -----------------------------------------
  0\. 現状把握・設計      2〜4週間                構成・権限・データ・リスク棚卸し

  1\. 全社ガードレール    2〜4週間                AI Controls、Private-only、Managed
                                                  Settings、SIEM

  2\. Pilot               4〜6週間                限定Org／Repoで機能・品質・安全性を検証

  3\. 段階展開            1〜3か月                Org単位で展開、教育、KPI運用

  4\. 継続改善            継続                    新機能審査、監査、リスク・設定見直し
  -----------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 27. 受入テストの重点項目

  テスト                   期待結果
  ------------------------ -----------------------------
  Public Repo新規作成      拒否
  Private RepoのPublic化   拒否
  未承認モデル・機能       利用不可
  未承認MCP                接続不可
  AgentのBypass            禁止
  mainへの直接Push         拒否
  必須レビュー省略         マージ不可
  SecretのPush             Push Protection等でブロック
  必須CI失敗               マージ不可
  設定変更                 Audit Logで追跡可能

------------------------------------------------------------------------

# 28. 5,000人・5,000 Repo運用のポイント

-   Enterpriseで全社共通ポリシーを一元化
-   Organization単位で部門・リスク差を吸収
-   Repo個別設定を減らし、Rulesets・Configuration・Templateで標準化
-   例外Repoのみ個別管理
-   `.github-private`をPR・レビュー・監査の対象にする
-   設定ドリフトを定期的に検知
-   管理者の少人数運用を前提に自動化・棚卸しを設計

------------------------------------------------------------------------

# 29. 最終推奨事項

## 導入前に必須

1.  Enterprise AI Controlsで機能・モデル・新機能を管理
2.  Private-onlyを作成・Visibility変更の両面で強制
3.  Managed SettingsでAgent・MCP等を制御
4.  Content Exclusionを設定し、限界も文書化
5.  Rulesets・Secret Scanning・CodeQL等を適用
6.  Audit LogをSIEMへStreaming
7.  利用ガイドライン・コードレビュー基準を公開

## Pilot後に拡大

-   Agent／Cloud Agent／Code Review
-   審査済みMCP／Plugin
-   KPI／KRIダッシュボード
-   Organization Instructions
-   利用者教育・管理者Runbook

------------------------------------------------------------------------

# 30. まとめ

## 安全なCopilot活用を支える5つの柱

1.  **Policy**：企業としての利用方針・禁止事項
2.  **Platform**：GHECネイティブ機能による強制統制
3.  **Development**：Rulesets・DevSecOps・レビュー
4.  **People**：利用者教育・人間の責任
5.  **Monitoring**：監査・リスク・コスト・継続改善

> 5,000人・5,000
> Repoの環境では、個別ユーザーへの注意喚起ではなく、Enterpriseを中心とした標準化・自動化・継続監視が重要。

------------------------------------------------------------------------

# 参考資料

### Gartner

-   [How to Mature Generative and Agentic AI Governance for Enterprise
    Applications](https://www.gartner.com/en/documents/7129330)
-   [Gartner Identifies Six Steps to Manage AI Agent
    Sprawl](https://www.gartner.com/en/newsroom/press-releases/2026-04-28-gartner-identifies-six-steps-to-manage-artificial-intelligence-agent-sprawl)

### 標準

-   [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)
-   [NIST Generative AI
    Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
-   [OWASP GenAI LLM Top 10
    2026](https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/)
-   [ISO/IEC 42001](https://www.iso.org/standard/42001)
-   [ISO/IEC 23894](https://www.iso.org/standard/77304.html)

### GitHub

-   [Copilot enterprise
    policies](https://docs.github.com/en/copilot/concepts/enterprise/policies)
-   [Enterprise managed
    settings](https://docs.github.com/en/enterprise-cloud/latest/copilot/reference/enterprise-administrators/enterprise-managed-settings)
-   [Content
    exclusion](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/content-exclusion)
-   [Copilot audit
    logs](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/review-audit-logs)

------------------------------------------------------------------------

```{=html}
<!-- _class: lead -->
```
# Thank you

## GitHub Copilot AI Governance

### GHEC｜Enterprise-scale governance
