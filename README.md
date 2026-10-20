# Marketing Data Platform with Databricks and dbt

## 1. Project Overview

This project consolidates Facebook Ads and Snapchat Ads performance data into a unified marketing data platform using Databricks, Unity Catalog and dbt.

The platform provides a standardized, documented and tested dataset for marketing analysis, business intelligence and downstream analytics.

## 2. Business Objectives

- Consolidate advertising spend, impressions and clicks.
- Compare campaign performance across advertising platforms.
- Calculate CTR, CPC and CPM consistently.
- Provide reusable and governed analytical data.

## 3. Architecture

Source CSV files → Databricks Bronze → dbt staging → Intermediate model → Final marketing fact table → SQL and BI consumers.

## 4. Data Sources

Facebook and Snapchat files, their source fields, data types, date coverage and known limitations.

## 5. Data Model

grain, key columns, deduplication logic, platform normalization and metric definitions for each model.

## 6. Data Quality

dbt tests, the business rules they protect and how to run them.

## 7. Documentation and Observability

generate dbt documentation, view model lineage and use Elementary to monitor test results.

## 8. Environments and Security

CI, development and production targets, Unity Catalog permissions, secrets management and production Service Principal authentication.

## 9. CI/CD

pull-request validation process and the production deployment workflow.

## 10. Setup and Usage

prerequisites, environment variables, dependency installation, dbt profile configuration and the commands required to build and test the project.

## 11. Engineering Decisions and Limitations
