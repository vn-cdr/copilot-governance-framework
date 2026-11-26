# GitHub Copilot AIガバナンス設計書 v3.2
## GitHub Enterpriseプラン（GHEC）・Copilot Enterprise相当前提／1,000ユーザー以上・1,000リポジトリ以上

- 作成基準日：2026年11月26日（2026年11月25日までに公開されたGitHub Changelog・Docsを反映し、11月26日に第三者レビューの指摘を反映）
- 改訂履歴：v3.2（2026年11月26日、レビュー指摘24件を反映）／v3.1（2026年11月25日、DevSecOpsフレームワーク・ランナーのEgress制御との分担を明確化）／v3（2026年11月25日）／v2（2026年11月13日）

### この文書の読み方

- 章と節は「第4章」「4.6」のように番号で表します。本文中の「（→4.6）」は「4.6節を参照」という意味です。
- 各記述の確度は次のラベルで示します。
  - **公式**：GitHub公式のDocs・Changelogで確認済み
  - **二次**：報道・解説記事による情報
  - **前版（未再確認）**：v1／v2作成時点の情報で、v3.2のレビューでも再確認できていない
  - **未確認**：公式で裏付けが取れていない。導入前に実機またはGitHubへの確認が必要
  - **社内設定値（仮）**：外部の根拠がない社内の初期値。パイロットの実測後に委員会で確定する

### 用語：GitHub Enterpriseに付いてくるCopilotと、別途購入するCopilot

本書は「Copilot Enterpriseと同等の機能を使う」ことを前提とし、その契約方法を次の2つから選びます。

| 呼び方 | 中身 | 本書での位置付け |
|---|---|---|
| **方式A：GitHub Enterpriseに付いてくるCopilot**（GitHubの呼称：Copilot access） | GitHub Enterprise（GHEC）のライセンスにCopilotが含まれる方式（2026-10-28発表）。シート料$0、含有クレジットなしで、使った分だけ従量課金。製品名ではないため、公式資料では「GitHub Enterprise now includes access to GitHub Copilot」などの表現で説明されている | **原則こちらを採用** |
| **方式B：Copilot Enterpriseを別途購入** | 従来どおりCopilot Enterpriseのシートを購入する方式。1人月$39で、月3,900クレジット分を含む | 方式Aを契約できない場合、または費用を「人数×$39」で固定したい場合に選ぶ |

本文中の「Copilot access」は方式Aを指します。

