# Aurora PostgreSQL Database Insights 関連パラメータ オーバーヘッド評価表

本ドキュメントは Aurora PostgreSQL で Database Insights（実行計画キャプチャ／スロークエリ分析）を有効化する際に設定するパラメータについて、**CPU・メモリ・ディスクへの影響**を整理したものです。

すべての評価は AWS 公式ドキュメント、AWS 公式ブログ、AWS re:Post、PostgreSQL 公式ドキュメント、および PostgreSQL コミュニティの実測値を出典としています。

---

## 1. 実行計画キャプチャ関連パラメータ

### 1-1. `aurora_compute_plan_id`

| 項目 | 内容 |
|---|---|
| **役割** | クエリ実行時にプラン識別子（plan_id）を付与する。実行計画キャプチャの中核パラメータ |
| **デフォルト値** | `off`（Aurora PostgreSQL 14.10、15.5 以上） |
| **推奨値（本番）** | ✅ `1`（on） |
| **CPU 影響** | 🟢 **極小** ― プランツリーのハッシュ計算のみ |
| **メモリ影響** | 🟢 **極小** ― プラン識別子（数バイト）の保持のみ |
| **ディスク影響** | 🟢 **なし** ― ローカルディスク／CloudWatch Logs への書き込みなし |
| **反映タイミング** | 動的（再起動不要） |
| **本番有効化** | ✅ **推奨** |
| **出典** | [pganalyze 実測: "without measurable overhead"](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora) ／ [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html) ／ [AWS re:Post](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature) |

---

### 1-2. `aurora_stat_plans.with_costs`

