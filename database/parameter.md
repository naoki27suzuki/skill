# Aurora PostgreSQL Database Insights 関連パラメータ オーバーヘッド評価表

本ドキュメントは Aurora PostgreSQL で Database Insights（実行計画キャプチャ／スロークエリ分析）を有効化する際に設定するパラメータについて、**CPU・メモリ・ディスクへの影響**を整理したものです。

---

## 🚨 重要な前提: 情報の信頼度区分

本ドキュメントでは情報源を以下の3区分に分けて明示しています。**特に本番環境への適用判断時は、情報源の信頼度を確認してください。**

| 区分 | マーク | 内容 | 拘束力 |
|---|:---:|---|---|
| **【AWS公式】** | 🅰️ | AWS公式ドキュメント・公式ブログ・AWS re:Post記事に明記 | 高（AWSが公式に述べている事実） |
| **【PostgreSQL公式】** | 🅿️ | PostgreSQL公式ドキュメント・PostgreSQLメーリングリストに記載 | 中（PostgreSQL の標準機能としての一般原則） |
| **【推測・第三者】** | 🅼️ | pganalyze等の第三者ブログ、論理的な推測 | 低（参考情報、本番判断には追加検証が必要） |

### ⚠️ 重要な注意事項

**AWS公式ドキュメントには `aurora_stat_plans.*` 系パラメータのオーバーヘッドに関する明示的な記述は一切ありません。** 公式ドキュメント "[Monitoring query execution plans and peak memory for Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html)" を全文確認した結果：

- ❌ "performance impact" の記述なし
- ❌ "overhead" の記述なし
- ❌ "CPU/memory/disk impact" の記述なし
- ❌ "production recommendation" の記述なし
唯一の機能仕様注記:「The configuration for `aurora_stat_plans.with_*` parameters takes effect only for newly captured plans.」のみ。

**したがって、本ドキュメントの `aurora_stat_plans.*` 系の影響評価は、PostgreSQL の `auto_explain` 拡張（同様の機能）に対する PostgreSQL公式の警告を参考にした「推測」です。**

`aurora_stat_plans` は内部的に `auto_explain` 相当の処理を行うと考えられるため、論理的に同様のオーバーヘッドが発生する可能性は高いものの、**AWSの公式保証ではない**点にご注意ください。本番投入前にステージング環境での負荷テストが必須です。

---

## 1. 実行計画キャプチャ関連パラメータ

### 1-1. `aurora_compute_plan_id`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | クエリ実行時にプラン識別子（plan_id）を付与する | 🅰️ |
| **デフォルト値** | `off`（Aurora PostgreSQL 14.10、15.5 以上） | 🅰️ |
| **推奨値（本番）** | `1`（on） ※実行計画キャプチャに必須 | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **第三者評価（pganalyze）** | "without measurable overhead" ※AWSの公式保証ではない | 🅼️ |
| **反映タイミング** | 動的（再起動不要） | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html) / [AWS re:Post](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature) / [pganalyze（第三者）](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora) | |

---

### 1-2. `aurora_stat_plans.with_costs`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | EXPLAIN プランに各ノードの推定コストと行数を含める | 🅰️ |
| **デフォルト値** | `on` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **PostgreSQL一般原則からの推測** | コスト情報追記のみで、オーバーヘッドは極めて小さいと推測 | 🅼️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **出典** | [AWS Doc: Parameter reference](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) | |

---

### 1-3. `aurora_stat_plans.with_analyze` ⚠️ 注意

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | EXPLAIN ANALYZE を実行し、実測（actual）の実行時統計をキャプチャ | 🅰️ |
| **デフォルト値** | `off` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし**（AWSは具体的な影響評価を公開していない） | 🅰️ |
| **PostgreSQLの類似機能（auto_explain.log_analyze）に対する警告** | "**This can have an extremely negative impact on performance**" ※`aurora_stat_plans` 固有の評価ではない | 🅿️ |
| **PostgreSQLコミュニティ実測（auto_explain）** | CPU バウンドなクエリで最大10倍の遅延事例あり ※`aurora_stat_plans` 固有の数値ではない | 🅿️ |
| **CloudWatchコンソール表示への影響** | デフォルトは推定プラン表示。実測プランを表示するには `on` が必要 | 🅰️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **本番判断の指針** | **AWS公式のオーバーヘッド評価がないため、本番有効化前にステージング環境での負荷テストが必須** | 🅼️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) / [AWS Doc: Database Insights Execution Plans](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Execution-Plans.html) / [PostgreSQL Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html) | |

