# 2025 GitHub Tech-Adoption Analysis

**The question:** Stars mix Linux with a weekend experiment. If you are deciding where a stack is actually being adopted — not just starred — which repositories are mature hits, and which are still niche?

This is a BA882 (Team 8) adoption study. It is not a “trending repos” list. It is a weekly **cut of the open-source market**: pull what GitHub already publishes, read the README the way a reviewer would, then group repos by *how* they are used — incumbents, community hits, research stacks, engineering tooling.

**How we built that:** a weekly **GCP** pipeline — **GitHub API → Cloud Functions → GCS / BigQuery**, **GPT-4.1-mini** README labels, then **StandardScaler + K-Means (k = 4)** — shipped to **Streamlit** and a Slack success/fail ping. Airflow (Astronomer) runs it weekly (Saturday 8pm Boston).

Deck: [`docs/presentation.pptx`](docs/presentation.pptx)

Team: Manyi Hong (GCP + Airflow), Emre Can Baykurt, Sanjal Atul Desai, Zehui Wang.

---

## Decision this supports

Stars and forks are easy to rank and easy to misread. A ten-year OS kernel and a new CV library can sit in the same “top 300” list.

Two decisions the work is built for:

1. **What kind of adoption is this?** — mature ecosystem (simple stack, general users), high-complexity community hit (ops / security), multi-language research stack, or engineering tooling (devops / web servers).
2. **What not to do with the rank** — do not treat raw stars as “what’s next.” High stars + a simple stack is often an incumbent. High complexity + a researcher / ML-engineer audience is a different bet.

---

## What we found

Working set: **~300** highly starred GitHub repositories (extract cap `limit=300`), with contributors, commits, languages, and READMEs. After the four summary tables are joined, an LLM adds `category`, `complexity`, `tech_stack`, `audience`, and `use_case`. K-Means then cuts the numeric table into **four** groups.

**The useful cut is the cluster, not the star sort.** The four signatures (from the decision deck):

| Cluster | Read as | What the data shows |
|---|---|---|
| 0 | **Mature ecosystem** | High stars / forks / watchers, older repos, **lower** LLM complexity, simple stack — general users and sysadmins |
| 1 | **High-complexity community hits** | High stars with strong forks, complexity **> 1.5**, ~48 active contributors, multi-language — backend, security, IT ops; monitoring, networking, deployment |
| 2 | **Multi-language innovation** | High stars, less intense commit activity, **large language mix**, complexity **2.0–2.2** — researchers and ML engineers; CV, graphics, ML, data processing |
| 3 | **Engineering-driven** | High stars and watchers, stronger forks-per-star, **highest** complexity (**~2.5+**), rich stack — web servers, devops, automation, deployment |

**What we would not claim:** this is not a forecast of next year’s winners, and it is not a sample of “all of GitHub.” The 300-repo cap is **star-biased toward incumbents**. Phase 2 named the next step: widen coverage (300 → 500) and try a clustering method that is less forced than k = 4.

One LLM label was too cheap to use as a signal: `cli-automation` showed up on **90%+** of repos, so the dashboard drops it from the category → use-case flow.

---

## How we got there (short)

| Step | Why it mattered |
|---|---|
| Five GitHub extracts (repos, contributors, commits, languages, READMEs) | Stars alone cannot separate an incumbent from a research stack |
| Parse JSON in GCS → BigQuery raw tables | The weekly snapshot has to be a table before any join |
| Four summary transforms, then an ML join (`repo_ml_dataset`) | Popularity, activity, contributor structure, language mix on one row |
| GPT-4.1-mini on the README | Category / complexity / audience / use case — the labels a reviewer would write |
| Filter missing signals, impute 0, `StandardScaler`, K-Means (k = 4) | Star counts would otherwise dominate every distance |
| Cluster signature (mean / median) + Streamlit + Slack | The output someone can actually use on Monday |

---

## What’s in the repo

