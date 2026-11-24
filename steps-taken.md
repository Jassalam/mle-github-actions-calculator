# Steps Taken: GitHub Actions and CI/CD for ML

A record of every step completed for this module, with the commands run and the results observed.

| Lesson | Status |
| --- | --- |
| 01 - Intro to GitHub Actions | Complete |
| 02 - Intro to DVC | Complete |
| 03 - CI/CD Workflow with CML | Complete |
| 04 - MLflow Model Tracking | Complete |
| 05 - CI Review and Troubleshooting | Complete, PR #1 merged |

## Repositories

| Repository | Visibility | Used for |
| --- | --- | --- |
| [Jassalam/mle-github-actions-calculator](https://github.com/Jassalam/mle-github-actions-calculator) | Public | Lesson 1 |
| [Jassalam/mle-github-actions-cicd](https://github.com/Jassalam/mle-github-actions-cicd) | Private | Lessons 2 to 5 |

## Environment Check

Before starting, the local setup was verified:

| Item | Result |
| --- | --- |
| uv | 0.13.7 (requires 0.13.5 or newer) |
| Python in `.venv` | 3.13.18 |
| `libomp` (needed by XGBoost on macOS) | Installed |
| GitHub CLI | Logged in as `Jassalam` |

---

## Lesson 1: Intro to GitHub Actions

### 1. Create the repository

Created a new, empty, public repository on GitHub: `mle-github-actions-calculator`.

### 2. Clone it and set up the uv project

```bash
cd /Users/sukhman/course_module
git clone https://github.com/Jassalam/mle-github-actions-calculator.git
cd mle-github-actions-calculator
echo "3.13" > .python-version
uv init --bare
uv add --dev pytest
```

This created `pyproject.toml` and `uv.lock` with pytest 9.2.0.

### 3. Add the code, tests, and workflow

`src/calculator.py`:

```python
def add(a, b):
    return a + b
```

`tests/test_calculator.py`:

```python
from src.calculator import add


def test_add():
    assert add(1, 2) == 3
    assert add(-2, 2) == 0
    assert add(0, 0) == 0
```

`.github/workflows/test.yaml`:

```yaml
name: Tests

on:
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v6

      - name: Set up uv
        uses: astral-sh/setup-uv@v7
        with:
          version: "0.13.5"

      - name: Install dependencies
        run: uv sync --locked

      - name: Run tests
        run: uv run python -m pytest
```

A `.gitignore` was also added to keep `.venv/`, `__pycache__/`, and `.pytest_cache/` out of Git.

Local test run: `uv run python -m pytest` gave **1 passed**.

### 4. Push the starter files to `main`

```bash
git add .gitignore .python-version pyproject.toml uv.lock src tests .github
git commit -m "Add calculator, tests, and GitHub Actions test workflow"
git push -u origin main
```

Commit: `87d5123`.

### 5. Protect `main`

In **Settings > Branches > Add classic branch protection rule**:

- Branch name pattern: `main`
- Require a pull request before merging: checked
- Require approvals: checked (1 approval)
- Require status checks to pass before merging: checked

Verified with `gh api repos/Jassalam/mle-github-actions-calculator/branches/main/protection`.

### 6. Open a pull request (exercise: add `subtract`)

```bash
git checkout -b add-subtract
```

Added to `src/calculator.py`:

```python
def subtract(a, b):
    return a - b
```

Added to `tests/test_calculator.py`:

```python
def test_subtract():
    assert subtract(5, 3) == 2
    assert subtract(0, 4) == -4
    assert subtract(-1, -1) == 0
```

```bash
git commit -am "Add subtract function with tests"
git push -u origin add-subtract
gh pr create --base main --head add-subtract --title "Add subtract function"
```

Opened [PR #1](https://github.com/Jassalam/mle-github-actions-calculator/pull/1). The **Tests** workflow started automatically and **passed**.

### 7. Break a test on purpose, then fix it

| Commit | Change | Check result |
| --- | --- | --- |
| `2b6f429` | Add subtract function with tests | Pass |
| `8549a32` | Changed `subtract(5, 3) == 2` to `== 99` | **Fail** |
| `0ab4cee` | Restored `== 2` | Pass |

The failure log showed:

```text
>       assert subtract(5, 3) == 99
E       assert 2 == 99
FAILED tests/test_calculator.py::test_subtract - assert 2 == 99
```

### 8. Require the `test` check and merge

The `test` check can only be selected as required after it has run once, so it was added after the first PR run:

```bash
gh api -X PATCH repos/Jassalam/mle-github-actions-calculator/branches/main/protection/required_status_checks \
  -f 'contexts[]=test' -F strict=false
```

The PR showed `BLOCKED / REVIEW_REQUIRED` because a repository owner cannot approve their own PR. It was merged with the admin override:

```bash
gh pr merge 1 --squash --delete-branch --admin
```

`main` is now at `fb4fbfb Add subtract function (#1)`.

---

## Lesson 2: Intro to DVC

All steps below ran in `mle-github-actions-cicd`.

### 1. Download the data

```bash
curl -L --fail -o data/green_tripdata_2025-01.parquet \
  https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-01.parquet
```

File size: 1.1 MB (1,178,451 bytes). The Parquet file is ignored by Git (`data/*.parquet` in `.gitignore`).

### 2. Initialize DVC and track the file

```bash
uv run dvc init
uv run dvc add ./data/green_tripdata_2025-01.parquet
```

This created `data/green_tripdata_2025-01.parquet.dvc`:

```yaml
outs:
- md5: 110776612e1b020e9c4a0e41f16da597
  size: 1178451
  hash: md5
  path: green_tripdata_2025-01.parquet
```

```bash
git add .dvc/.gitignore .dvc/config .dvcignore data/green_tripdata_2025-01.parquet.dvc
git commit -m "Track taxi data with DVC"
```

Commit: `a6d76ea`.

### 3. Add a local DVC remote and push the data

```bash
mkdir -p ../dvc-local-remote
uv run dvc remote add -d localremote ../dvc-local-remote
git add .dvc/config
git commit -m "Add local DVC remote"
uv run dvc push
```

Commit: `f7c9d06`. Result: **1 file pushed**.

The resulting `.dvc/config`:

```ini
[core]
    remote = localremote
['remote "localremote"']
    url = ../../dvc-local-remote
```

The URL is relative to the `.dvc/` folder, so it points to `/Users/sukhman/course_module/dvc-local-remote`.

### 4. Push the commits to GitHub

```bash
git push origin main
```

### Exercises

| Exercise | Result |
| --- | --- |
| Which fields does Git track in the `.dvc` file? | Only `md5`, `size`, `hash`, and `path`. The data itself is not in Git. |
| `dvc status` before and after changing the Parquet file | After appending a byte: `modified: data/green_tripdata_2025-01.parquet`. After restoring: `Data and pipelines are up to date.` |
| Inspect `../dvc-local-remote` after `dvc push` | Object stored at `files/md5/11/0776612e1b020e9c4a0e41f16da597`, named by its hash. |
| Clone into a separate folder and `dvc pull` | `1 file fetched and 1 file added`. The md5 of the pulled file matched the original. The test clone was deleted afterwards. |

---

## Lesson 3: CI/CD Workflow with CML

### 1. Train locally (baseline)

```bash
export MLFLOW_TRACKING_URI="sqlite:///mlflow.db"
uv run python src/train.py --cml-run
cat metrics.txt
```

```text
# Training Metrics

- RMSE on the train set: 4.6850
- RMSE on the test set: 5.0074
- Rows after filtering: 46307
```

This matched the expected values in the lesson exactly and created MLflow model version 1.

### 2. Create a branch and add a metric (exercise)

```bash
git checkout -b cml-experiment
```

Changes to `src/train.py` (in `mle-github-actions-cicd`):

- Recorded `rows_before_filtering = len(df)` before preprocessing.
- Added `rows_before_filtering` to the MLflow metrics.
- Added `- Rows before filtering: ...` to `metrics.txt`.
- Set the `developer` tag from `<your-name>` to `sukhman`.

```bash
git commit -am "Add rows_before_filtering metric and set developer tag"
git push -u origin cml-experiment
gh pr create --base main --head cml-experiment \
  --title "CML experiment: extra metric and n_estimators change"
```

Opened [PR #1](https://github.com/Jassalam/mle-github-actions-cicd/pull/1). The CML workflow passed in 40s and posted:

```text
# Training Metrics

- RMSE on the train set: 4.6850
- RMSE on the test set: 5.0074
- Rows before filtering: 48326
- Rows after filtering: 46307
```

### 3. Change one training parameter (exercise)

```diff
-        xgb.XGBRegressor(n_estimators=100, random_state=42, n_jobs=-1),
+        xgb.XGBRegressor(n_estimators=200, random_state=42, n_jobs=-1),
```

```bash
git commit -am "Increase n_estimators to 200"
git push
```

The new CML comment, compared with the previous one:

| Metric | n_estimators=100 | n_estimators=200 | Change |
| --- | --- | --- | --- |
| RMSE train | 4.6850 | 4.4785 | -0.21 |
| RMSE test | 5.0074 | 4.9639 | -0.04 |
| Gap (test - train) | 0.32 | 0.49 | Wider |
| Rows before filtering | 48326 | 48326 | Same |
| Rows after filtering | 46307 | 46307 | Same |

Train error improved about five times more than test error, which is a mild overfitting signal.

### 4. Break the data URL on purpose, then restore it (exercise)

In `.github/workflows/cml.yaml`, the URL was changed from `green_tripdata_2025-01.parquet` to `green_tripdata_2099-01.parquet`.

```bash
git commit -am "Break data URL on purpose"
git push
```

Workflow steps in the failed run:

| Step | Result |
| --- | --- |
| Install dependencies | Success |
| Download taxi data | **Failure** |
| Train model | Skipped |
| Publish CML report | Skipped |

Error in the log:

```text
curl: (22) The requested URL returned error: 403
##[error]Process completed with exit code 22.
```

No CML comment was posted for this run. The URL was then restored:

```bash
git commit -am "Restore data URL"
git push
```

The workflow passed again in 54s and posted a third comment with the same metrics as the `n_estimators=200` run.

### Branch commits

| Commit | Message | CI result |
| --- | --- | --- |
| `f897172` | Add rows_before_filtering metric and set developer tag | Pass |
| `63b7a65` | Increase n_estimators to 200 | Pass |
| `2c24938` | Break data URL on purpose | **Fail** |
| `8a64cb1` | Restore data URL | Pass |

---

## Lesson 4: MLflow Model Tracking

### 1. Create a second run to compare

With the branch checked out (`n_estimators=200`), the model was trained again locally:

```bash
export MLFLOW_TRACKING_URI="sqlite:///mlflow.db"
uv run python src/train.py --cml-run
```

### 2. Runs and registered model versions

| Model version | Run ID | n_estimators | RMSE train | RMSE test | Rows before | Rows after | Developer tag |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | `33594ee6` | 100 | 4.6850 | 5.0074 | Not logged | 46307 | `<your-name>` |
| 2 | `37cba227` | 100 | 4.6850 | 5.0074 | 48326 | 46307 | sukhman |
| 3 | `7ac42f7d` | 200 | 4.4785 | 4.9639 | 48326 | 46307 | sukhman |

All three model versions have status `READY` under the registered model `green-taxi-trip-duration-xgb`.

### 3. Open the MLflow UI

Port 5001 was used instead of the default 5000:

```bash
uv run mlflow ui --backend-store-uri sqlite:///mlflow.db --port 5001
```

UI address: <http://127.0.0.1:5001>

What to inspect in the UI:

1. **Experiments > green-taxi-trip-duration-xgb**: select runs 2 and 3 and click **Compare**.
2. **Run > Overview**: tags for `dataset`, `target`, `features`, and `developer`.
3. **Models > green-taxi-trip-duration-xgb**: versions 1 to 3, each linked to its run.

### Observations

- Run 1 has no `rows_before_filtering` and still has the `<your-name>` tag, because it ran before the branch changes. This shows why consistent metrics and tags matter when comparing runs.
- The `features` tag says `PULocationID,DOLocationID,trip_distance`, but the model actually trains on `trip_route` (pickup and dropoff combined) plus `trip_distance`. The tag is slightly misleading.

---

## Lesson 5: CI Review and Troubleshooting

[PR #1](https://github.com/Jassalam/mle-github-actions-cicd/pull/1) was reviewed in the order the lesson recommends.

### 1. Conversation

Three CML comments, one per successful run, with test RMSE 5.0074, 4.9639, and 4.9639. The failed run posted no comment, which matches the rule that a missing comment means the workflow failed before reporting.

### 2. Checks

Four runs: pass, pass, fail, pass. The latest commit `8a64cb1` is green. GitHub reports the PR as `MERGEABLE` with merge state `CLEAN`.

### 3. Files changed

Only `src/train.py` changed (+5 / -2). The workflow file is identical to `main` after the URL was restored. Every metric movement is explained by the diff:

- `n_estimators` 100 to 200 explains the RMSE drop.
- The row counts did not change, which is correct because the data and filters were not touched.

### 4. Commits

The failure and its fix were added as new commits, not by rewriting history.

### Merge decision

| Question | Answer |
| --- | --- |
| Did the workflow pass? | Yes, the latest run is green. |
| Did test RMSE increase? | No, it decreased by 0.04. |
| Did the row count change? | No. |
| Is this a docs-only PR that changed metrics? | No. |

Decision table result: **Workflow passes and metrics are stable, so continue with normal code review. The PR is merge-ready.**

Review notes that do not block the merge:

- The train/test gap widened from 0.32 to 0.49. Doubling the trees gave a small test improvement for twice the training time. Worth watching if `n_estimators` keeps increasing.
- The `features` tag does not match the features the model actually uses. This existed before the PR.

Recommended action: squash-merge PR #1, so the break and restore commits do not clutter `main`.

---

### Merge

```bash
gh pr merge 1 --squash --delete-branch
git checkout main
git pull
git fetch --prune
```

PR #1 was squash-merged. `main` is now at `17f5e12 CML experiment: extra metric and n_estimators change (#1)`, and the `cml-experiment` branch was deleted locally and on GitHub. The MLflow UI was stopped.

## Final State

- [x] Merge [PR #1](https://github.com/Jassalam/mle-github-actions-cicd/pull/1) in `mle-github-actions-cicd`.
- [x] Switch back to an up-to-date `main`.
- [x] Stop the MLflow UI.
- [ ] Optional: delete or ignore the stray `ruvector.db` file in the repository root. It is not part of this course.

## Files Created Locally (Not Committed)

| Path | Purpose |
| --- | --- |
| `data/green_tripdata_2025-01.parquet` | Raw data, tracked by DVC |
| `mlflow.db` | Local MLflow tracking and model registry |
| `mlruns/` | MLflow run artifacts |
| `metrics.txt` | CML report from the last local run |
| `../dvc-local-remote/` | Local DVC remote storage |