---

### 1-4. `aurora_stat_plans.with_timing`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ANALYZE 使用時、各プランノードの開始時間と所要時間を含める | 🅰️ |
| **デフォルト値** | `on` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **PostgreSQLの類似機能（auto_explain.log_timing）に対する警告** | "The overhead of repeatedly reading the system clock can **slow down queries significantly on some systems**" ※`aurora_stat_plans` 固有の評価ではない | 🅿️ |
| **動作条件** | `with_analyze=off` の時は影響しない（ANALYZE時のみ動作） | 🅰️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) / [PostgreSQL Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html) | |

---

### 1-5. `aurora_stat_plans.with_buffers`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ANALYZE 使用時、バッファ使用統計を含める | 🅰️ |
| **デフォルト値** | `off` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **PostgreSQLコミュニティ実測（log_buffers）** | 約2%のTPS低下 ※`aurora_stat_plans` 固有の数値ではない | 🅿️ |
| **動作条件** | `with_analyze=on` の時のみ動作 | 🅰️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) / [PostgreSQL ML（log_buffers実測）](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt) | |

---

### 1-6. `aurora_stat_plans.with_wal`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ANALYZE 使用時、WAL レコード生成情報を含める | 🅰️ |
| **デフォルト値** | `off` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **動作条件** | `with_analyze=on` の時のみ動作 | 🅰️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) | |

---

### 1-7. `aurora_stat_plans.with_triggers`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ANALYZE 使用時、トリガー実行統計を含める | 🅰️ |
| **デフォルト値** | `off` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **動作条件** | `with_analyze=on` の時のみ動作 | 🅰️ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) | |

---

### 1-8. `aurora_stat_plans.minutes_until_recapture`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | プラン再キャプチャまでの分数。`0` で再キャプチャ無効 | 🅰️ |
| **デフォルト値** | `0` | 🅰️ |
| **許容範囲** | `0-1073741823` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **動作仕様** | `aurora_stat_plans.calls_until_recapture` のしきい値超過時にプランを再キャプチャ | 🅰️ |
| **反映タイミング** | 動的 | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) | |

---

### 1-9. `aurora_stat_plans.calls_until_recapture`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | プラン再キャプチャまでの呼び出し回数 | 🅰️ |
| **デフォルト値** | `0`（再キャプチャ無効） | 🅰️ |
| **許容範囲** | `0-1073741823` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし** | 🅰️ |
| **反映タイミング** | 動的 | 🅰️ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) | |

---

## 2. スロークエリログ関連パラメータ

> **このセクションは AWS公式に明確な記述があります。** スロークエリログ系のパラメータは、AWS公式ドキュメント・公式ブログ・AWS re:Post で**ストレージ消費と CPU 使用量への影響が明示的に警告**されています。

