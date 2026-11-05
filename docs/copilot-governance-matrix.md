# GitHub Copilot AIガバナンス リスク×統制マトリクス（たたき台）

Nov 5, 2026 · @Hiroyuki Muraoka

## 前提と統制の考え方

GitHub設定で強制するのは「事故が起きたら取り返しがつかないもの」と「全社規程に直結するもの」に限り、それ以外はガイドラインに基づくユーザー責任とする。

**環境前提**

- 規模：約4,000ユーザー／約4,000リポジトリ（GitHub Enterprise Cloud）
- リポジトリ：ユーザーはprivateのみ作成可（internal・public不可）
- GHAS：追加費用のため使用しない
- 社内で承認済みの生成AI：Claude、ChatGPT
- 運用方針：申請・個別承認の負荷を避け、「標準＝ユーザー責任」を基本とする

**統制区分の判定基準**

| 区分 | 定義 | 採用する条件 |
| --- | --- | --- |
| 強制 | Enterprise／Orgのポリシーで技術的に制限する | 全社規程違反に直結する、または1回の事故で重大損失（情報流出・高額請求）になる |
| ユーザー責任 | ガイドラインで要請し、遵守は利用者に委ねる | リポジトリや業務ごとに適切な設定が異なり、一律設定すると利便性を大きく損なう |
| 検知 | 監査ログ・利用状況で事後に把握する | 強制はしないが、逸脱や異常値を定期的に確認したい |

運用移管後の管理負荷を下げるため、強制項目はできるだけEnterpriseレベルで固定し、Orgへの委任は最小限とする。

## リスク×統制マトリクス

13シナリオのうち、強制は7件（モデル・外部エージェント・MCP・公開コード一致・コスト・プレビュー・CLI既定値）、残りはユーザー責任と検知で受ける。設定場所はいずれもEnterpriseの「AI controls」タブが起点となる。

| # | リスクシナリオ | 区分 | 統制内容 | 対応するGitHub設定（レベル） |
| --- | --- | --- | --- | --- |
| R1 | 承認外のAIモデル利用（社内規程違反） | 強制 | 利用可能モデルをAnthropic・OpenAI系に限定。BYOK（自前キーのモデル追加）は禁止 | Models ポリシー、カスタムモデル／BYOK 無効（Enterprise固定） |
| R2 | 承認外の外部エージェント・Agent appsとの連携 | 強制 | パートナーエージェント（Claude／Codex）とAgent appsの可否を全社で決定 | Third-party coding agents、Agent apps ポリシー（Enterprise固定） |
| R3 | MCPサーバー経由のデータ流出・不正ツール実行 | 強制 | 利用を許可リスト／社内MCPレジストリ掲載分に限定 | MCP servers in Copilot、Enterprise allowlist、レジストリによる制限（Enterprise固定） |
| R4 | 公開コードとの一致によるライセンス・IP侵害 | 強制 | 公開コードに一致する候補をブロック | Suggestions matching public code：Blocked（Enterprise固定、名称は要確認） |
| R5 | 高価格モデル・エージェント多用によるコスト増大 | 強制＋検知 | 予算上限を設定し、AIクレジット利用を月次で確認。高単価モデルの無効化も検討 | Budgets（Enterprise／コストセンター）、Models ポリシー、利用状況ダッシュボード |
| R6 | 未成熟なプレビュー機能の利用 | 強制 | プレビュー機能はオプトインしない。GA後に個別判断 | Opt in to preview features：無効（Enterprise固定） |
| R7 | CLI・ローカルエージェント（Autopilot等）による意図しないコマンド実行 | 強制（既定値）＋ユーザー責任 | 既定の許可ツールを最小にし、追加許可はユーザー判断 | Copilot CLI ポリシー、Enterprise managed settings |
| R8 | Cloud agentの過剰権限・意図しない変更 | ユーザー責任＋強制（前提条件） | 有効化はOrg単位で可。ファイアウォール既定ON、PRレビュー必須を前提条件とする | Cloud agent ポリシー（Org委任）、エージェントのファイアウォール、ルールセット（safe-settingsで配布） |
| R9 | 機密情報・個人情報のプロンプト／コンテキスト送信 | ユーザー責任 | ガイドラインで送信禁止情報を定義。除外設定はリポジトリ管理者の責任 | Content exclusion（Repo／Org、ユーザー責任） |
| R10 | シークレットのコード混入・エージェント経由の露出 | ユーザー責任 | GHAS不使用のためpush protectionは使えない。pre-commitでの検査を推奨 | 該当設定なし（GHAS前提機能のため） |
| R11 | 生成コードの脆弱性・品質低下 | ユーザー責任 | 人によるPRレビューを必須化。Copilot code reviewは補助として任意利用 | ルールセット（PRレビュー必須）、Copilot code review ポリシー（Org委任） |
| R12 | Copilot Memory・Spacesでの情報の意図しない保存・共有 | 要判断 | 保存範囲と共有範囲を確認のうえ可否を決定 | Copilot Memory ポリシー、Spaces 関連ポリシー（Enterprise） |
| R13 | 利用状況・逸脱の把握不能 | 検知 | 監査ログとエージェント操作ログを保管し、四半期ごとに確認 | 監査ログ（agentic audit log events、ストリーミング）、Copilot usage metrics |

設定名は2026年11月時点のGitHub Docsに基づく。GitHubの機能追加が速いため、適用前に各設定画面で名称と既定値を再確認すること。出典：[Managing policies and features for GitHub Copilot in your organization](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-organization/manage-policies)、[GitHub Copilot policies for enterprises and organizations](https://docs.github.com/en/enterprise-cloud@latest/copilot/concepts/enterprise/policies)

## ユーザー責任とする事項（ガイドライン記載項目）

ユーザー責任の統制は、ガイドラインに「何を・誰が・どこまで」を明記して初めて機能する。最低限、次の項目を記載する。

- **送信禁止情報の定義**（R9）：個人情報、顧客の秘密情報、認証情報など、プロンプトやコンテキストに含めてはならない情報の具体例
- **Content exclusionの設定責任**（R9）：リポジトリ管理者が機密ファイル（設定ファイル、鍵、顧客データ等）を除外対象に登録する
- **Custom instructionsの記載責任**（R9・R11）：リポジトリ・個人のinstructionsに機密情報を書かない、品質基準を記載してよい
- **シークレット管理**（R10）：コードへの直書き禁止、pre-commit等によるローカル検査の推奨
- **生成コードのレビュー義務**（R11）：AI生成コードも人がレビューし、マージ責任はPR作成者とレビュアーが負う
- **エージェント利用時の注意**（R7・R8）：CLIで追加許可するツール・コマンドの範囲、Cloud agentに渡す権限とシークレット
- **承認外AIツールの禁止**（R1・R2）：Copilot以外のAIツールに社内コードを入力しない（社内承認済みツールを除く）

## 未決事項・関係部門との確認事項

設定値を確定する前に、次の6点を決める必要がある。

- [ ] Copilot経由のGemini等のモデル利用を「承認外AI」とみなすか（情報セキュリティ部門）— R1
- [ ] Claude／CodexのパートナーエージェントをCopilot経由で使うことを、既存の承認範囲に含めるか — R2
- [ ] Cloud agentを全社で許可するか、Org委任とするか — R8
- [ ] Copilot MemoryとSpacesの可否（保存先・保持期間の確認）— R12
- [ ] 予算上限の金額と、超過時の挙動（停止か通知のみか）（経理・管理部門）— R5
- [ ] 監査ログの保管先・保管期間と、確認担当者（PJ終了後の運用移管先）— R13
