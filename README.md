# CI/CD with GitHub Actions: Live Session

Material for the **DataMaroc Bootcamps** live on **YouTube**. Date and time: coming soon.
Speakers: **Marouane Ghoulami** (Senior Data Engineer), **Mohammed Errakho** (Senior Data Engineer · Tech Lead).

| Folder | What's inside |
|---|---|
| [`slides/`](slides/) | The presentation (17 slides, speaker notes included) |
| [`lab/`](lab/) | The hands-on lab: a tiny data pipeline, its tests, its data and the 2 workflows |
| [`cover/`](cover/) | Announcement visuals (16:9 and 4:5) |

## The plan (~1h30)

| Part | Time | What |
|---|---|---|
| Concepts | ~30 min | Why CI/CD, CI vs Continuous Delivery vs Deployment, branches & pull requests, what to test in a data project, GitHub Actions, Docker images, environments & guardrails |
| Live lab | ~50 min | From a pull request to production, with 3 audience challenges |
| Q&A | rest | |

## The lab

**Problem:** *Café Atlas* (fictional) runs coffee shops in 6 Moroccan cities. Its sales pipeline is a small Python script.
Last month a "quick fix" doubled the revenue on the dashboard and nobody noticed for a week. From now on:
every change must be **tested** before it's merged, every merge must be **deployed** the same way (with an approval),
and a broken data file must **never** reach the report.

**The app** (deliberately tiny, the star is the CI/CD): [`lab/cafe_atlas/pipeline.py`](lab/cafe_atlas/pipeline.py) reads the
CSV files of `lab/data/raw/`, removes duplicates, sends broken rows to quarantine, computes the revenue per day and city,
runs data quality checks and writes an HTML report.

| File | Content |
|---|---|
| `lab/cafe_atlas/pipeline.py` | extract → clean → transform → quality checks (exit code 1 if a check fails) |
| `lab/cafe_atlas/report.py` | writes `output/`: `daily_revenue.csv`, `quarantine.csv`, `summary.md`, `index.html` |
| `lab/tests/test_pipeline.py` | 11 unit tests (pytest) |
| `lab/data/raw/sales_2026_10.csv` | 4,000 October sales (+ 5 duplicates and 4 broken rows) |
| `lab/data/new_data/sales_2026_11_week1.csv` | November week 1: arrives **during** the lab ("new data arrives") |
| `lab/data/new_data/sales_2026_11_week2.csv` | November week 2: a broken export 😈 |
| `lab/Dockerfile` | the deployable artifact: code + Python + pandas + data |
| [`lab/workflows/ci.yml`](lab/workflows/ci.yml) | **CI**: lint + unit tests + pipeline with its quality gate, on every pull request |
| [`lab/workflows/cd.yml`](lab/workflows/cd.yml) | **CD**: on every merge to `main`: CI again → build & push the image → approval → deploy to GitHub Pages |

```
 Pull request ──► CI (ci.yml, job "test")                       red   ──► merge blocked (ruleset)
                  ruff ─► pytest ─► pipeline + quality gate ─── green ──► merge allowed
                                                                              │
 Merge to main ──► CD (cd.yml) ◄──────────────────────────────────────────────┘
                   ci (reused) ─► build: docker build + push ghcr.io/<you>/<repo>:<commit>
                                    ─► deploy: ✋ approval (environment github-pages)
                                          ─► docker run <image> ─► report on https://<you>.github.io/<repo>/
```

> **Why are the workflows in `lab/workflows/` and not in `.github/workflows/`?** GitHub only runs the workflows found in
> `.github/workflows/` at the root of the repo. We keep them aside so that we switch them on **live, one by one**,
> through pull requests.

## Before you start: 100% free

