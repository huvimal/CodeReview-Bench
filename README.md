# CodeReview Bench

Compare a Ruff baseline with a Ruff + LLM reviewer on labeled Python changes.
Each finding has a line number in the new file, a category, and an explanation.
The reviewer reads code as text and does not execute the code under review.

## Run on Windows

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe review.py bench --mode ruff
.\.venv\Scripts\python.exe review.py bench --mode replay
```

`ruff` runs the installed linter with F401 and F821. `replay` adds prepared
LLM-like findings to real Ruff output. It includes a deliberately false alarm,
a duplicate and a finding on an unchanged line. Replay checks the report flow.
It does not measure model performance. Reports are written under `reports/`.

## Compare with Groq

Use public or synthetic code. Live mode sends the diff and numbered new file
to Groq. It sends eight requests when running the bundled set.

```powershell
$reviewKey = Read-Host "Groq API key" -AsSecureString
$env:GROQ_API_KEY = [System.Net.NetworkCredential]::new('', $reviewKey).Password
.\.venv\Scripts\python.exe review.py bench --mode groq --case wrong-total
.\.venv\Scripts\python.exe review.py bench --mode groq
Remove-Item Env:GROQ_API_KEY
```

The default model is `openai/gpt-oss-120b`. Use `--model` for another
JSON-capable model available to your Groq project. If one case hits a provider
limit, rerun it with `--case ID`. A single-case run writes a separate report.

## Review your own file pair

```powershell
.\.venv\Scripts\python.exe review.py review --before private_data/before.py --after private_data/after.py --mode ruff
```

Save two versions of the same Python file as UTF-8 before running this command.
For model findings as well, use `--mode groq` with an API key set.

## Read the report

The tool compares file snapshots, runs Ruff, and keeps findings on added or
replaced lines. It then accepts only model findings on those lines and
deduplicates by line and category. For the benchmark, a prediction is counted
as correct if both its line and category match a gold label.

- **Precision** measures the fraction of predicted findings that matched gold.
- **Recall** measures the fraction of labeled defects that were detected.
- **FP** counts predictions with no matching gold label, including warnings
  on the clean changes. **FN** counts labeled defects that were missed.

For live runs, compare `paired_ruff_summary` and `augmented_summary` on the
same successful cases. Provider failures are shown separately. A case that
failed to call Groq is excluded from both paired summaries.

## Live pilot results

Both configurations reviewed the same eight synthetic Python changes, which
included five labeled defects. A finding matched when its new-file line and
category agreed with the reference label.

| Configuration | Matched defects | False positives | Missed defects | Precision | Recall |
| --- | ---: | ---: | ---: | ---: | ---: |
| Ruff (F401, F821) | 2 | 0 | 3 | 100% | 40% |
| Ruff + GPT-OSS 120B via Groq | 5 | 0 | 0 | 100% | 100% |

The model added three findings: tax subtracted instead of added, an exclusive
shipping threshold where the boundary should be inclusive, and a missing
`None` guard. These results describe one run on the bundled pilot set.

## Scope and portfolio

The starter set has eight synthetic changes. One file pair is reviewed at a
time, with 12,000 characters per snapshot. The tool only anchors findings to
new-file lines, so deletion-only changes and defects requiring multi-file
context are outside this version. The restricted Ruff rule set is a baseline.
An exact match score does not evaluate the usefulness of the explanation.

For a portfolio report, show one correct logic finding from a live run and
one missed or incorrect finding if observed. Record the model, the size of
the set, and the two paired scores. Keep replay results separate.