出典：方式Aの概要は[GitHub Changelog「GitHub Enterprise now includes access to GitHub Copilot」（2026-10-28）](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/)。両方式の料金・含有クレジットは[GitHub Docs「What's changing with GitHub Enterprise and Copilot billing」](https://docs.github.com/en/copilot/get-started/whats-changing-with-github-enterprise-and-copilot)の比較表。

---

## 目次

- 第0章 変更点
- 第1章 前提とスコープ
- 第2章 課題・疑問の整理
- 第3章 基本方針・体制・段階導入
- 第4章 必要なGitHub設定
- 第5章 法務・データ取扱い
- 第6章 ドキュメント体系
- 第7章 リスク台帳と標準との対応
- 第8章 モニタリングとKPI
- 第9章 次のアクション
- 第10章 参照した情報

---

## 第0章 変更点

### 0.1 v3.2：レビューで修正した主な点

v3.1に対して第三者レビューを行い、24件の指摘（重大度 高7件・中12件・低5件）を反映しました。高の7件は、そのまま設定・運用すると統制が効かない、または誤設定になる内容です。

| # | 箇所 | v3.1の記載 | v3.2での修正 | 確度 |
|---|---|---|---|---|
| 1 | 4.5 | strictPluginOnlyCustomizationの値に`"mcpServers"`を例示 | 正しいカテゴリ名は`"mcp"`。誤った値ではMCPを遮断できないおそれがある | 公式 |
| 2 | 4.6 | 予算アラートは50%／75%／90% | **75%／90%／100%の固定**で、50%は設定できない（出典：[GitHub Docs「Setting up budgets」](https://docs.github.com/en/billing/how-tos/set-up-budgets)）。早期検知はメトリクスの週次集計で代替 | 公式 |
| 3 | 4.4 | Copilot appはCLIポリシーに依存 | 2026-07-27から**Copilot app専用のポリシー**があり、CLIとは独立 | 公式 |
| 4 | 4.7、3.3 | コンテンツ除外はIDEのエージェントにも適用 | **Docsの記載に差異あり**（IDEのEdit／Agentモードは非対応とする記載が残っている）。また`permissions.deny`等はVS Codeでは**Agent Hostセッションにしか効かない**。フェーズ2の条件に実機検証とAgent Host限定を追加 | 公式（記載差異あり） |
| 5 | 1.3、第6章⑦ | 予算$0でCopilotを即時停止できる | Copilot accessでは予算$0でも**コード補完・次の編集候補は継続**し、アクセスも失われない（出典：[GitHub Changelog「GitHub Enterprise now includes access to GitHub Copilot」（2026-10-28）](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/)）。全面停止はシート割当の解除で行う | 公式 |
| 6 | 4.3 | BYOKはポリシーで無効化できる | IDEはポリシーで無効化できるが、**Copilot CLIのローカルBYOKはエンタープライズポリシーの対象外**。規程で禁止し、メトリクスで検知する | 公式 |
| 7 | 4.4 | 新機能の既定ポリシーを無効にすればよい | 既定ポリシーは**2026-10-22〜26に未構成の機能へ一回限り適用済み**。後から既定を変えても戻らないため、フェーズ0で全ポリシーを棚卸しする | 公式 |

**中・低の指摘への対応（要約）**

- **ラベルの格上げ**：チーム単位カスタマイズ、Copilot appのブラウザ操作・Azure DevOps連携、AIマネージャー用カスタムロール、ファイアウォールの組織設定、ユーザー単位予算、Fusionのモデルポリシー連動などを「公式」に変更しました。
- **日付の訂正**：Claude Fable 5.2の公開日を11/16から11/09に直しました。
- **補足の追加**：MCP許可リストの細則、`forceRemoteSettingsRefresh`の挙動、サンドボックスの強制と最小バージョン、`strictKnownMarketplaces`、code reviewのトリガー制限、セッションのリモート操作を追記しました。
- **数値の扱い**：パイロット比率、予算倍率、教育修了率などの数値は外部の根拠が確認できなかったため、「社内設定値（仮）」と明記しました。
- **標準との対応の見直し**：R5にOWASP ASI03、R10にLLM09／ASI09、R2にISO/IEC 27001 8.28を追加しました。
- **新設**：4.9「エージェントの権限と指示ファイル」、第5章「法務・データ取扱い」、7.2「NIST AI RMFとの対応」、8.2「KPI定義」を設け、2.1の論点ツリーにQ1-g「エージェントの誤動作・乗っ取り」を加えました。
- **章番号の変更**：新設に伴い、旧第5章（ドキュメント体系）以降を1つずつ繰り下げました。

**レビューと異なる判断をした点**

| 指摘 | v3.2の扱い | 理由 |
|---|---|---|
| Copilot accessの「他プランとの混在不可」「月初発効」「`*.enterprise.githubcopilot.com`」は未確認 | **公式のまま** | 2026-11-13にGitHub Docs「What's changing with GitHub Enterprise and Copilot billing」の本文で3点とも確認済み |
| コンテンツ除外はIDEのEdit／Agentモードで非対応（事実誤り） | **記載差異として扱い、実機検証を必須化** | 2026-11-13に確認した現行Docs（Security, governance, and network settings＞Content exclusion）には、IDE・CLI・Copilot appのエージェントにも適用される（間接アクセスは検知できない場合あり）と書かれている。旧ページの記載と食い違うため、どちらとも断定しない |

### 0.2 v2からv3.1までの変更

| # | 変更内容 |
|---|---|
| 1 | 規模を「約5,000」から「**1,000ユーザー以上・1,000リポジトリ以上**」に変更。段階導入の人数は比率で表記 |
| 2 | **Copilot Businessをスコープ外**に変更。Copilot Enterpriseと同等の機能を前提とし、契約方法は方式A（GitHub Enterpriseに付いてくるCopilot）または方式B（Copilot Enterpriseを別途購入）とする |
| 3 | 2026年11月25日までのリリースを反映 |
| 4 | 論点ツリーの各問いと、本書の該当章・設定・文書との対応表を追加 |
| 5 | 「§」表記を廃止し、「第X章」「X.X」「（→X.X）」の表記に統一 |
| 6 | 一般的なソフトウェアサプライチェーン対策と、ランナーのEgress制御を含む項目を、別資料「DevSecOpsフレームワーク」の範囲とし、本書の対象外に変更（→1.4） |

### 0.3 反映した主なリリース

| 日付 | リリース | ガバナンス上の影響 | 反映先 | 確度 |
|---|---|---|---|---|
| 07/27 | Copilot app専用のポリシーを新設。managed-settingsがCopilot appとクラウドエージェントにも適用 | CLIとappを別々に開放・遮断できる | 4.4、4.5 | 公式 |
| 09/02 | コンテンツ除外がCopilot app・CLIでGA | CLI・appでも除外が効く | 4.7 | 公式 |
| 10/08 | code reviewの「Only allow Copilot code review to be triggered by authorized users」 | 自社付与ライセンス以外によるレビュー依頼を禁止できる | 4.4 | 公式 |
| 10/09 | ローカルサンドボックスがGA（CLI・Copilot app・VS Code） | エージェントのファイル・ネットワーク・資格情報へのアクセスを制限できる。追加費用なし | 4.5 | 公式 |
| 10/09 | Copilot appで、Copilotライセンスとリポジトリに別々のGitHubアカウントを使用可能に | 会社のライセンスで社外アカウントのリポジトリを扱う経路になり得る | 4.4 | 公式 |
| 10/19 | モデルポリシーのAPI（エンタープライズ・組織・チーム） | モデルポリシーをIaC化できる | 4.3 | 公式 |
| 10/22 | 新機能のグローバル既定ポリシーがGA。10/22〜26に未構成の機能へ一回限り適用 | 意図せず有効化された機能がないか棚卸しが必要 | 4.4 | 公式 |
| 10/28 | GitHub EnterpriseにCopilot access同梱（シート料$0、含有クレジットなし） | 従量部分の上限は予算のみ | 1.3、4.6 | 公式 |
| 10/28 | Copilot Fusion（自動モデルルーティング）GA | モデルポリシーで有効なモデルのみ使用 | 4.3 | 公式 |
| 11/03 | 使用量メトリクスAPIでBYOK利用を区別して取得可能に | BYOK利用の検知 | 第8章 | 公式 |
| 11/06 | code reviewの設定をREST／GraphQL APIで管理可能に。Enterpriseが管理する設定は下位で上書き不可 | IaC化・ドリフト防止 | 4.4 | 公式 |
| 11/09 | 新モデルClaude Fable 5.2：既定でデータ保持が必要、ポリシー既定オフ | データ保持条件の異なるモデルの承認判断が必要 | 4.3、第5章 | 公式 |
| 11/10 | エンタープライズのGitHub AppからCopilotシート管理APIを呼べるように | シート割当の自動化をPATなしで実装できる | 4.1 | 公式 |
| 11/16 | 一部モデルの廃止。既定のモデル有効化が有効だと、代替モデルが自動で有効化される | 既定有効化を切らないと承認外モデルが増える | 4.3 | 公式 |
| 11/16 | managed setting「strictPluginOnlyCustomization」追加 | 利用者個人・リポジトリ単位の野良MCP・エージェント定義を遮断できる | 4.5 | 公式 |
| 11/17 | Copilot CLIでMicrosoft Entraサインイン（プレビュー、EMU＋OIDCのみ） | SSO統合が容易に。プレビューのため評価対象 | 4.1 | 公式 |
| 11/17 | Copilot appのブラウザ操作（プレビュー、既定オフ）、Azure DevOps連携（Experimental、ADOリポジトリではクラウドセッション不可） | エージェントの操作範囲が拡大 | 4.4、4.5 | 公式 |
| 11/18 | チーム単位のカスタマイズGA（チームメンテナーがマーケットプレイスを追加し、直下メンバーにプラグインを既定で有効化） | マーケットプレイス制限がないと、チーム経由で任意のプラグインが入り得る | 4.5 | 公式 |

---

## 第1章 前提とスコープ

### 1.1 環境

| 項目 | 前提 |
|---|---|
| プラットフォーム | GitHub Enterprise Cloud（GHESは対象外） |
| 規模 | **1,000ユーザー以上**（全員がGHECライセンス保有と仮定）、**1,000リポジトリ以上** |
| リポジトリ可視性 | 数名の管理者以外はprivateのみ |
| Copilotの契約方法 | Copilot Enterpriseと同等の機能を前提とする。原則は**方式A：GitHub Enterpriseに付いてくるCopilot**（GitHubの呼称：Copilot access）を採用する。方式Aを契約できない場合、または費用を人数×$39で固定したい場合は、**方式B：Copilot Enterpriseを別途購入**する（→1.3、2.4 D1） |
| GHAS | 追加費用のため未導入の可能性あり。「あり／なし」を併記 |
| 承認済み生成AI | 限定的（例：Claude・ChatGPTのみ）。Copilot内のモデルも同じ範囲に合わせる想定 |
| 運用方針 | 申請・例外処理を最小化し、ユーザー責任の「標準」を中心にする |

### 1.2 スコープ外

- **Copilot Business**（同一エンタープライズ内でCopilot accessと混在できないため、比較・設定の対象外とする）
- GHES
- 個人プラン（Copilot Free／Pro／Pro+／Max）の業務利用。禁止の対象としてのみ扱う
- 一般的なソフトウェアサプライチェーン対策と、ランナーのEgress制御。いずれもDevSecOpsフレームワークで扱う（→1.4）

### 1.3 GitHub Enterpriseに付いてくるCopilotと、別途購入するCopilotの比較

| 項目 | 方式A：GitHub Enterpriseに付いてくるCopilot（Copilot access） | 方式B：Copilot Enterpriseを別途購入 |
|---|---|---|
| シート料 | GHECライセンス保有者は**$0**（非保有者はCopilot Standaloneで月$10） | 月$39／ユーザー |
| 含有AIクレジット | **なし**（初回利用から従量） | 3,900／ユーザー／月（エンタープライズ内でプール） |
| AIクレジット単価 | 1クレジット＝$0.01 | 同左 |
| 機能・モデル | Copilot Enterpriseと同等 | ― |
| 予算が尽きたとき | AIクレジットを消費する機能（チャット、CLI、code review、クラウドエージェント等）は停止。**コード補完と次の編集候補は継続**し、アクセス自体は失われない | ― |
| シート割当 | 必要（自動では付与されない） | 必要 |
| 契約経路 | 2026-10-28時点ではGitHub／Microsoftのアカウントチーム経由のみ。セルフサービスは段階的に開放予定（日程未発表） | 通常購入 |
| 移行 | 任意。月初1日付で発効 | ― |

確度：公式

出典：
- シート料・含有クレジット・Copilot Standalone・混在不可・月初発効：[GitHub Docs「What's changing with GitHub Enterprise and Copilot billing」](https://docs.github.com/en/copilot/get-started/whats-changing-with-github-enterprise-and-copilot)
- 契約経路・予算が尽きたときの挙動：[GitHub Changelog「GitHub Enterprise now includes access to GitHub Copilot」（2026-10-28）](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/)
- AIクレジット単価：[GitHub Docs「Billing（Copilot）」](https://docs.github.com/en/copilot/concepts/billing-and-usage/organizations-and-enterprises/billing)

**試算（1,000名、全員GHECライセンス保有、単価は公式の1クレジット＝$0.01。当社固有の割引は2.5で確認）**

| 月間の利用額 | 方式B（Copilot Enterprise） | 方式A（Copilot access） |
|---|---|---|
| $20,000 | $39,000（未使用枠は失効） | **$20,000** |
| $39,000 | $39,000 | $39,000 |
| $60,000 | $39,000＋超過$21,000＝$60,000 | $60,000 |

計算方法（本書の試算）：方式Bは「$39×1,000名＝$39,000」が最低額で、含有クレジット（3,900×1,000名×$0.01＝$39,000相当）を超えた分だけ追加課金。方式Aは利用額がそのまま請求額。利用額の3つの水準は説明用の仮定値。

示唆：単価が同じなら、方式Aの総額は常に方式B以下になる。ただし**シート料という上限がなく、従量部分の歯止めは予算設定だけ**になる（→4.6）。予算で止まらない補完の課金上の扱いは2.5で確認する。

### 1.4 前提とする既存の取り組み（本書の対象外）

AIの有無にかかわらず必要な対策は、別資料の「DevSecOpsフレームワーク設計書」で定義・実装する。本書はこれが整備されていることを前提とし、内容は記載しない。関係する資料は本書とDevSecOpsフレームワーク設計書の2つだけで、ほかの資料は必要ない。

| 資料 | 主な範囲 |
|---|---|
| **DevSecOpsフレームワーク**（ランナーのEgress制御を含む） | Actionsの許可リスト・SHAピン留め、GITHUB_TOKENの権限、OIDC、Dependabot、SBOM、Artifact attestations、CIでの依存関係・シークレット・SAST検査、自社ランナー（セルフホスト／ラージランナー）からの外部通信の制限（Egress制御） |

**本書との分担**：判断基準は「AIが新たに持ち込む、またはAIによって大きくなるリスクかどうか」。該当するものは本書、該当しないものはDevSecOpsフレームワークで扱う。両方に関わる項目は、**要件（何を満たすか）を本書**に、**実装（どう実現するか）をDevSecOpsフレームワーク**に書き、2つの資料の両方に記載する。

| 項目 | 本書（AIガバナンス） | DevSecOpsフレームワーク |
|---|---|---|
| モデル、MCP・プラグインの入手元 | 決定・設定（→4.3、4.5） | ― |
| AIが提案する依存パッケージ | レビュー基準として実在確認を定める（→第6章③） | CIでの検査 |
| 指示ファイル（AGENTS.md等）の保護 | CODEOWNERSの対象に指定（→4.2、4.9） | フレームワークのリポジトリ構成に組み込み |
| クラウドエージェントの外部通信 | 要件を定める（→4.7） | 自社ランナーで動かす場合はEgress制御で実装（組込みファイアウォールが自社ランナーと非互換のため必須） |
| エージェントが変更したワークフローの実行 | 人の承認を必須とする（→4.7） | ― |
| Actionsの許可リスト、SHAピン留め、Dependabot、SBOMなど | 対象外 | 定義・実装 |

---

## 第2章 課題・疑問の整理

### 2.1 論点ツリー

```
Copilot AIガバナンス
├─ Q1 何を守るのか
│   ├─ Q1-a 機密情報・ソースコードの社外流出
│   ├─ Q1-b 生成コードの品質・脆弱性
│   ├─ Q1-c 知的財産（ライセンス汚染）
│   ├─ Q1-d コスト（含有枠なし＝上限なし）
│   ├─ Q1-e 承認外AIへの迂回（シャドーAI、BYOK、別アカウント）
│   ├─ Q1-f 説明責任（誰が・どのAIで・何をしたか）
│   └─ Q1-g エージェントの誤動作・乗っ取り（プロンプトインジェクション、過剰権限、MCP）
├─ Q2 何を技術で強制し、何をルール（ユーザー責任）に任せるか
├─ Q3 誰が決め、誰が運用し、誰が責任を負うか
├─ Q4 どの順序で機能を開放するか（補完→チャット→エージェント）
└─ Q5 どう測り、どう見直すか
```

### 2.2 各問いと本書の対応表

| 問い | 答えている章・節 | 主なGitHub設定 | 関連リスク（第7章） | 関連文書（第6章） |
|---|---|---|---|---|
| **Q1-a** 機密情報の流出 | 4.5、4.7、4.8、第5章 | コンテンツ除外、managed-settingsのdeny・sandbox、セッション同期の無効化 | R1、R13 | ①②⑦ |
| **Q1-b** コード品質・脆弱性 | 4.2、4.8 | ルールセット（PR必須・Code Ownerレビュー）、Copilot code review、Dependabot | R2、R10 | ③ |
| **Q1-c** 知的財産 | 4.4、第5章 | パブリックコード一致のブロック | R3 | ②③ |
| **Q1-d** コスト | 1.3、4.6 | 多層予算、上限到達時の停止 | R8 | ⑤⑥ |
| **Q1-e** 承認外AIへの迂回 | 2.3、4.3、4.4、4.5 | 新モデル既定無効、BYOK無効（IDE）、strictPluginOnlyCustomization、全リポジトリでのブロック系ポリシー、EMU | R7、R9、R12 | ①④ |
| **Q1-f** 説明責任 | 4.1、第8章 | 監査ログストリーミング、使用量メトリクスAPI | R11 | ⑤⑦ |
| **Q1-g** エージェントの誤動作・乗っ取り | 4.5、4.7、4.9 | deny・sandbox・bypass禁止、ファイアウォール、strictPluginOnlyCustomization・strictKnownMarketplaces、指示ファイルのCODEOWNERS | R4、R5、R6 | ②④ |
| **Q2** 強制とルールの線引き | 第3章（方針2・3）、第4章全体、第6章 | 第4章の各設定 | 全般 | ①② |
| **Q3** 体制・責任分担 | 3.2 | AIマネージャー用カスタムロール | R11 | ⑤⑥ |
| **Q4** 開放の順序 | 3.3 | エンタープライズチーム単位のシート割当、機能ポリシー | R5 | ② |
| **Q5** 測定と見直し | 第8章、7.2、3.3の各フェーズの移行条件 | 使用量メトリクスAPI、予算アラート | 全般 | ⑨⑩ |

### 2.3 GitHub Enterpriseプラン特有の課題

| # | 課題 | 問題になる理由 | 対応（参照先） |
|---|---|---|---|
| E1 | 含有クレジットがなく全量従量 | シート料という上限がなく、従量部分の歯止めは予算設定だけ。補完は予算0でも止まらない | 予算を多層化し、上限到達時の停止を必須にする。全面停止はシート解除（→4.6） |
| E2 | シート割当が必要 | 「Enterpriseに含まれる＝全員が自動で使える」ではない | IdP同期のエンタープライズチームで割当。GitHub AppによるAPI自動化（→4.1） |
| E3 | 他のCopilotプランと混在不可 | 組織単位の段階移行ができない | 移行はエンタープライズ単位で一括判断（→2.4 D1） |
| E4 | アカウントチーム経由のみ | セルフサービス契約では現時点で利用不可 | 契約形態の確認を最初のアクションにする（→第9章） |
| E5 | 接続先がEnterpriseエンドポイントに切り替わる | FWが`*.business.githubcopilot.com`のみ許可だと利用が止まる | 許可リストに`*.enterprise.githubcopilot.com`を追加（→4.1） |
| E6 | Copilot Enterprise相当の全機能が対象 | 統制対象の機能が多い | 既定で無効にし、段階的に開放（→3.3、4.4） |
| E7 | 通常のポリシーが及ぶのは自社付与のシートのみ | 個人プラン利用者や、別アカウントでリポジトリを扱う経路が残る | 全リポジトリでブロックするポリシー、code reviewのトリガー制限、EMU、利用規程（→4.4、第6章①） |

### 2.4 未決事項（社内で決める必要があること）と推奨案

| # | 未決事項 | 推奨案と理由 |
|---|---|---|
| D1 | 方式A（GitHub Enterpriseに付いてくるCopilot）と方式B（Copilot Enterpriseを別途購入）のどちらで契約するか | **方式Aを推奨（条件付き）**。単価が同じなら総額は方式B以下になる（→1.3）。条件は、アカウントチーム経由で契約できることと、予算を多層で設定・運用できること。どちらかを満たせない場合は方式Bを選ぶ |
| D2 | 承認モデルの範囲 | 社内承認済みAIのベンダー系に限定。新モデルと代替モデルは手動承認。Fusionもこの範囲に限定し、高コストモデルはFusion用に個別無効化 |
| D3 | データ保持が必要なモデル（例：Claude Fable 5.2）を許可するか | 法務がデータ保持条件を確認するまで**無効のまま**（既定オフを維持）。ポリシーを有効にすると保持要件を承認したことになる。適格エンタープライズにはZDR（データ非保持）の例外が2026年末まであるため、適格性をアカウントチームに確認。2027年以降の条件（Enterprise Frontier Safeguards）は法務がレビュー |
| D4 | BYOK（個人キー・ローカルモデル）を許可するか | **禁止**。IDEはポリシーで遮断する。Copilot CLIのローカルBYOKはポリシー対象外のため、規程（第6章①）で禁止し、使用量メトリクスで検知する |
| D5 | エージェント機能の開放範囲 | 段階開放。IDEエージェントモード（VS CodeはAgent Hostセッションに限定）→CLI／Copilot app→クラウドエージェント（対象リポジトリ限定） |
| D6 | MCP・スキル・カスタムエージェントの配布方法 | **プラグイン経由に限定**（strictPluginOnlyCustomization）し、承認済みの社内マーケットプレイス（strictKnownMarketplaces）で配布。チームメンテナーは承認済みマーケットプレイス内のプラグインだけを選べる |
| D7 | ローカルセッションのクラウド同期 | 初期は無効。開放する場合も「View from cloud」までとし、リモート操作（View and control）は認めない |
| D8 | 予算の水準 | 社内設定値（仮）：ユーザー上限はパイロット実測の中央値の2〜3倍、全体予算は月次見積り＋20%。パイロット後に委員会で確定 |
| D9 | GHASの導入 | 予算がつくなら、機密度の高いリポジトリからSecret Protectionを優先導入。導入しない場合は4.8の代替策 |
| D10 | VS Codeのエージェント利用をAgent Hostセッションに限定するか | **限定を推奨**。`permissions.deny`などの細かな権限ルールは、VS CodeではAgent Hostセッションにしか適用されないため |
| D11 | OpenTelemetryで社内に収集する範囲 | **identityのみ**。プロンプト・応答・ツール出力の収集（既定は無効）を有効にすると、機密情報が社内の収集基盤に複製される |

### 2.5 GitHub・アカウントチームへの確認事項

1. 当社の契約でCopilot accessを利用できるか。移行可能な最短の月初日。
2. 当社固有の割引条件。Copilot accessでの補完・次の編集候補の課金上の扱い。
3. surfaceごとのプロンプト・応答・セッションの保持期間（第5章で整理した内容の確認）。
4. 移行後も既存のポリシー・予算・managed-settingsが引き継がれるか。
5. 個人プラン利用者と、Copilot appの「ライセンスとリポジトリで別アカウント」機能を、EMU以外の方法で統制できる範囲。
6. managed-settingsの各キーが、どのクライアント（VS Code、JetBrains、CLI、Copilot app、クラウドエージェント）で強制されるか。
7. IDEのEdit／Agentモードと、クラウドエージェントに対するコンテンツ除外の適用（Docsの記載に差異があるため）。
8. Claude Fable 5.2のZDR例外の適格性。
9. AIマネージャーのカスタムロールで、Copilotメトリクスの閲覧権限を付与できるか。
10. code reviewの設定APIで使える認証方式（GitHub App・fine-grained PATを使えないとの報道があるため）。

---

## 第3章 基本方針・体制・段階導入

### 3.1 基本方針

1. **既定は閉じ、承認したものだけ開ける**：新機能・新モデル・代替モデルを自動で有効化させない。すでに適用された既定値は棚卸しで確認する。
2. **技術で強制できるものは強制する**：モデル、BYOK、カスタマイズの配布元、エージェント権限、ネットワーク、ブランチ保護、予算は設定で担保する。
3. **技術で強制できない部分はルールと監視で補う**：CLIのローカルBYOK、別アカウントでの利用、コンテンツ除外が効かない経路などは、規程で禁止し、メトリクスと監査ログで検知する。
4. **ルールは短く、ユーザー責任の「標準」に絞る**：開発者が読む文書は3本に限定する（→第6章）。
5. **エージェントは別リスクとして段階開放する**。
6. **コストは予算で物理的に止める**。

### 3.2 体制（RACIの要点）

| 活動 | 説明責任 | 実行 |
|---|---|---|
| ポリシー策定・モデル／機能の承認 | AIガバナンス委員会（議長：CISO） | AIマネージャー |
| AI Controls・managed-settings・予算の設定 | AIマネージャー | AIマネージャー |
| ルールセット・ブランチ保護 | Enterprise Owner | Org Owner、リポジトリ管理者 |
| MCP・プラグイン・マーケットプレイスの審査 | セキュリティ | AIマネージャー |
| 指示ファイル（AGENTS.md等）の変更レビュー | セキュリティ | プラットフォームチーム |
| AI生成コードのレビュー | リポジトリ管理者 | 開発者 |
| データ保持・契約条件の確認 | 法務 | AIマネージャー |
| インシデント対応 | セキュリティ | AIマネージャー |

AIマネージャーには、Enterprise Ownerの代わりに、エンタープライズのカスタムロールで「Manage enterprise AI controls」「Read enterprise audit logs」の権限を付与する（公式。エンタープライズのカスタムロールはパブリックプレビュー）。AI Controls画面から監査ログなどへ移動するには、対応する権限を別途付与する必要がある。Copilotメトリクスの閲覧権限は未確認（2.5-9）。

### 3.3 段階導入

期間・対象比率・移行条件の数値はすべて**社内設定値（仮）**です。GitHub公式・Gartnerで数値の根拠は確認できていません。パイロットの実測後に委員会で確定します。

| フェーズ | 期間目安 | 対象 | 開放する機能 | 次へ進む条件 |
|---|---|---|---|---|
| 0 基盤 | 1〜2か月 | 管理者 | 契約、ネットワーク、EMU／IdP、**既存ポリシーの棚卸し**、AI Controlsの既定値、予算、監査ログ、ルールセット、managed-settings | 第4章のチェックリストを完了 |
| 1 パイロット | 2〜3か月 | 全体の**5〜10%**（複数部門、若手とベテランを混在） | IDE補完、IDEチャット、github.comチャット、code review | 品質指標の悪化なし、コスト実測値の取得 |
| 2 拡大 | 2〜4か月 | 全体の**30〜50%** | ＋IDEエージェントモード（**VS CodeはAgent Hostセッションに限定**、sandbox強制、bypass禁止） | 教育修了率95%以上、予算内、**コンテンツ除外とdenyの実機検証で効果を確認** |
| 3 全社 | 以降 | 全員 | ＋承認済みプラグイン（MCP含む）、Copilot CLI・Copilot app（希望チーム。ポリシーは別々に開放） | 四半期レビューで承認 |
| 4 エージェント拡張 | 個別 | 選定チーム | クラウドエージェント（対象リポジトリ限定）、自動化 | 個別のリスク評価で承認 |

---

## 第4章 必要なGitHub設定

凡例：「GHAS」欄の「不要」は追加ライセンス不要、「SP」はSecret Protection、「CS」はCode Securityが必要という意味です。

### 4.1 契約・ライセンス・ID

| 設定 | 推奨値 | 理由 | 確度 |
|---|---|---|---|
| 契約方法 | 方式A：GitHub Enterpriseに付いてくるCopilot（エンタープライズ単位で一括移行） | 2.4 D1 | 公式 |
| Copilotシート割当 | IdP同期のエンタープライズチーム単位。フェーズごとにチームを拡大 | 自動付与されないため、申請なしで管理できる方式にする | 公式 |
| シート割当の自動化 | エンタープライズのGitHub App（権限：Enterprise Copilot license management）でシート管理APIを呼ぶ | 個人のPATに依存しない自動化 | 公式（11/10） |
| EMU | 採用（既存環境なら移行計画を検討） | 個人アカウント混在による迂回と流出を構造的に防ぐ | 前版（未再確認） |
| SSO／SCIM | OIDC（Entra ID）＋条件付きアクセス | 退職者の即時無効化。OIDCならCopilot CLIのEntraサインイン（プレビュー）も利用できる | 前版（未再確認）／公式（11/17） |
| IP許可リスト | 社内・VPNのみ | 端末外からのアクセス制限 | 前版（未再確認） |
| ネットワーク許可リスト | `*.enterprise.githubcopilot.com`を許可 | Copilot accessはEnterpriseエンドポイントを使う | 公式 |
| 監査ログ | SIEMへストリーミング。保存期間は社内の監査要件に合わせてSIEM側で確保 | エージェント操作の追跡。GitHub側の保存期間は未確認 | 前版（未再確認） |

### 4.2 リポジトリ・ルールセット

| 設定 | 推奨値 | 確度 |
|---|---|---|
| リポジトリ作成 | Privateのみ許可 | 前版（未再確認） |
| 可視性の変更・削除・移管 | Enterprise Owner／Org Ownerのみ | 前版（未再確認） |
| フォーク | 同一組織内のみ、または禁止 | 前版（未再確認） |
| ユーザー名前空間リポジトリ（EMU） | 作成禁止 | 前版（未再確認） |
| Enterpriseルールセット（デフォルトブランチ） | 削除禁止、force push禁止、PR必須（承認1名以上、Code Ownerレビュー必須、古い承認の破棄）、ステータスチェック必須 | 前版（未再確認） |
| CODEOWNERS | `.github/workflows/`、`.github/copilot-instructions.md`、`AGENTS.md`、`.github/agents/`、スキル定義、`copilot-setup-steps.yml`をプラットフォームチームの所有にする（→4.9） | 前版（未再確認） |
| カスタムプロパティ | `data_classification`、`copilot_agent_allowed`、`criticality` | 前版（未再確認） |

### 4.3 AI Controls：モデル

| 設定 | 推奨値 | 理由 | 確度 |
|---|---|---|---|
| リリース済みモデルの既定有効化（グローバル既定） | **無効** | 新モデルだけでなく、モデル廃止時の代替モデルも自動で有効化されるため | 公式（11/16） |
| 個別モデルポリシー | 承認ベンダー系のみ有効 | 2.4 D2 | 公式 |
| データ保持が必要なモデル（例：Claude Fable 5.2） | **無効のまま**（既定オフ） | 有効化は保持要件の承認を意味する（2.4 D3、第5章） | 公式（11/09） |
| Copilot Fusion | 承認モデルのみで動作させる。高コストモデルはFusion用に個別無効化。既定モデルにする場合はmanaged-settingsで指定 | Fusionはモデルポリシーで有効なモデルだけを使う。Fusion専用の無効化ができる | 公式（10/28） |
| Fusionのローカルモデル選択 | 使用しない（第6章①で禁止） | 実験的機能。ルーティング判断はクラウドで行われ、ローカルモデルは安全保護がクラウドモデルと異なる | 公式（10/28） |
| BYOK（IDE）：「Bring Your Own Language Model Key in Select IDEs」（旧称「…in VS Code」） | **無効**（既定は有効） | 承認外プロバイダへの経路 | 公式 |
| BYOK（Copilot CLIのローカルキー） | 技術的に遮断できないため、規程で禁止し、メトリクスで検知（→第8章） | CLIのローカルキーはエンタープライズポリシーの対象外 | 公式 |
| Enterprise BYOK（カスタムモデル） | 無効（パブリックプレビュー） | 必要時に個別評価 | 公式 |
| モデルポリシーのAPI管理 | APIでIaC化し、週次で差分を検知 | ドリフト防止 | 公式（10/19、APIの存在） |

### 4.4 AI Controls：機能・エージェント

| 設定 | 推奨値 | 理由 | 確度 |
|---|---|---|---|
| 新機能の既定ポリシー（Default policy for new features） | **Disabled**。あわせてフェーズ0で**全ポリシーを棚卸し** | 2026-10-22〜26に未構成の機能へ一回限り適用済み。後から既定を変えても、すでに設定された機能は戻らない。対象はFeatures & clients、code review、MCP servers in Copilot。プレビュー機能はオプトインのまま | 公式 |
| パブリックコードに一致する提案 | ブロック | ライセンス汚染の防止 | 前版（未再確認） |
| Copilot CLI | フェーズ3まで無効。開放は希望チームのみ | シェル実行を伴う | 公式 |
| GitHub Copilot app（AI Controls＞Copilot Clients） | フェーズ3まで「Disabled everywhere」 | 2026-07-27から専用ポリシー。CLIとは独立して制御する | 公式（07/27） |
| Copilot appのブラウザ操作・Azure DevOps連携 | 無効（`features.computerUse:false`で強制） | プレビュー・Experimental機能。エージェントの操作範囲が広がる | 公式（11/17） |
| Copilot appの「ライセンスとリポジトリで別アカウント」 | 利用規程で社外アカウントのリポジトリでの利用を禁止 | 機能の存在は公式。管理者が無効化するポリシーは見つかっていない（2.5-5） | 公式（機能の存在）／未確認（制御手段） |
| Copilot in the cloud（クラウドエージェント） | フェーズ4まで無効 | 自律実行 | 前版（未再確認） |
| Copilot cloud agentを全リポジトリでブロック | フェーズ4まで有効 | 個人プランや他エンタープライズのユーザーも含めて遮断 | 公式 |
| Copilot code review | フェーズ1から有効（予算設定後）。設定はAPIで管理し、Enterpriseで固定する | Enterpriseが管理する設定は下位で上書きできない | 公式（11/06） |
| Only allow Copilot code review to be triggered by authorized users | 有効 | 組織またはエンタープライズが付与したライセンス以外（個人ライセンス等）によるレビュー依頼を禁止する | 公式（10/08） |
| サードパーティエージェント／Agent apps | 無効 | 承認外AIの経路を閉じる | 前版（未再確認） |
| Store local sessions in the Cloud | **Disabled**を明示設定。開放時も「View from cloud」まで | 「View and control」は同期に加えてリモート操作を許すため。`remoteControl.mode:"disabled"`でも制限できる | 公式 |
| Copilot Memory、Spaces、Web検索、Connectors | 初期は無効、個別評価 | 情報の持続・外部送信を伴う。ポリシーの存在は確認済み、既定値はフェーズ0の棚卸しで実機確認 | 未確認（既定値） |
| プレビュー機能 | 無効（パイロット群のみ評価） | プレリリース条項 | 前版（未再確認） |
| AI credits paid usage | **有効のまま** | Copilot accessでは無効だとクレジット消費機能が使えない | 公式 |

### 4.5 managed-settings（`.github-private`リポジトリの`copilot/managed-settings.json`）

フェーズ2以降の目標設定です。キー名・値はGitHub Docs「Enterprise managed settings」に準拠しています。

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "deny": [
      "Read(./.env)",
      "Read(~/.ssh/**)",
      "Shell(git push --force *)",
      "Shell(rm -rf *)",
      "Shell(terraform destroy *)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowBypass": false
  },
  "features": { "computerUse": false },
  "remoteControl": { "mode": "disabled" },
  "strictPluginOnlyCustomization": ["mcp", "extensions", "hooks"],
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "ORG/approved-plugins" }
  ],
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.internal.example.co.jp/*" }
  ],
  "deniedMcpServers": [],
  "forceRemoteSettingsRefresh": true
}
```

| 項目 | 内容 | 確度 |
|---|---|---|
| permissions（deny・bypass禁止） | エージェントに禁止する読み取り・コマンドを指定する。**VS Codeでは、この細かな権限ルールはAgent Hostセッションにしか適用されない**（→2.4 D10） | 公式 |
| sandbox | ローカルサンドボックスを強制する。`failIfUnavailable:true`でサンドボックスが使えない環境では実行させない。最小バージョン（VS Code 1.141.0／CLI 1.0.93／Copilot app 1.1.26。出典：[GitHub Docs「Enterprise managed settings」](https://docs.github.com/en/copilot/reference/enterprise-administrators/enterprise-managed-settings)）を満たさないクライアントは接続を遮断されるため、事前に周知する | 公式 |
| features.computerUse | ブラウザ操作を含むcomputer useを止める | 公式 |
| remoteControl | セッションのリモート操作を止める | 公式 |
| strictPluginOnlyCustomization | 指定したカテゴリ（`mcp`、`extensions`、`hooks`など）について、ユーザー・プロジェクト単位の独自設定を遮断し、プラグインまたはmanaged settings由来だけを許可する。`true`で全カテゴリが対象。複数の管理ソースの制限は結合され、ユーザー設定・リポジトリ設定では緩められない。**マーケットプレイスの許可リストではない**ため、strictKnownMarketplacesと組み合わせる | 公式 |
| strictKnownMarketplaces | 承認済みの社内マーケットプレイスだけを許可する。空配列は全面ロックダウン。**未設定だと、チーム単位のカスタマイズ（11/18）経由で任意のマーケットプレイスが入り得る** | 公式 |
| チーム単位のカスタマイズ | チームメンテナーがマーケットプレイスを追加し、直下のメンバーにプラグインを既定で有効化できる（VS Code・Copilot app・CLI）。クライアントは、許可されたマーケットプレイスと必須・禁止プラグインの設定を引き続き強制する | 公式（11/18） |
| allowedMcpServers／deniedMcpServers | 未設定なら全許可、`[]`なら組込みの既定サーバー以外を全遮断、denyが常に優先。複数ソースのallowは積集合、denyは和集合。**GitHub MCPなどファーストパーティのサーバーは対象外**。クラウドエージェントでは、リポジトリのカスタムエージェント定義に書かれたMCPには適用されないため、`.github/agents/`をCODEOWNERSで管理する（→4.2）。ファイアウォールはMCPサーバーには適用されない | 公式 |
| forceRemoteSettingsRefresh | 起動時にサーバー設定の再取得を必須にする（CLI・VS Code・Copilot app）。失敗時、CLIとappは操作を制限し、VS CodeはCopilot機能を遮断する。オフライン環境と初回起動の扱いを第6章⑤に記載する | 公式 |
| 対応クライアント | 各キーの対応クライアントは公式リファレンスで確認する（2.5-6）。2026-07-27からCopilot appとクラウドエージェントも対象 | 公式（07/27）／未確認（キー単位） |

### 4.6 予算（Copilot accessでは最重要）

| 設定 | 推奨値 | 確度 |
|---|---|---|
| ユーザー単位の予算（共通上限／コストセンター別上限／個別ユーザー上限の3種） | 共通上限：パイロット実測の中央値の2〜3倍（社内設定値（仮））。個別ユーザー上限は例外時のみ、有効期限付き。**ユーザー単位の予算は常にハードストップ**（停止の設定は不要） | 公式（仕様）／社内設定値（仮）（金額） |
| コストセンター／組織／エンタープライズ予算 | 設定し、**「Stop usage when budget limit is reached」を明示的に選ぶ**（選ばないと上限後も止まらない） | 公式 |
| アラート | **75%／90%／100%（固定）**（出典：[GitHub Docs「Setting up budgets」](https://docs.github.com/en/billing/how-tos/set-up-budgets)）。50%時点の早期検知は、使用量メトリクスAPIのユーザー別`ai_credits_used`を週次で集計して代替（社内設定値（仮）） | 公式 |
| 予算申請 | 例外申請の窓口として利用 | 公式（機能の存在） |
| コストセンターの確認 | REST APIでリソースの所属コストセンターを照会し、割当漏れを検知 | 公式（11/03） |
| 予算が尽きたとき（Copilot access） | AIクレジット消費機能は停止、補完・次の編集候補は継続（出典：[GitHub Changelog「GitHub Enterprise now includes access to GitHub Copilot」（2026-10-28）](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/)）。**利用を全面停止するにはシート割当を解除する** | 公式 |
| code reviewのActions分数 | code reviewはAIクレジットに加えてActions分数も消費する。Actions分数はAIクレジットの予算の対象外のため、Actions側の予算も設定する | 公式（コミュニティ告知） |
| 予算の見直し頻度 | 月次。ユーザー別メトリクスは日次集計の確定まで最大2日遅れるため、日次での厳密な追跡には向かない | 公式（遅延）／社内設定値（仮）（頻度） |

### 4.7 組織：コンテンツ除外・クラウドエージェント

| 設定 | 推奨値 | 確度 |
|---|---|---|
| コンテンツ除外 | `**/.env*`、`**/secrets/**`、`**/*.pem`、`**/*.key`、顧客データのサンプル等 | 公式 |
| コンテンツ除外の適用範囲 | IDEの補完・チャット、Copilot CLI・Copilot app（09/02 GA）、code reviewには適用される。**IDEのEdit／Agentモードとクラウドエージェントは、Docsの記載に差異があるため未確定**（2.5-7）。フェーズ2の前に実機で検証する | 公式／未確認（IDEエージェント・クラウドエージェント） |
| コンテンツ除外の位置付け | 間接的なアクセスは検知できない場合があり、シンボリックリンクとリモートファイルシステムは対象外。**OS権限・アクセス制御の代替にはならない**。エージェント利用時はsandboxとdenyを併用する（→4.5） | 公式 |
| 組織カスタム指示 | セキュアコーディング規約の要点、ハードコード禁止など | 公式（機能の存在） |
| クラウドエージェントのファイアウォール（GitHubホストのランナー） | 組織で「Enable firewall」をEnabled、「Allow repository custom rules」をDisabled、推奨許可リストを有効 | 公式 |
| クラウドエージェントの実行ランナー（自社ランナー） | 組込みファイアウォールは**自社ランナーと非互換**（Windowsとも非互換）。自社ランナーで動かす場合は、DevSecOpsフレームワークのEgress制御（→1.4）を前提条件とし、許可する通信先の要件を本書で定める。ランナーはephemeral（使い捨て）に限定 | 公式 |
| code reviewの実行ランナー | 自社ランナー上のcode reviewはファイアウォールなしで動くため、同じくEgress制御を前提にする | 公式（07/17） |
| クラウドエージェントのワークフロー実行 | 承認必須を維持 | 前版（未再確認） |
| copilot環境のシークレット | 原則登録しない | 前版（未再確認） |
| クラウドセッションの共有 | Actions上のセッションはリポジトリ閲覧者に既定で共有され、sandboxのセッションは既定で非共有。利用規程で周知し、高機密リポジトリは対象外にする | 公式 |
| 対象リポジトリ | `copilot_agent_allowed=true`のリポジトリのみ | 前版（未再確認） |

### 4.8 セキュリティ機能（GHASあり／なし）

| 機能 | GHASなし（private） | GHASあり |
|---|---|---|
| Dependabot、依存関係グラフ | 利用可 | 利用可 |
| ルールセット | 利用可 | 利用可 |
| シークレットリスク評価（無料のポイントインタイム診断） | 利用可 | 利用可 |
| クラウドエージェント組込みのCodeQL・シークレット検査 | 利用可（エージェント生成コードのみ） | 利用可 |
| secret scanning・push protection | **不可** | SP |
| CodeQLコードスキャン・Copilot Autofix | 不可 | CS |

確度：前版（未再確認）。導入判断の前に、GitHubの最新の機能一覧・価格で確認する。

**GHASなしの代替策**：CIでのOSSのSAST／シークレット検出は、DevSecOpsフレームワークで定義する（→1.4）。本書では、AI利用に伴って増えるシークレット混入の確認として、無料のシークレットリスク評価を四半期ごとに実行することを定める。

> Actionsの設定（許可するアクション、GITHUB_TOKENの権限、OIDCなど）は、DevSecOpsフレームワークで扱う（→1.4）。

### 4.9 エージェントの権限と指示ファイル

エージェントは「誰の権限で、何を読み、どこへ通信できるか」で被害範囲が決まります。統制の効く範囲がクライアントごとに異なるため、効かない経路を把握したうえで重ねて止めます。

| 対象 | 方針 | 確度 |
|---|---|---|
| ローカルエージェント（IDE・CLI・Copilot app） | bypass禁止、sandbox強制、`permissions.deny`で秘密情報の読み取りと破壊的コマンドを禁止（→4.5）。VS CodeはAgent Hostセッションに限定 | 公式 |
| クラウドエージェント | GitHubホストのランナー＋ファイアウォール強制を基本とする。自社ランナーはephemeral限定で、Egress制御を前提にする（→4.7） | 公式 |
| 指示ファイル | `copilot-instructions.md`、`AGENTS.md`、`.github/agents/`、スキル定義をCODEOWNERS必須にし、変更時はセキュリティ部門がレビューする。プロンプトインジェクションを持ち込む経路になり、カスタムエージェント定義内のMCPはMCP許可リストの対象外のため | 公式（MCPの対象外）／社内設定値（仮）（運用） |
| 成果物 | エージェントのPRも人のレビュー・承認を経てマージする（→4.2）。エージェントが変更したワークフローは承認されるまで実行しない | 前版（未再確認） |
| 監視 | エージェント関連の監査ログイベントと、OpenTelemetry（identityのみ）をSIEMへ送る（→第5章、第8章） | 公式（監査イベント・OpenTelemetryの存在） |

---

## 第5章 法務・データ取扱い

法務・契約部門と確認すべき論点です。数値・条件はGitHubの規約・Trust Centerの改定で変わるため、契約前と年1回見直します。

| 論点 | 現時点の情報 | 確度 | 本書での扱い |
|---|---|---|---|
| 学習への利用 | Copilot Business／Enterpriseのデータはモデルの学習に使わないとGitHubが表明 | 前版（未再確認） | Trust Center・DPAで確認 |
| データ保持（github.com上のチャット） | 2026-09-28以降、github.com上のCopilotはエージェントセッションに移行し、チャットデータは28日ではなくアカウント存続期間保持される | 公式（Changelog 2026-08-28） | 利用ガイドライン（第6章②）で「github.comチャットへの入力は長期保持される」と周知 |
| データ保持（IDE・クラウドエージェント） | IDEのチャット・補完は既定で非保持、エンゲージメントデータは2年、クラウドエージェントのセッションログはアカウント存続期間とされる | 未確認（Trust Centerで要確認） | 2.5-3で確認 |
| データ保持が必要なモデル | Claude Fable 5系は保持が必須。ポリシーの有効化は保持要件の承認を意味する。ZDR例外は適格企業のみ・2026年末まで | 公式 | 2.4 D3 |
| セッションの同期・共有 | ローカルセッションの同期は「Store local sessions in the Cloud」で制御。Actions上のクラウドセッションはリポジトリ閲覧者に既定で共有 | 公式 | 4.4、4.7 |
| OpenTelemetry | プロンプト・応答・ツール出力の収集は既定で無効。有効にすると機密が社内の収集基盤に複製される | 公式 | identityのみ許可（2.4 D11） |
| IP補償 | 補償の適用条件（パブリックコード一致のブロック設定との関係など） | 未確認 | 法務がDPA・製品条項で確認 |
| データレジデンシー | GHE.com（データレジデンシー付きGHEC）の要否 | 未確認 | 社内の規制要件に応じて判断 |
| AI生成物の著作権・帰属 | AI生成コードの著作権の扱い、帰属表示（コミットトレーラー等）の社内方針 | 社内設定値（仮） | 法務と方針を決め、第6章③に反映 |
| 個人情報 | 個人情報・顧客データをプロンプトに含めないこと、コンテンツ除外の対象にすること | 社内設定値（仮） | 第6章②の入力禁止情報 |

---

## 第6章 ドキュメント体系

開発者が読むのは①②③の3本に限定します。

| # | 文書 | 読者 | 主な内容 | 対応する問い |
|---|---|---|---|---|
| ① | 生成AI利用ポリシー（Copilot章） | 全員 | 承認ツールは会社付与のCopilotのみ。個人プラン・BYOK（**CLIのローカルキーを含む**）・ローカルモデル（Fusionのローカルルーティングを含む）・承認外AIの業務利用禁止。社外アカウントのリポジトリでの会社ライセンス利用禁止 | Q1-a、Q1-e、Q2 |
| ② | Copilot利用ガイドライン（標準） | 開発者 | フェーズ別の利用可能機能、承認モデル一覧（Fusion含む）、入力禁止情報（個人情報・顧客データ・シークレット）、**VS CodeでのエージェントはAgent Hostセッションで使うこと**、クラウドセッションの共有範囲、github.comチャットの長期保持、コンテンツ除外はセキュリティ境界ではないこと | Q1-a、Q1-c、Q1-g、Q2、Q4 |
| ③ | AI生成コード・レビュー基準 | 開発者・レビュアー | AI生成PRも同一基準でレビュー、依頼者以外が承認、ワークフロー変更の確認、依存パッケージの実在確認、AI生成物の帰属表示 | Q1-b、Q1-c |
| ④ | プラグイン（MCP・スキル・エージェント）登録標準 | 開発者・チームメンテナー・AIマネージャー | プラグイン経由に限定、承認済みマーケットプレイスの審査基準、**チームメンテナーは承認済みマーケットプレイス内のみ選択可**、指示ファイルの変更レビュー | Q1-e、Q1-g |
| ⑤ | 管理者運用手順書 | AIマネージャー | Copilot access移行手順、**フェーズ0のポリシー棚卸し**、シート割当の自動化、設定のIaC化と差分検知、新機能・新モデルの評価手順、**forceRemoteSettingsRefreshのオフライン・初回起動時の扱い**、sandboxの最小バージョン周知 | Q1-d、Q1-f、Q3 |
| ⑥ | 例外管理手順 | 管理者 | 例外は予算引き上げ・プラグイン追加・クラウドエージェント対象追加の3種に限定、期限付き | Q1-d、Q3 |
| ⑦ | インシデント対応手順 | セキュリティ | シークレット入力時のローテーション、**個別ユーザー予算を$0にしてAIクレジット消費機能を即時停止。補完も含め全面停止する場合はシート割当を解除**、モデルの無効化、指示ファイル改ざん時の対応 | Q1-a、Q1-f、Q1-g |
| ⑧ | 教育プログラム | 全員 | 受講・同意をシート割当の条件にする。AIの過信防止、プロンプトインジェクションの見分け方 | Q2、Q1-g |
| ⑨ | KPI定義 | 委員会 | 8.2の指標 | Q5 |
| ⑩ | AIリスク台帳 | 委員会 | 第7章 | Q5 |

---

## 第7章 リスク台帳と標準との対応

### 7.1 リスク台帳

| ID | リスク | 管理策（参照先） | OWASP LLM 2025 | OWASP Agentic 2026 | ISO/IEC | NIST | 対応する問い |
|---|---|---|---|---|---|---|---|
| R1 | 機密情報・シークレットの漏えい | コンテンツ除外、deny・sandbox（→4.5、4.7） | LLM02 | ― | 27001 8.12 | AI 600-1 Data Privacy／Information Security | Q1-a |
| R2 | 脆弱なコードの混入 | ルールセット、code review、Dependabot（→4.2、4.8） | LLM05 | ASI05 | 27001 8.28 | SSDF PW.7 | Q1-b |
| R3 | ライセンス汚染 | パブリックコード一致のブロック（→4.4、第5章） | ― | ― | 42001 A.10 | AI 600-1 Intellectual Property | Q1-c |
| R4 | プロンプトインジェクション | ファイアウォール、プラグイン限定、指示ファイルのCODEOWNERS（→4.5、4.7、4.9） | LLM01 | ASI01 | 23894 | AI 600-1 Information Security | Q1-g |
| R5 | エージェントの過剰権限 | 段階開放、全リポジトリでのブロック、bypass禁止、sandbox（→3.3、4.4、4.5、4.9） | LLM06 | ASI03（主）、ASI02 | 27001 8.2 | ― | Q1-g、Q4 |
| R6 | MCP・プラグイン経由の流出 | strictPluginOnlyCustomization、strictKnownMarketplaces、MCP許可リスト（→4.5） | LLM03 | ASI04 | 42001 A.10 | AI 600-1 Value Chain and Component Integration | Q1-g、Q1-e |
| R7 | 承認外・保持条件の異なるモデルの利用 | 既定有効化の無効、モデル個別承認（→4.3、第5章） | LLM03 | ― | 42001 A.10 | AI 600-1 Value Chain and Component Integration | Q1-e |
| R8 | コストの上限なし | 多層予算、上限到達時の停止、シート解除（→4.6） | LLM10 | ― | ― | ― | Q1-d |
| R9 | シャドーAI（個人プラン・別アカウント） | EMU、ブロック系ポリシー、code reviewのトリガー制限、利用規程（→4.1、4.4、第6章①） | ― | ― | 27001 5.10 | ― | Q1-e |
| R10 | 過信・レビューの形骸化 | 教育、依頼者以外の承認（→第6章③⑧） | LLM09 | ASI09 | ― | AI 600-1 Human-AI Configuration | Q1-b |
| R11 | 監査証跡の不足 | 監査ログストリーミング、メトリクス（→4.1、第8章） | ― | ― | 27001 8.15 | ― | Q1-f |
| R12 | BYOK・ローカルモデル経由の承認外AI利用 | IDEのBYOK無効化、CLIは規程で禁止しメトリクスで検知（→4.3、第8章） | LLM03 | ASI04 | 27001 5.10 | ― | Q1-e |
| R13 | セッション・メモリの共有・汚染による開示 | 同期の無効化、リモート操作の禁止、共有範囲の周知（→4.4、4.7、第5章） | LLM02 | ASI06 | 27001 8.12 | AI 600-1 Data Privacy | Q1-a |

注記：
- OWASP Agentic 2026は「OWASP Top 10 for Agentic Applications for 2026」（2025-12-09公開、ASI01〜ASI10）を指す。IDと名称は解説記事経由で確認しており、OWASP原本では照合していない（二次）。R5は、付与済み権限の誤用（ASI02）よりも、権限の昇格・統制の回避（ASI03 Identity and Privilege Abuse）を主とする。
- ISO/IEC 27001:2022附属書Aの条項名は二次資料で照合した（規格本文は有料）。27001 8.2（特権的アクセス権）と23894の対応付けは本書の提案。
- NIST AI 600-1のリスク区分名は原本（NIST.AI.600-1.pdf）の英語名で記載した。

### 7.2 NIST AI RMFとの対応

| 機能 | 内容 | 本書の該当箇所 |
|---|---|---|
| GOVERN（統治） | 方針・体制・責任・文書 | 3.1方針、3.2体制、第6章ドキュメント体系、⑥例外管理、第5章法務 |
| MAP（把握） | 利用状況とリスクの特定 | 第2章論点ツリー、1.4分担表、7.1リスク台帳 |
| MEASURE（測定） | リスクと効果の測定 | 第8章モニタリングとKPI、四半期の設定ドリフト点検 |
| MANAGE（対処） | 対策の実施と見直し | 第4章GitHub設定、3.3段階導入、⑦インシデント対応 |

Gartnerの推奨（AI TRiSM、エージェントのスプロール対策など）との対応は、v1で報道経由の情報として整理した内容を引き継いでおり、v3.2では再確認していない（前版（未再確認））。

---

## 第8章 モニタリングとKPI

### 8.1 データソース

| データソース | 取得できる主な情報 | 確度 |
|---|---|---|
| 使用量メトリクスAPI | ユーザー別AIクレジット消費（`ai_credits_used`）、BYOK利用（`used_local_byok`等、11/03）、code reviewの指摘内容と対応率（11/06）、テレメトリ欠落・クライアントバージョンの遅れ（11/06）。確定は最大2日遅れ | 公式 |
| インパクトダッシュボード | チャット・VS Codeエージェントの利用状況（11/06） | 公式 |
| 監査ログ | エージェント関連の監査イベント（Agentic audit log events）、ポリシー変更 | 公式（イベント一覧の存在） |
| 予算・課金 | 予算到達、アラート（75／90／100%）、コストセンター別の消費 | 公式 |
| API差分検知 | モデルポリシー・code review設定・managed-settingsのドリフト | 社内設定値（仮）（仕組み） |
| CI・インシデント管理 | PRリードタイム、変更失敗率 | 社内設定値（仮）（定義） |

### 8.2 KPI定義

閾値は、明記したもの以外すべて社内設定値（仮）です。

| KPI | 定義 | データソース | 閾値 | 対応する問い |
|---|---|---|---|---|
| PRリードタイム（成果の主指標） | PR作成からマージまでの中央値 | GitHub API | 導入前比で改善 | Q5 |
| 変更失敗率 | AI支援PRに起因する差し戻し・障害の比率 | CI・インシデント記録 | 導入前比で悪化なし | Q1-b、Q5 |
| 予算到達ユーザー率 | 月内にユーザー単位予算へ到達した人数÷シート数 | 予算・使用量メトリクスAPI | 社内設定値（仮） | Q1-d |
| BYOK利用者数 | BYOK利用が記録されたユーザー数（禁止後も検知用に監視） | 使用量メトリクスAPI | 0 | Q1-e |
| code review対応率 | Copilotの指摘のうち対応されたものの比率 | 使用量メトリクスAPI | 社内設定値（仮） | Q1-b |
| 設定ドリフト件数 | AI Controls・managed-settingsの承認外の変更 | 監査ログ・API差分 | 0 | Q2 |
| 未対応クライアント率 | sandboxの最小バージョン未満で接続を遮断された利用者の比率 | 接続遮断ログ・メトリクス | 0に近づける | Q1-g |
| セッション共有・同期の設定逸脱 | 高機密リポジトリでのクラウドセッション共有、同期ポリシーの逸脱 | 監査ログ | 0 | Q1-a |
| 教育修了率 | シート保有者のうち教育を修了した比率 | 教育システム | 95%以上（社内設定値（仮）） | Q2 |

---

## 第9章 次のアクション

1. アカウントチームに2.5の1・2・4・8を確認し、Copilot accessへの移行可否と移行月を確定する。
2. 2.4の未決事項D1〜D11を委員会で決定する。
3. フェーズ0として、次を実施する。
   - **既存ポリシーの棚卸し**（10/22〜26の一回限りの既定適用で有効化された機能がないか）
   - ネットワーク許可リスト
   - モデルの既定有効化の無効化と、IDEのBYOKの無効化
   - managed-settings（strictPluginOnlyCustomization、strictKnownMarketplaces、bypass禁止）
   - 多層予算（停止の明示選択、Actions分数の予算を含む）
4. フェーズ2の前に、IDEエージェント（Agent Hostセッション）とクラウドエージェントで、コンテンツ除外とdenyが効くかを実機で検証する（2.5-7）。
5. 法務とTrust Center・DPAを確認し、第5章の未確認事項を埋める。

---

## 第10章 参照した情報（2026年11月26日時点）

**GitHub公式（Changelog）**
- 「Manage GitHub Copilot app access with a dedicated policy」「Enterprise managed settings now apply to the GitHub Copilot app」（2026-07-27）
- 「Copilot code review customization and configurability improvements」（2026-07-17）
- 「Upcoming changes to GitHub Copilot policies and billing」（2026-08-28）
- 「Content exclusions generally available in Copilot app and CLI」（2026-09-02）
- 「Default enablement of Copilot features for Copilot Business and Enterprise」（2026-09-24）
- 「Copilot code review: new organization billing options and controls」（2026-10-08）
- 「GitHub Copilot weekly releases — October 5」（2026-10-09）
- 「Global default for Copilot features generally available」（2026-10-22）
- 「GitHub Enterprise now includes access to GitHub Copilot」「Route and orchestrate models with GitHub Copilot Fusion」「GitHub Copilot Fusion now supports selection of local models」（2026-10-28）
- 「BYOK and enterprise custom models in the Copilot usage API」（2026-11-03）
- 「Copilot code review: API management and review overview updates」（2026-11-06）
- 「Claude Fable 5.2 in GitHub Copilot」（2026-11-09）
- 「GitHub App support for Enterprise Copilot seat management」（2026-11-10）
- 「GitHub Copilot models deprecated on November 16, 2026」「Control where GitHub Copilot customizations come from」（2026-11-16）
- 「GitHub Copilot CLI adds Microsoft Entra sign-in as a preview」「GitHub Copilot app: Microsoft Ignite releases」（2026-11-17）
- 「Customize GitHub Copilot for your team」（2026-11-18）
- 「Bring your own language model key in VS Code now available」（2026-04-22）
- 「Delegate AI controls management to members of your enterprise」（2025-11-03）

**GitHub公式（Docs）**
- What's changing with GitHub Enterprise and Copilot billing
- Adopting GitHub Copilot in your enterprise（Enterprise AI governance）
- Enterprise managed settings（reference）、Managing plugin marketplaces and plugins with enterprise-managed settings
- Content exclusion for GitHub Copilot、About GitHub Copilot session data、About remote control
- Blocking agentic features in your enterprise、Establishing AI managers
- Bring your own key、Administering Copilot CLI for your enterprise
- Customizing the agent environment（クラウドエージェントのファイアウォール）
- Budgets and alerts、Setting up budgets、Billing（Copilot）
- GitHub Copilot usage metrics

**標準**
- NIST AI 600-1（Generative AI Profile）原本
- OWASP Top 10 for LLM Applications 2025、OWASP Top 10 for Agentic Applications for 2026（ID・名称は解説記事経由）
- ISO/IEC 27001:2022、42001、23894（条項名は二次資料で照合）

**二次情報**
- code review設定APIの認証方式の制限（報道）
- Gartnerの推奨はv1で報道経由として整理した内容を継続使用（v3.2では再確認していない）