- A **GitHub account** ([github.com](https://github.com), free). On a **public** repository, GitHub Actions minutes are
  unlimited and GitHub Pages and the container registry (ghcr.io) are free. No credit card, no cloud account, no Docker on your laptop.
- Optional, to run things locally: **Git**, **Python 3.12** and VS Code. Without them you can do the whole lab in the browser.

## Run it

Commands are given for a local clone (Git). Everything can also be done in the browser: the **Add file**, ✏️ **Edit**
and **Commit changes → Create a new branch → Propose changes** buttons of GitHub.

1. **Your own copy.** On this repo: **Use this template → Create a new repository** → owner: you →
   name `cicd-github-actions-live` → **Public** → **Create repository**.
   *(Use the template, not a fork: with a fork, pull requests target the original repo by default.)*
2. **Look around, run it locally** (optional):
   ```bash
   git clone https://github.com/<you>/cicd-github-actions-live.git
   cd cicd-github-actions-live/lab
   python -m venv .venv
   source .venv/bin/activate              # Windows: .venv\Scripts\activate
   pip install -r requirements-dev.txt
   ruff check .                           # lint
   pytest -v                              # 11 unit tests
   python -m cafe_atlas.pipeline          # → output/index.html (open it in a browser)
   ```
3. **CI, through a pull request.** Add the CI workflow on a branch:
   ```bash
   git switch -c add-ci
   mkdir -p .github/workflows
   cp lab/workflows/ci.yml .github/workflows/ci.yml
   git add .github && git commit -m "Add CI" && git push -u origin add-ci
   ```
   *(Browser: **Add file → Create new file** → name `.github/workflows/ci.yml` → paste the content of `lab/workflows/ci.yml`
   → **Commit changes** → **Create a new branch** `add-ci` → **Propose changes**.)*
   Open the pull request: the **CI / test** check starts at the bottom of the PR. Click **Details** to watch the steps,
   then the run's **Summary** page (data quality numbers + the downloadable `report`). Green → **Merge pull request** → **Delete branch**.
4. **Protect `main`.** **Settings → Rules → Rulesets → New ruleset → New branch ruleset**:
   - name `protect-main`, Enforcement status **Active**, Target branches → **Add target → Include default branch**
   - ✅ **Require a pull request before merging** (required approvals: 0, you're alone)
   - ✅ **Require status checks to pass** → **Add checks** → `test`
   - **Create**. From now on, nobody can push to `main` directly, and a red check blocks the merge.
5. **🎯 Challenge 1: break it on purpose.** A colleague "optimizes" the pipeline: in `lab/cafe_atlas/pipeline.py`, replace
   ```python
   df = raw.drop_duplicates(subset="sale_id").copy()
   ```
   with `df = raw.copy()`, on a branch `remove-dedup`, and open a PR. *Guess before you look: green or red? Which step?*
   → **Unit tests** fail (`test_duplicates_are_removed`) and the **Merge** button is blocked. Close the PR without merging.
6. **CD.**
   1. **Settings → Pages → Build and deployment → Source: GitHub Actions.**
   2. **Settings → Environments → `github-pages`** (create it if it's not there) → ✅ **Required reviewers** → add yourself
      → **Save protection rules**. That's the "✋ approval" before production.
   3. Add `.github/workflows/cd.yml` (copy of `lab/workflows/cd.yml`) through a PR, exactly like step 3 (branch `add-cd`).
      CI runs on the PR → **Merge**.
   4. **Actions → CD**: `ci` → `build` → `deploy` waits. **Review deployments → github-pages → Approve and deploy.**
   5. The `deploy` job shows the URL: `https://<you>.github.io/cicd-github-actions-live/`. The report says which **commit**
      it was built from. The image is in your repo page → **Packages**, tagged with the same commit.
7. **🎯 Challenge 2: new data arrives.** Move the November week 1 file into `raw/` on a branch and open a PR:
   ```bash
   git switch main && git pull
   git switch -c data/november-week1
   git mv lab/data/new_data/sales_2026_11_week1.csv lab/data/raw/
   git commit -m "Add November week 1 sales" && git push -u origin HEAD
   ```
   *(Browser: open the file → ✏️ Edit → in the file name box, type `../raw/` in front of the name → commit to a new branch.)*
   *Guess: how many duplicates and broken rows will the CI summary show?* → green, then **Merge** → **approve** the
   deployment → the report now shows November and the new commit.
8. **🎯 Challenge 3: bad data.** Same thing with `sales_2026_11_week2.csv` (branch `data/november-week2`).
   *Guess: will the pipeline crash, publish wrong numbers, or be stopped?* → **stopped**: the step
   *Run the pipeline + data quality checks* fails, the summary says why, the merge is blocked, production is untouched.
   Close the PR.

**Live challenges:** 🎯 1 (a bug in the code), 🎯 2 (new data), 🎯 3 (bad data): each time, guess before you look.

### Bonus

- **Fix the bad data properly**: make the pipeline understand `15,00` (hint: `.str.replace(",", ".")` before
  `pd.to_numeric`), add a test, open a PR with the fix + the week 2 file: green, merge, approve.
- **Rollback**: **Actions → CD** → an older successful run → re-run its `deploy` job: the older image (older commit) is deployed again.
- **Matrix**: test on several Python versions in `ci.yml` (`strategy: matrix: python-version: ["3.11", "3.12", "3.13"]`).
- **Schedule**: add `schedule: [{cron: "0 6 * * *"}]` under `on:` to run a workflow every morning at 06:00 UTC.

### Reset

Delete your copy (**Settings → General → Delete this repository**) and its image (your profile → **Packages** →
`cicd-github-actions-live` → **Package settings → Delete this package**), then start again from step 1.

## Regenerate the data

```
python lab/tools/generate_data.py
```
Pure Python, no dependency, deterministic: you always get the same files (and the same answers to the challenges).