### 2-1. `log_min_duration_statement` ⚠️ AWS公式警告あり

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | この時間（ミリ秒）を超えるクエリをログ出力。スロークエリ判定の閾値 | 🅰️ |
| **デフォルト値** | `-1`（無効） | 🅰️ |
| **AWS公式の影響に関する記述** | "**Increasing database logging impacts storage size, I/O use, and CPU use. Because of this, test these changes before deploying them in production.**" | 🅰️ |
| **AWS公式の警告** | "**don't set the parameters to values that generate extensive logging, such as ... log_min_duration_statement to 0**" | 🅰️ |
| **AWS公式のストレージ警告** | "**the DB instance is unavailable when the volume storage is full**" | 🅰️ |
| **AWS公式の推奨例** | "log_min_duration_statement to **1000** (1 second)" | 🅰️ |
| **反映タイミング** | 動的（再起動不要） | 🅰️ |
| **出典** | [AWS Database Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) / [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging) / [AWS re:Post: Optimize query performance](https://repost.aws/knowledge-center/rds-postgresql-query-performance) | |

---

### 2-2. `log_statement` ⚠️ AWS公式警告あり

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | SQL ステートメント種別のログ出力（`none`/`ddl`/`mod`/`all`） | 🅰️ |
| **デフォルト値** | `none` | 🅰️ |
| **AWS公式の警告** | "**don't set the parameters to values that generate extensive logging, such as log_statement to all**" | 🅰️ |
| **AWS公式のパスワード露出警告** | `all`/`ddl`/`mod` 設定時、ログにパスワードが平文で記録される可能性 | 🅰️ |
| **AWS公式の影響評価** | "**Increasing database logging impacts storage size, I/O use, and CPU use.**" | 🅰️ |
| **反映タイミング** | 動的（再起動不要） | 🅰️ |
| **出典** | [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging) / [AWS Doc: Mitigating risk of password exposure](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html) / [AWS Database Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) | |

---

### 2-3. `log_destination`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ログ出力先（`stderr`/`csvlog`/`jsonlog`） | 🅰️ |
| **デフォルト値** | `stderr` | 🅰️ |
| **AWS公式のオーバーヘッド記述** | **記述なし**（出力先指定のみで実体は他パラメータの動作に依存） | 🅰️ |
| **反映タイミング** | 動的 | 🅰️ |
| **出典** | [AWS Doc: Configuring slow SQL queries](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_DatabaseInsights.SlowSQL.html) | |

---

### 2-4. `rds.log_retention_period`

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | ローカルログの保持期間（分） | 🅰️ |
| **デフォルト値** | `4320`（3日） | 🅰️ |
| **許容範囲** | 最大 `10080`（7日） | 🅰️ |
| **AWS公式の影響評価** | ローカルストレージ枯渇回避の調整キーとして公式推奨 | 🅰️ |
| **AWS公式の推奨** | "modify the **rds.log_retention_period** parameter to remove unnecessary logs" | 🅰️ |
| **反映タイミング** | 動的 | 🅰️ |
| **出典** | [AWS Database Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) / [AWS re:Post: Local storage issues](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue) / [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging) | |

---

### 2-5. CloudWatch Logs エクスポート

| 項目 | 内容 | 情報源区分 |
|---|---|:---:|
| **役割** | PostgreSQL ログを CloudWatch Logs に転送 | 🅰️ |
| **AWS公式の課金警告** | "**Publishing your PostgreSQL logs to CloudWatch Logs consumes storage, and you incur charges for that storage.**" | 🅰️ |
| **AWS公式の推奨** | "Be sure to **delete any CloudWatch Logs that you no longer need**." | 🅰️ |
| **出典** | [AWS Doc: Publishing Aurora PostgreSQL logs to CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html) | |

---

## 3. 影響評価サマリ（一覧）

### 3-1. AWS公式に影響評価がある項目（信頼度: 高）

| パラメータ | AWS公式の影響評価 | 推奨設定 | 参照元 |
|---|---|---|---|
| `log_min_duration_statement` | ✅ "impacts storage size, I/O use, and CPU use" | `1000`（AWS公式推奨例） | [AWS Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) / [AWS re:Post](https://repost.aws/knowledge-center/rds-postgresql-query-logging) |
| `log_statement` | ✅ "extensive logging" 警告 ／ パスワード露出 | `none`（スロークエリ目的なら） | [AWS re:Post](https://repost.aws/knowledge-center/rds-postgresql-query-logging) / [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html) |
| `rds.log_retention_period` | ✅ ストレージ調整キーとして公式推奨 | デフォルト `4320` | [AWS re:Post](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue) |
| CloudWatch Logs エクスポート | ✅ "consumes storage, and you incur charges" | 保持期間を必ず設定 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html) |

### 3-2. AWS公式に影響評価がない項目（信頼度: 推測）

⚠️ 以下のパラメータについて **AWS公式ドキュメントにはオーバーヘッドの記述がありません**。評価は PostgreSQL の類似機能（`auto_explain`）の挙動からの**推測**です。本番投入前に必ずステージング環境で負荷テストを実施してください。

| パラメータ | AWS公式の影響評価 | PostgreSQLからの推測 | 参照元（推測の根拠） |
|---|---|---|---|
| `aurora_compute_plan_id` | ❌ 記述なし | 第三者評価で "no measurable overhead" | [pganalyze](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora) |
| `aurora_stat_plans.with_costs` | ❌ 記述なし | コスト情報追記のみで影響軽微と推測 | （推測） |
| `aurora_stat_plans.with_analyze` | ❌ 記述なし | PostgreSQL: "extremely negative impact on performance"（auto_explain.log_analyze） | [PostgreSQL Doc](https://www.postgresql.org/docs/current/auto-explain.html) |
| `aurora_stat_plans.with_timing` | ❌ 記述なし | PostgreSQL: "slow down queries significantly"（auto_explain.log_timing） | [PostgreSQL Doc](https://www.postgresql.org/docs/current/auto-explain.html) |
| `aurora_stat_plans.with_buffers` | ❌ 記述なし | PostgreSQLコミュニティ実測 約2% TPS低下（log_buffers） | [PostgreSQL ML](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt) |
| `aurora_stat_plans.with_wal` | ❌ 記述なし | （推測） | （推測） |
| `aurora_stat_plans.with_triggers` | ❌ 記述なし | （推測） | （推測） |
| `aurora_stat_plans.minutes_until_recapture` | ❌ 記述なし | 再キャプチャ時のみ影響と推測 | （推測） |
| `aurora_stat_plans.calls_until_recapture` | ❌ 記述なし | 再キャプチャ時のみ影響と推測 | （推測） |

---

## 4. 重要な公式記述（原文引用）

### 🅰️ AWS公式の記述（明示あり）

#### スロークエリログ / ロギング全般の影響（AWS Database Blog）
> "**Increasing database logging impacts storage size, I/O use, and CPU use. Because of this, test these changes before deploying them in production.**"
> — [AWS Database Blog: Working with RDS and Aurora PostgreSQL logs Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/)

#### `log_statement = all` / `log_min_duration_statement = 0` のリスク（AWS re:Post）
> "To minimize storage usage, **don't set the parameters to values that generate extensive logging**, such as log_statement to all or log_min_duration_statement to 0. Because **the DB instance is unavailable when the volume storage is full**, it's a best practice to modify the rds.log_retention_period parameter to remove unnecessary logs."
> — [AWS re:Post: Turn on query logging with Amazon RDS for PostgreSQL](https://repost.aws/knowledge-center/rds-postgresql-query-logging)

#### CloudWatch Logs課金（AWS公式Doc）
> "**Publishing your PostgreSQL logs to CloudWatch Logs consumes storage, and you incur charges for that storage.** Be sure to delete any CloudWatch Logs that you no longer need."
> — [AWS Doc: Publishing Aurora PostgreSQL logs to CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)

#### `aurora_compute_plan_id` が動的パラメータ（AWS re:Post）
> "This is a dynamic parameter, no reboot required."
> — [AWS re:Post: How to log and analyze query execution plans in Amazon Aurora PostgreSQL](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature)

#### `aurora_stat_plans.with_*` の機能仕様（AWS Doc）
> "The configuration for `aurora_stat_plans.with_*` parameters takes effect only for newly captured plans."
> — [AWS Doc: Monitoring query execution plans for Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html)

> **⚠️ AWS公式ドキュメントには、`aurora_stat_plans.*` 系パラメータのオーバーヘッド・パフォーマンス影響に関する記述は他にありません。**

---

### 🅿️ PostgreSQL公式の記述（類似機能 `auto_explain` についての警告。Aurora固有の保証ではない）

#### `auto_explain.log_analyze` のリスク
> "When this parameter is on, per-plan-node timing occurs for all statements executed, whether or not they run long enough to actually get logged. **This can have an extremely negative impact on performance**."
> — [PostgreSQL Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)

#### `auto_explain.log_timing` のリスク
> "The overhead of repeatedly reading the system clock can **slow down queries significantly on some systems**, so it may be useful to set this parameter to off when only actual row counts, and not exact times, are needed."
> — [PostgreSQL Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)

> **⚠️ 上記は PostgreSQL の標準拡張 `auto_explain` に関する記述です。Aurora の `aurora_stat_plans` が内部で同等の処理を行うかは AWS が明示していないため、Aurora 固有の挙動については AWS の公式保証ではありません。**

---

### 🅼️ 第三者・推測情報（参考程度）

#### `aurora_compute_plan_id` のオーバーヘッド（pganalyze、第三者ブログ）
> "This new integration gives our users on Amazon Aurora access to execution plans not usually retained by Postgres, **without measurable overhead**."
> — [pganalyze: Plan Statistics for Amazon Aurora](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora)

> **⚠️ pganalyze は AWS のパートナーですが、AWS の公式評価ではありません。**

---

## 5. 本番判断の指針

### 5-1. 安全に有効化できる項目（AWS公式記述あり、または影響軽微と推測）

| パラメータ | 設定値 | 判断理由 |
|---|---|---|
| `aurora_compute_plan_id` | `1` | 実行計画キャプチャに必須。第三者評価でオーバーヘッドなし（AWS公式保証ではないが） |
| `log_min_duration_statement` | `1000`（1秒） | AWS公式推奨例。閾値が適切なら影響軽微 |
| `log_statement` | `none` | デフォルト。`all` 設定は AWS公式で明確に非推奨 |
| `log_destination` | `stderr` | デフォルトのまま |
| `rds.log_retention_period` | `4320`（3日） | デフォルト。ストレージ圧迫時は短縮で対処 |

### 5-2. 慎重な判断が必要な項目（AWS公式記述なし）

⚠️ **以下のパラメータは AWS公式にオーバーヘッド評価がないため、本番投入前にステージング環境での負荷テストが必須です。**

- `aurora_stat_plans.with_analyze`
- `aurora_stat_plans.with_timing`
- `aurora_stat_plans.with_buffers`
- `aurora_stat_plans.with_wal`
- `aurora_stat_plans.with_triggers`
- `aurora_stat_plans.minutes_until_recapture`
- `aurora_stat_plans.calls_until_recapture`
#### 推奨される検証手順
1. ステージング環境で本番相当のワークロードを再現
2. ベースライン取得（パラメータ無効状態で CPUUtilization / DBLoad / レイテンシ計測）
3. パラメータ有効化後に同じワークロードで再計測
4. 差分が許容範囲か判定
5. 問題なければメンテナンスウィンドウ中に本番適用
### 5-3. 明確に避けるべき設定（AWS公式が明示的に非推奨）

| 設定 | 根拠 |
|---|---|
| `log_min_duration_statement = 0` | AWS公式: "extensive logging" → ストレージ枯渇でインスタンス停止リスク |
| `log_statement = all` | AWS公式: 同上 + パスワード平文露出リスク |

---

## 6. 監視推奨メトリクス

パラメータ有効化後に監視すべき CloudWatch メトリクス（参考）:

| メトリクス | 確認ポイント |
|---|---|
| `CPUUtilization` | 有効化前後で平均・最大値を比較 |
| `DBLoad` / `DBLoadCPU` | DB Load の上昇傾向 |
| `FreeableMemory` | メモリ逼迫の兆候 |
| `FreeStorageSpace` | ローカルログによる枯渇監視 |
| `ReadIOPS` / `WriteIOPS` | ログ書き込み増加の検出 |
| CloudWatch Logs `IncomingBytes` | ログ流入量とコスト見積もり |

---

## 7. 出典一覧

### 🅰️ AWS 公式ドキュメント
- [Monitoring query execution plans and peak memory for Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html)
- [Parameter reference for Aurora PostgreSQL query execution plans](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters)
- [Configuring your database to monitor slow SQL queries with Database Insights for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_DatabaseInsights.SlowSQL.html)
- [Turning on query logging for your Aurora PostgreSQL DB cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html)
- [Publishing Aurora PostgreSQL logs to Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)
- [Analyzing execution plans with CloudWatch Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Execution-Plans.html)
### 🅰️ AWS 公式ブログ
- [Monitor query plans for Amazon Aurora PostgreSQL](https://aws.amazon.com/blogs/database/monitor-query-plans-for-amazon-aurora-postgresql/)
- [Working with RDS and Aurora PostgreSQL logs: Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/)
### 🅰️ AWS re:Post
- [How to log and analyze query execution plans in Amazon Aurora PostgreSQL](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature)
- [Turn on query logging with Amazon RDS for PostgreSQL](https://repost.aws/knowledge-center/rds-postgresql-query-logging)
- [Optimize query performance in Aurora PostgreSQL-Compatible](https://repost.aws/knowledge-center/rds-postgresql-query-performance)
- [Troubleshoot local storage issues Aurora PostgreSQL](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue)
- [Use Database Insights to log and analyze Aurora PostgreSQL query execution plans](https://repost.aws/knowledge-center/database-insights-aurora-postgresql-query-execution-plans)
### 🅿️ PostgreSQL 公式ドキュメント・コミュニティ（類似機能の参考情報）
- [PostgreSQL Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)
- [PostgreSQL ML: auto_explain.log_analyze の本番運用警告](https://www.postgresql.org/message-id/CAMkU%3D1ync6hzQgJJVZ5B3U2Y%3DMkZ29oMFBkteTOHbpzTc-G-Tg%40mail.gmail.com)
- [PostgreSQL ML: log_buffers オーバーヘッド実測](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt)
### 🅼️ 第三者ソース
- [pganalyze Blog: Plan Statistics for Amazon Aurora](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora)