| 項目 | 内容 |
|---|---|
| **役割** | EXPLAIN プランに各ノードの推定コストと行数を含める |
| **デフォルト値** | `on` |
| **推奨値（本番）** | ✅ `on`（デフォルトのまま） |
| **CPU 影響** | 🟢 **極小** ― プラン文字列生成時のコスト情報追記のみ |
| **メモリ影響** | 🟢 **小** ― プラン保存時の文字列領域がやや増加 |
| **ディスク影響** | 🟢 **小** ― `aurora_stat_plans` ビュー内の保存領域のみ |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | ✅ **推奨**（デフォルトで有効） |
| **出典** | [AWS Doc: Parameter reference for Aurora PostgreSQL query execution plans](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |

---

### 1-3. `aurora_stat_plans.with_analyze` ⚠️ 最重要

| 項目 | 内容 |
|---|---|
| **役割** | EXPLAIN ANALYZE を実行し、**実測（actual）の実行時統計をキャプチャ**。プランが最初にキャプチャされたときのみ適用 |
| **デフォルト値** | `off` |
| **推奨値（本番）** | ⚠️ **慎重に判断**（業務要件次第） |
| **CPU 影響** | 🔴 **大** ― しきい値を超えなかった軽量クエリも含めて**全クエリで per-plan-node timing 計測**が走る |
| **メモリ影響** | 🟡 **中** ― ANALYZE 実行時の一時統計領域 |
| **ディスク影響** | 🟡 **小〜中** ― プラン保存領域の増加 |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | ⚠️ **要負荷テスト** ― ステージング検証必須 |
| **既知の懸念** | PostgreSQL コミュニティで CPU バウンドなクエリで**最大10倍の遅延**事例あり |
| **AWS公式の評価** | AWS は `aurora_stat_plans.with_analyze` 固有のオーバーヘッド数値を公開していない |
| **出典** | [PostgreSQL Doc: "**extremely negative impact on performance**"](https://www.postgresql.org/docs/current/auto-explain.html) ／ [PostgreSQL ML: 10倍遅延事例](https://www.postgresql.org/message-id/CAMkU%3D1ync6hzQgJJVZ5B3U2Y%3DMkZ29oMFBkteTOHbpzTc-G-Tg%40mail.gmail.com) ／ [PostgreSQL ML: 本番非推奨ガイダンス](https://www.postgresql.org/message-id/4EE3F060.6000602%40fuzzy.cz) |

---

### 1-4. `aurora_stat_plans.with_timing`

| 項目 | 内容 |
|---|---|
| **役割** | ANALYZE 使用時、各プランノードの開始時間と所要時間を含める |
| **デフォルト値** | `on` |
| **推奨値（本番）** | `with_analyze=on` 時のみ問題化、必要に応じて `off` で回避 |
| **CPU 影響** | 🔴 **中〜大** ― システムクロックを多用するため、`with_analyze=on` と組み合わせると顕著 |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟢 **小** ― プラン保存時にタイミング情報が追記される |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | ⚠️ `with_analyze=on` の場合に再考。`off` にすると軽量化可能だが、所要時間情報を失う |
| **既知の懸念** | PostgreSQL Doc: "**The overhead of repeatedly reading the system clock can slow down queries significantly on some systems**" |
| **出典** | [PostgreSQL Doc: auto_explain.log_timing](https://www.postgresql.org/docs/current/auto-explain.html) |

---

### 1-5. `aurora_stat_plans.with_buffers`

| 項目 | 内容 |
|---|---|
| **役割** | ANALYZE 使用時、バッファ使用統計（共有/ローカル/ヒット/読み込み）を含める |
| **デフォルト値** | `off` |
| **推奨値（本番）** | ⭕ `with_analyze=on` 時は **有効化推奨**（バッファ統計はチューニングに有用） |
| **CPU 影響** | 🟢 **小** ― PostgreSQL コミュニティ実測で約 **2% のTPS低下** |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟢 **小** ― プラン保存時のバッファ統計追記分 |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | ⭕ `with_analyze=on` の場合は有効化推奨 |
| **出典** | [PostgreSQL コミュニティ実測値（log_buffers_overhead.txt）](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt) |

---

### 1-6. `aurora_stat_plans.with_wal`

| 項目 | 内容 |
|---|---|
| **役割** | ANALYZE 使用時、WAL レコード生成情報を含める |
| **デフォルト値** | `off` |
| **推奨値（本番）** | △ 書き込みヘビーなワークロードで WAL 量を追跡したい場合のみ |
| **CPU 影響** | 🟢 **小** |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟢 **小** ― プラン保存時の WAL 情報追記分 |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | △ **必要時のみ** |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |

---

### 1-7. `aurora_stat_plans.with_triggers`

| 項目 | 内容 |
|---|---|
| **役割** | ANALYZE 使用時、トリガー実行統計を含める |
| **デフォルト値** | `off` |
| **推奨値（本番）** | △ トリガーを多用するスキーマで、トリガー実行時間を追跡したい場合のみ |
| **CPU 影響** | 🟢 **小** |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟢 **小** |
| **反映タイミング** | 動的（新規キャプチャプランから反映） |
| **本番有効化** | △ **必要時のみ** |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |

---

### 1-8. `aurora_stat_plans.minutes_until_recapture`

| 項目 | 内容 |
|---|---|
| **役割** | プラン再キャプチャまでの分数。指定時間経過後に同じプランを再キャプチャ |
| **デフォルト値** | `0`（再キャプチャ無効） |
| **推奨値（本番）** | デフォルト `0` でも実用可能。プランの経時変化を追跡したい場合は `60`（1時間）程度から開始 |
| **CPU 影響** | 🟢 **小** ― 再キャプチャ実行時のみ発生 |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟡 **小〜中** ― 同じプランの複数バージョンを保存するため `aurora_stat_plans` の容量増加 |
| **反映タイミング** | 動的 |
| **本番有効化** | △ 統計の経時変化を見たい場合のみ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |

---

### 1-9. `aurora_stat_plans.calls_until_recapture`

| 項目 | 内容 |
|---|---|
| **役割** | プラン再キャプチャまでの呼び出し回数 |
| **デフォルト値** | `0`（再キャプチャ無効） |
| **推奨値（本番）** | `minutes_until_recapture` と組み合わせて利用 |
| **CPU 影響** | 🟢 **小** |
| **メモリ影響** | 🟢 **小** |
| **ディスク影響** | 🟡 **小〜中** |
| **反映タイミング** | 動的 |
| **本番有効化** | △ 必要時のみ |
| **出典** | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |

---

## 2. スロークエリログ関連パラメータ

### 2-1. `log_min_duration_statement` ⚠️ 値の選定が重要

| 項目 | 内容 |
|---|---|
| **役割** | この時間（ミリ秒）を超えるクエリを PostgreSQL ログに出力。スロークエリ判定の閾値 |
| **デフォルト値** | `-1`（無効） |
| **推奨値（本番）** | ✅ **`1000`（1秒）** から開始、ワークロードに応じて調整 |
| **CPU 影響** | 🟡 **値に依存** ― 閾値が低いほどログ出力 I/O が増加 |
| **メモリ影響** | 🟢 **なし** |
| **ディスク影響** | 🔴 **大（値次第）** ― ローカルストレージ＋CloudWatch Logs に書き込み |
| **反映タイミング** | 動的（再起動不要） |
| **本番有効化** | ✅ **推奨**（適切な閾値で） |
| **避けるべき値** | ❌ `0`（全クエリ記録）／`100` 未満（過度に低い閾値） |
| **AWS公式推奨** | "for example, you can change log_statement to ddl or **log_min_duration_statement to 1000 (1 second)**" |
| **ストレージ影響** | "**The DB instance is unavailable when the volume storage is full**" |
| **出典** | [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging) ／ [AWS re:Post: Optimize query performance](https://repost.aws/knowledge-center/rds-postgresql-query-performance) ／ [AWS Database Blog: Working with logs Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) |

---

### 2-2. `log_statement` ⚠️ 本番では `none` 推奨

| 項目 | 内容 |
|---|---|
| **役割** | SQL ステートメント種別のログ出力（`none`/`ddl`/`mod`/`all`） |
| **デフォルト値** | `none` |
| **推奨値（本番）** | ✅ **`none`** ― スロークエリ目的なら `log_min_duration_statement` のみで十分 |
| **CPU 影響** | 🔴 **大（`all` 時）** ― 全SQL文をログ出力 |
| **メモリ影響** | 🟢 **なし** |
| **ディスク影響** | 🔴 **超大（`all` 時）** ― ローカル＋CW Logs に全SQL書き込みで爆発的増加 |
| **反映タイミング** | 動的（再起動不要） |
| **本番有効化** | ❌ **`all` は本番非推奨** ／ `ddl` は監査要件あれば検討 |
| **既知の懸念** | パスワードが平文でログ記録される可能性（`all`/`ddl`/`mod`） |
| **AWS公式警告** | "**don't set the parameters to values that generate extensive logging, such as log_statement to all**" |
| **出典** | [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging) ／ [Mitigating risk of password exposure when using query logging](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html) |

---

### 2-3. `log_destination`

| 項目 | 内容 |
|---|---|
| **役割** | ログ出力先（`stderr`/`csvlog`/`jsonlog`） |
| **デフォルト値** | `stderr` |
| **推奨値（本番）** | ✅ `stderr`（CloudWatch Logs エクスポートと互換） |
| **CPU 影響** | 🟢 **極小** |
| **メモリ影響** | 🟢 **なし** |
| **ディスク影響** | 🟢 **なし**（出力先指定のみ） |
| **反映タイミング** | 動的 |
| **本番有効化** | ✅ デフォルトの `stderr` で問題なし |
| **出典** | [AWS Doc: Configuring slow SQL queries](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_DatabaseInsights.SlowSQL.html) |

---

### 2-4. `rds.log_retention_period`

| 項目 | 内容 |
|---|---|
| **役割** | ローカルログの保持期間（分） |
| **デフォルト値** | `4320`（3日） |
| **許容範囲** | 最大 `10080`（7日） |
| **推奨値（本番）** | デフォルト `4320` で開始、ストレージ逼迫時は短縮 |
| **CPU 影響** | 🟢 **なし** |
| **メモリ影響** | 🟢 **なし** |
| **ディスク影響** | 🟡 **保持期間に比例** ― ローカルストレージ枯渇回避のキーパラメータ |
| **反映タイミング** | 動的 |
| **本番有効化** | ✅ **必ず設定** ― ストレージ枯渇時は短縮で対処 |
| **出典** | [AWS Database Blog: Working with logs Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) ／ [AWS re:Post: Troubleshoot local storage issues](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue) |

---

## 3. 影響評価サマリ（一覧）

| パラメータ | 推奨設定 | CPU | メモリ | ディスク | 本番判定 | 参照元 |
|---|---|:---:|:---:|:---:|:---:|---|
| `aurora_compute_plan_id` | `1` | 🟢 | 🟢 | 🟢 | ✅ 有効化 | [pganalyze](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora) / [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html) / [AWS re:Post](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature) |
| `aurora_stat_plans.with_costs` | `on`（デフォルト） | 🟢 | 🟢 | 🟢 | ✅ 有効化 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |
| `aurora_stat_plans.with_analyze` | `off`（要検証で `on`） | 🔴 | 🟡 | 🟡 | ⚠️ 要検証 | [PostgreSQL Doc](https://www.postgresql.org/docs/current/auto-explain.html) / [PostgreSQL ML（10倍遅延事例）](https://www.postgresql.org/message-id/CAMkU%3D1ync6hzQgJJVZ5B3U2Y%3DMkZ29oMFBkteTOHbpzTc-G-Tg%40mail.gmail.com) / [PostgreSQL ML（本番非推奨）](https://www.postgresql.org/message-id/4EE3F060.6000602%40fuzzy.cz) |
| `aurora_stat_plans.with_timing` | `on`（デフォルト） | 🔴※ | 🟢 | 🟢 | ⚠️ `with_analyze=on`時注意 | [PostgreSQL Doc](https://www.postgresql.org/docs/current/auto-explain.html) |
| `aurora_stat_plans.with_buffers` | `with_analyze=on`時 `on` | 🟢 | 🟢 | 🟢 | ⭕ 条件付き有効化 | [PostgreSQL コミュニティ実測](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt) |
| `aurora_stat_plans.with_wal` | `off`（必要時 `on`） | 🟢 | 🟢 | 🟢 | △ 必要時 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |
| `aurora_stat_plans.with_triggers` | `off`（必要時 `on`） | 🟢 | 🟢 | 🟢 | △ 必要時 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |
| `aurora_stat_plans.minutes_until_recapture` | `0`（または `60`） | 🟢 | 🟢 | 🟡 | △ 必要時 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |
| `aurora_stat_plans.calls_until_recapture` | `0` | 🟢 | 🟢 | 🟡 | △ 必要時 | [AWS Doc](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters) |
| `log_min_duration_statement` | `1000` | 🟡 | 🟢 | 🔴※ | ✅ 適切な閾値で | [AWS re:Post（クエリログ設定）](https://repost.aws/knowledge-center/rds-postgresql-query-logging) / [AWS re:Post（クエリ最適化）](https://repost.aws/knowledge-center/rds-postgresql-query-performance) / [AWS Database Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) |
| `log_statement` | `none` | 🔴※ | 🟢 | 🔴※ | ❌ `all`は非推奨 | [AWS re:Post（クエリログ設定）](https://repost.aws/knowledge-center/rds-postgresql-query-logging) / [AWS Doc（パスワード露出リスク）](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html) |
| `log_destination` | `stderr` | 🟢 | 🟢 | 🟢 | ✅ デフォルト | [AWS Doc（スロークエリ設定）](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_DatabaseInsights.SlowSQL.html) |
| `rds.log_retention_period` | `4320`〜 | 🟢 | 🟢 | 🟡 | ✅ 必ず設定 | [AWS Database Blog](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) / [AWS re:Post（ローカルストレージ）](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue) |

凡例: 🟢 影響小 ／ 🟡 影響中 ／ 🔴 影響大 ／ 🔴※ 設定値次第で影響大

---

## 4. 重要な公式記述（原文引用）

### `aurora_compute_plan_id`
> "This is a dynamic parameter, no reboot required"
> — [AWS re:Post](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature)

> "This new integration gives our users on Amazon Aurora access to execution plans not usually retained by Postgres, **without measurable overhead**."
> — [pganalyze](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora)

### `with_analyze` / `log_analyze` のリスク（PostgreSQL公式）
> "When this parameter is on, per-plan-node timing occurs for all statements executed, whether or not they run long enough to actually get logged. **This can have an extremely negative impact on performance**."
> — [PostgreSQL 18 Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)

### `log_timing` のリスク（PostgreSQL公式）
> "The overhead of repeatedly reading the system clock can **slow down queries significantly on some systems**, so it may be useful to set this parameter to off when only actual row counts, and not exact times, are needed."
> — [PostgreSQL 18 Doc: auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)

### `log_statement = all` / `log_min_duration_statement = 0` のリスク（AWS公式）
> "To minimize storage usage, **don't set the parameters to values that generate extensive logging**, such as log_statement to all or log_min_duration_statement to 0. Because **the DB instance is unavailable when the volume storage is full**, it's a best practice to modify the rds.log_retention_period parameter to remove unnecessary logs."
> — [AWS re:Post: Turn on query logging](https://repost.aws/knowledge-center/rds-postgresql-query-logging)

### ロギングのリソース影響（AWS公式ブログ）
> "**Increasing database logging impacts storage size, I/O use, and CPU use. Because of this, test these changes before deploying them in production.**"
> — [AWS Database Blog: Working with RDS and Aurora PostgreSQL logs Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/)

### CloudWatch Logs 課金（AWS公式）
> "**Publishing your PostgreSQL logs to CloudWatch Logs consumes storage, and you incur charges for that storage.** Be sure to delete any CloudWatch Logs that you no longer need."
> — [AWS Doc: Publishing Aurora PostgreSQL logs to CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)

---

## 5. 出典一覧

### AWS 公式ドキュメント
- [Monitoring query execution plans and peak memory for Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html)
- [Parameter reference for Aurora PostgreSQL query execution plans](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#AuroraPostgreSQL.Monitoring.Query.Plans.Parameters)
- [Configuring your database to monitor slow SQL queries with Database Insights for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_DatabaseInsights.SlowSQL.html)
- [Turning on query logging for your Aurora PostgreSQL DB cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.Query_Logging.html)
- [Publishing Aurora PostgreSQL logs to Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)
- [Analyzing execution plans with CloudWatch Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Execution-Plans.html)
### AWS 公式ブログ
- [Monitor query plans for Amazon Aurora PostgreSQL](https://aws.amazon.com/blogs/database/monitor-query-plans-for-amazon-aurora-postgresql/)
- [Working with RDS and Aurora PostgreSQL logs: Part 1](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/)
### AWS re:Post
- [How to log and analyze query execution plans in Amazon Aurora PostgreSQL](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature)
- [Turn on query logging with Amazon RDS for PostgreSQL](https://repost.aws/knowledge-center/rds-postgresql-query-logging)
- [Optimize query performance in Aurora PostgreSQL-Compatible](https://repost.aws/knowledge-center/rds-postgresql-query-performance)
- [Troubleshoot local storage issues Aurora PostgreSQL](https://repost.aws/knowledge-center/postgresql-aurora-storage-issue)
- [Use Database Insights to log and analyze Aurora PostgreSQL query execution plans](https://repost.aws/knowledge-center/database-insights-aurora-postgresql-query-execution-plans)
### PostgreSQL 公式ドキュメント・コミュニティ
- [PostgreSQL Doc: auto_explain（log_analyze の "extremely negative impact" 警告）](https://www.postgresql.org/docs/current/auto-explain.html)
- [PostgreSQL ML: auto_explain.log_analyze の本番運用警告（10倍遅延事例）](https://www.postgresql.org/message-id/CAMkU%3D1ync6hzQgJJVZ5B3U2Y%3DMkZ29oMFBkteTOHbpzTc-G-Tg%40mail.gmail.com)
- [PostgreSQL ML: 本番でのスロークエリログレベル別ガイダンス](https://www.postgresql.org/message-id/4EE3F060.6000602%40fuzzy.cz)
- [PostgreSQL ML: log_buffers オーバーヘッド実測（約2%）](https://www.postgresql.org/message-id/attachment/177027/log_buffers_overhead.txt)
### 第三者ソース（参考）
- [pganalyze Blog: Plan Statistics for Amazon Aurora（aurora_stat_plans のオーバーヘッド評価）](https://pganalyze.com/blog/introducing-postgres-plan-statistics-for-amazon-aurora)