| Path | |
|---|---|
| [`docs/presentation.pptx`](docs/presentation.pptx) | Decision deck |
| [`airflow/dags/github_pipeline_v3.py`](airflow/dags/github_pipeline_v3.py) | Weekly orchestration |
| `cloud_functions/` | Extract → parse → transform → LLM → cluster |
| `streamlit_app/` | Overview, deep dive, cluster view, GenAI labels |

Secrets stay out of git: GitHub token in **Secret Manager**, `OPENAI_API_KEY` and `SLACK_WEBHOOK_URL` in the function / Airflow environment.

---

## How we built it (technical)

Stack: **GitHub API** → **Cloud Functions** (extract / parse / transform) → **Cloud Storage** + **BigQuery** → **GPT-4.1-mini** structured README JSON → **scikit-learn** `StandardScaler` + `KMeans(n_clusters=4, n_init=10)` → **Streamlit** + Slack. Orchestration: **Airflow on Astronomer**, DAG `github_data_pipeline_v3`, `schedule="@weekly"`.

**Extract.** Five HTTP functions, `limit=300`: repos, contributors, commits, languages, READMEs. Token from Secret Manager. Raw JSON lands in GCS (`ba882-t8-github`), then `raw-parse-github` loads dated snapshots into `raw_github_data`.

**Transform.** Four clean summaries (`repos_summary`, contributors, commits, languages). `transform-repos-ml` joins them into `cleaned_github_data.repo_ml_dataset` (~19 numeric fields: stars, forks, watchers, issues, age, stars/day, commits, merge/bot ratios, contributors, language mix, `commits_per_contributor`, `forks_per_star`). The live DAG currently jumps from the four summaries to LLM enrich — the join function is in the repo and is what the later tables read; it should sit immediately before enrich.

**LLM enrich.** `transform_repos_llm_enrich` pulls the README (`master` / common casings) and asks **GPT-4.1-mini** (`temperature=0`) for a closed JSON schema: category (AI/ML, DevOps, …), `tech_stack`, `complexity_level` / `complexity_score`, `audience`, `use_cases`. Output: `repo_llm_enriched`. `repo_ml_ready` left-joins that onto the numeric table.

**Numeric encode + cluster.** `ml_transform_final` keeps numerics (NaN → 0), bools → int, timestamps → unix, strings → a simple label encode, writes `repo_ml_numeric_final`. `cluster-repos` scales all non-name columns and fits K-Means (k = 4, `random_state=42`). The Cloud Function writes the **cluster-mean signature** to `ml_results.repo_cluster_summary`. The Streamlit cluster page **re-fits** K-Means in the app on a shorter feature list and still uses the older names (High Activity / Growing / Mature / Niche). The **decision labels in the deck** are the LLM-enriched reading above — that is the cut we present.

**Notify + viz.** Slack on all-success / one-failed (webhook from the environment, not the repo). Streamlit: repository overview, deep dive, cluster scatter, GenAI (category pie, Sankey, language × category heatmap, sunburst).

**Limits that change how you should use it.** Star-top-300 selection bias. Forced k = 4 (Phase 2 flagged DBSCAN / hierarchical as the next test). `cli-automation` is too common to interpret. Image is a weekly batch, not a live search index. `OPENAI_API_KEY`, the GitHub token, and `SLACK_WEBHOOK_URL` never belong in the tree.

---

## Setup

The pipeline is meant to run on **GCP + Astronomer**, not as a single local script. You need a GCP project, BigQuery datasets `raw_github_data` / `cleaned_github_data` / `ml_results`, deployed Cloud Functions, and:

- a GitHub token in Secret Manager
- `OPENAI_API_KEY` on the enrich function
- `SLACK_WEBHOOK_URL` on the Airflow environment

```sh
# Streamlit (reads BigQuery; needs Application Default Credentials)
cd streamlit_app
pip install -r requirements.txt
streamlit run app.py
```

```sh
# Airflow (Astronomer CLI)
cd airflow
astro dev start
```

---

## License

MIT. See [LICENSE](LICENSE).
