# GitHub Tech-Adoption Pipeline

BA882 Team 8 — Manyi Hong, Emre Can Baykurt, Sanjal Atul Desai, Zehui Wang.

A weekly **GCP** pipeline that turns GitHub repository metadata into an analysis-ready ML table, enriches READMEs with an LLM, clusters repos with **K-Means (k=4)**, and ships the result to **Streamlit** plus a Slack success/fail ping.

Deck: [`docs/presentation.pptx`](docs/presentation.pptx)

---

## What it does

1. **Extract** — Cloud Functions pull repos, contributors, commits, languages, and READMEs from the GitHub API (Secret Manager for the token). Raw JSON lands in **Cloud Storage**.
2. **Parse / transform** — Functions clean JSON into **BigQuery** summary tables (repo / contributor / commit / language).
3. **LLM enrich** — Read the README, add `category`, `complexity`, `tech_stack`, `audience`, `use_case`.
4. **Cluster** — Filter missing signals, impute 0, `StandardScaler`, K-Means with 4 clusters, write a signature (mean/median) per cluster.
5. **Notify + viz** — Slack on success/fail. Streamlit for overview, deep dive, cluster analysis, GenAI insights.

Airflow (Astronomer) runs the DAG **weekly, Saturday 8pm Boston**.

---

## Cluster story (from the deck)

| Cluster | Read as |
|---|---|
| 0 | Mature ecosystem — high stars/forks/watchers, older, simpler stack |
| 1 | High-complexity community hits — active contributors, multi-language, ops/security audience |
| 2 | Multi-language innovation — CV / ML / data, researchers and ML engineers |
| 3 | Engineering-driven — rich stack, devops / automation / web servers |

---

## Layout

```
airflow/dags/github_pipeline_v3.py   # weekly orchestration
cloud_functions/                     # extract → parse → transform → LLM → cluster
streamlit_app/                       # dashboard
docs/presentation.pptx
```

Set `SLACK_WEBHOOK_URL` and GitHub token in Secret Manager / Airflow variables. Do not commit keys.

---

## Stack

GitHub API · Cloud Functions · Cloud Storage · BigQuery · Airflow / Astronomer · LLM enrichment · scikit-learn K-Means · Streamlit · Slack

---

## License

MIT. See [LICENSE](LICENSE).
