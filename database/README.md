# Amazon Aurora PostgreSQL における Database Insights まとめ

## Database Insights とは
主な特徴：
- AWS側で提供のダッシュボードが利用可能
- アプリケーション・データベース・OS のログ／メトリクスを単一のコンソールビューに統合
- DB フリートを横断して、パフォーマンスが低下しているインスタンスをドリルダウンで特定可能
- 既存の **Performance Insights** の機能をベースに、監視・分析・最適化機能を拡張したもの


**参照**:
- [CloudWatch Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)

---

## 動作モード

Database Insights には2つのモードがある、

| モード | 概要 | 課金 |
|---|---|---|
| **Standard** | デフォルトで有効。Performance Insights の基本機能をベースとした標準モード。 | 既存の Performance Insights 無料枠と同等 |
| **Advanced** | 機能を拡張した上位モード。15か月のメトリクス保持、SQL 実行計画キャプチャ、オンデマンド分析、ロック分析等。 | 追加料金: $0.0125/vCPU(or ACU)-hour／インスタンス |

---

## Standard モードで確認できること

Aurora PostgreSQL の DB クラスター作成時にデフォルトで有効化されており、以下の項目を確認できる。

[CloudWatch Database Insights のデータベースインスタンスダッシュボードの表示](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Database-Instance-Dashboard.html)

## Advanced モードでさらに確認できること

詳細は以下にてまとまっている

[データベース分析のためのモード](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html#Database-Insights-modes)


## 各機能を利用するために必要な設定

### 実行計画

#### 前提条件
- Aurora PostgreSQL バージョン **14.10、15.5 以上**
#### 必須パラメータ

| パラメータ名 | 設定値 | 反映 | 説明 |
|---|---|---|---|
| `aurora_compute_plan_id` | on | 動的（再起動不要） | クエリ実行時に実行計画識別子を付与する。 |

> 「動的（再起動不要）」の根拠: AWS re:Post 公式記事 [How to log and analyze query execution plans in Amazon Aurora PostgreSQL for performance tuning using the Performance Insights Advanced feature](https://repost.aws/articles/ARP1e3deh9RxC7F3rcBM634A/how-to-log-and-analyze-query-execution-plans-in-amazon-aurora-postgresql-for-performance-tuning-using-the-performance-insights-advanced-feature) 。

[CloudWatch Database Insights を使用した実行計画の分析](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Execution-Plans.html)

その他実行計画関連のパラメータ
[Aurora PostgreSQLのクエリ実行プランとピークメモリの監視](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Monitoring.Query.Plans.html#aurora.with_analyze)

実行計画はDB側で自動作成、エンジニアはインデックス、統計情報の更新（→これが古いと間違った経路を踏む可能性がある）、SQLの修正などで改善

| パラメータ名 | デフォルト | 説明 |
|---|---|---|
| `aurora_stat_plans.minutes_until_recapture` | `0` | プラン再キャプチャまでの分数。`0` で再キャプチャ無効。許容値 `0-1073741823`。 |
| `aurora_stat_plans.calls_until_recapture` | `0` | プラン再キャプチャまでの呼び出し回数。`0` で再キャプチャ無効。許容値 `0-1073741823`。 |
| `aurora_stat_plans.with_costs` | `on` | EXPLAIN プランに各ノードの推定コストと行数を含める。 |
| `aurora_stat_plans.with_analyze` | `off` | EXPLAIN ANALYZE を実行し、**実測（actual）の実行時統計をキャプチャ**。プランが最初にキャプチャされたときのみ適用。 |
| `aurora_stat_plans.with_timing` | `on` | ANALYZE 使用時、各プランノードの開始時間と所要時間を含める。 |
| `aurora_stat_plans.with_buffers` | `off` | ANALYZE 使用時、バッファ使用統計を含める。 |
| `aurora_stat_plans.with_wal` | `off` | ANALYZE 使用時、WAL レコード生成情報を含める。 |
| `aurora_stat_plans.with_triggers` | `off` | ANALYZE 使用時、トリガー実行統計を含める。 |


### 6-2. スロー SQL クエリ分析を有効化するための設定

#### 前提条件
- CloudWatch Logs へのログエクスポート権限および有効化

#### パラメータグループの設定


| パラメータ名 | 推奨値 | 説明 |
|---|---|---|
| `log_min_duration_statement` | `1000`（1秒） | この時間（ミリ秒）を超えるクエリをログ出力する。**スロークエリ判定の閾値**。`-1` で無効。`0` で全クエリ出力（本番では非推奨）。 |
| `log_statement` | `none` | SQL ステートメント種別のログ出力。スロークエリ目的なら `none` が推奨（`all` 等にするとログ量が爆発的に増える）。 |
| `log_destination` | `stderr` | ログ出力先。 |


#### 補足: パスワード露出リスクへの注意

`log_statement` を `all`、`ddl`、`mod` のいずれかに変更する場合、ログにパスワードが平文で記録される可能性があります。リスク軽減策については公式ドキュメント "Mitigating risk of password exposure when using query logging" を参照してください。

### 5-3. OS プロセス単位メトリクスを有効化するための設定

#### 前提条件
- Amazon RDS Enhanced Monitoring** の有効化




### aiagでは不要かと思いますが、一応

複数アカウント・複数リージョンの DB フリートを統合監視も可能

1. **CloudWatch cross-account observability** のセットアップ
   - CloudWatch Observability Access Manager でモニタリングアカウントを指定
   - 各ソースアカウントからモニタリングアカウントへのリンクを作成
2. **cross-account cross-region CloudWatch console** のセットアップ
3. **Database Insights 用のデータ共有設定**
   - cross-account cross-region CloudWatch console のセットアップ画面で **"Include read-only access for Database Insights"** を選択
   - これによりモニタリングアカウントのユーザーが Database Insights テレメトリを参照可能になる
**参照**:
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Cross-Account-Cross-Region.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Cross-Account-Cross-Region.html
---

## 料金（Pricing）

| 項目 | 料金 |
|---|---|
| **Standard モード** | 無料（Performance Insights の無料枠と同等） |
| **Advanced モード（Aurora プロビジョンド）** | $0.0125 / vCPU-hour（約 $9/vCPU/月） |

**参照**:
- https://aws.amazon.com/cloudwatch/pricing/
---
