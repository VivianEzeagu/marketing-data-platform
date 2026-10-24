# Marketing Data Platform (Databricks + dbt)

A unified, tested and documented model of Facebook Ads and Snapchat Ads performance, built so
marketers, analysts, BI tools and AI agents can all answer spend / impressions / clicks questions
without knowing how each platform structures its data.

## Architecture

```
Facebook CSVs ─┐                                         ┌─► Analysts (SQL / Databricks)
               ├─► bronze_production.marketing_raw ─► dbt ├─► BI tools
Snapchat CSVs ─┘   (raw, all strings, untouched)          └─► AI agents (natural language)

dbt layers
  sources ─► staging (views) ─► int_marketing_ads (ephemeral) ─► marts (tables / incremental)
              rename, cast,      union platforms, impute          fct_marketing_spend
              dedupe latest      nulls, flag imputation           mart_campaign_daily_performance
                                                                  dim_platform_coverage
Observability: Elementary (tests, anomalies, run history) -> edr report
```

| Layer | Models | Materialization | Why |
|---|---|---|---|
| Raw | `bronze_production.marketing_raw.*` | tables (CSV load) | Source of truth, never edited |
| Staging | `stg_facebook_*`, `stg_snapchat_*` | view | Cheap renames and casts, always fresh |
| Intermediate | `int_marketing_ads` | ephemeral | Reusable union logic, never queried directly |
| Mart | `fct_marketing_spend` | incremental (merge) | Largest table, only recent days need reprocessing |
| Mart | `mart_campaign_daily_performance` | table | Small roll-up queried constantly by BI and agents |
| Mart | `dim_platform_coverage` | view | Tiny metadata summary, always current |

## Grain and keys

- `fct_marketing_spend`: **one row per platform + ad + day**. Key: `marketing_spend_key = hash(platform, ad_id, ad_date)`.
  Snapchat stats are aggregated from ad + day + country to ad + day so both platforms share a grain.
- `mart_campaign_daily_performance`: one row per platform + campaign + day.
- Ids are cast to string; they are only unique within a platform.

## Key engineering decisions (driven by profiling the data)

| Finding | Decision |
|---|---|
| Facebook has USD and EUR accounts, Snapchat EUR; no FX rates supplied | `fx_rates` seed (flat, documented simplification). Both `spend_original_currency` and `spend_eur` are exposed. `accepted_values` on `currency_code` fails loudly if a currency without a rate appears. |
| 26% of Facebook rows have null clicks, 1.6% null impressions | Treated as 0. Null-click rows are tiny deliveries (median 24 impressions, ~0.3% of spend); all null-impression rows have zero spend. Rows are flagged with `is_clicks_imputed` / `is_impressions_imputed`, and a warn-level test guards the assumption. |
| Snapchat has several rows per ad-day (country) | Aggregated to ad + day in staging. `data_src` has a single value, so no double counting. |
| Snapchat ad groups have renamed ids (3 duplicates) | Latest version per id kept in staging. |
| **Facebook = 2025, Snapchat = 2024, no overlap** | Documented on every model, exposed in `dim_platform_coverage`, and monitored by a warn-level test. Compare rates (CTR/CPC/CPM) or average daily spend, not totals. |
| Zero clicks / impressions | `try_divide` returns NULL (undefined), never 0 or an error. |
| Late-arriving data | Incremental merge with a 3-day lookback, with a **per-platform** watermark so one platform's later dates never hide the other's late rows. |

Metric definitions: CTR = clicks / impressions, CPC = spend / clicks, CPM = spend / impressions x 1000
(always computed from summed numerators and denominators). Full descriptions are in the dbt docs.

## Data quality

dbt tests protect the mart: unique / not-null keys, `accepted_values` on platform and currency,
`accepted_range >= 0` on spend, impressions and clicks, relationships from stats to ad dimensions,
and singular tests (clicks <= impressions, no spend without impressions, platform overlap).
Elementary adds `volume_anomalies` and `schema_changes` on the fact table. The extract is static,
so anomaly tests have little training data and are set to `warn`.

## Environments and authentication

| Environment | Catalog | Schema | Auth | Triggered by |
|---|---|---|---|---|
| dev | `marketing_dev` | `dbt_vivian` | your PAT or OAuth login (local only) | manual |
| ci | `ci_development` | `ci_pr_<PR number>` (dropped after the run) | CI token | pull request |
| prod | `marketing_prod` | `marketing` | **Service Principal, OAuth M2M, no PAT** | merge to `main` + nightly |

Raw data lives in `bronze_production.marketing_raw` for all environments. Permissions are in
`sql/grants.sql` (least privilege: the production Service Principal can read raw and create objects
in `marketing_prod`, nothing else).

## CI/CD

```
feature branch ─► pull request ─► CI: dbt deps, parse, build (+tests), docs generate
                                       │ must pass (branch protection)
                                       ▼
                                  merge to main ─► Production: dbt build --target prod
                                                   (Service Principal) ─► docs + Elementary report
```

## Setup

1. Create catalogs/schemas and load the 9 CSVs into `bronze_production.marketing_raw` (strings, no type inference).
2. Run `sql/grants.sql` after replacing `<SP_APPLICATION_ID>` and `<YOUR_EMAIL>`.
3. Local:
   ```bash
   python3 -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   cp .env.example .env   # fill in, then: source .env
   dbt deps && dbt build --target dev
   ```
4. Service Principal: Settings > Identity and access > Service principals > Add; generate an OAuth
   secret; give it "Can use" on the SQL warehouse.
5. GitHub repository secrets:
   `DATABRICKS_HOST`, `DATABRICKS_HTTP_PATH`, `DATABRICKS_CI_TOKEN` (CI),
   `DATABRICKS_CLIENT_ID`, `DATABRICKS_CLIENT_SECRET` (production Service Principal).
6. Branch protection on `main`: require a pull request and the `dbt-ci` status check.

## Documentation and observability

```bash
dbt docs generate && dbt docs serve     # lineage graph, column descriptions
edr report --profiles-dir . --profile-target dev   # Elementary report
```

Screenshots: add the lineage graph and Elementary report here.

## Known limitations

- Flat FX rates, not time-varying.
- No date overlap between platforms in the provided extract.
- Imputing null clicks/impressions as 0 is an assumption validated on this extract only.
