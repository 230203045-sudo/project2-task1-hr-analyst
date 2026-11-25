# Project 2, Task 1: HR case, Data Analyst (Almaty)

500 applicants, 2 hires. This repo explains how the 2 were picked, in data analytics terms.

**Answer: `C1131` and `C1133`** (reserve: `C1097`).

| ID | Company test | Interview | Experience | English | Skills | Salary ask | Weighted score |
|---|---|---|---|---|---|---|---|
| C1131 | 97 | 5/5 | 5 yrs | C1 | SQL, Python, Tableau, statistics, A/B tests | 550k KZT | 95.4 |
| C1133 | 94 | 4/5 | 3 yrs | IELTS 7.0 (C1) | SQL, Python, Power BI, statistics | about 550k KZT | 80.7 |
| C1097 (reserve) | 93 | 5/5 | 8 yrs | C2 | SQL, Excel, ML | "by agreement" | 75.4 |

## Files (6)

| File | What it is |
|---|---|
| `README.md` | this page: result, method, data dictionary |
| `data_raw.csv` | the HR export exactly as received (two exports glued together, messy on purpose) |
| `data_clean.csv` | 488 unique candidates, 33 columns, no names / phones / e-mails / CV links |
| `analysis.ipynb` | all code: parsing, cleaning, funnel, scoring, robustness, fairness audit (run top to bottom; needs `pandas`, `numpy`, `scipy`) |
| `data_note.pdf` | 1 page: source, contents, what is missing, what it must not be used for |
| `memo.pdf` | 1 page: picks, weights, funnel, blind spots |

## Method in four lines

1. **Weights (add to 100%):** company test 30%, interview 25%, nice-to-have skills (Python, BI tool, statistics / A/B tests, English C1+) 30%, experience 15%.
2. **Funnel:** 500 to 488 (duplicates) to 123 (no SQL) to 83 (under 2 years) to 42 (English below B2) to 26 (salary above 700k) to 10 (test below 70) to 5 (interview below 4) to 2 (weighted score, then judgment).
3. **Robustness:** 5,000 random weight sets: C1131 is in the top 2 every time; the second place swings between C1097 (62%) and C1133 (38%). C1133 chosen because it meets all must-haves and all three analytics nice-to-haves with a verified salary; C1097 has none of the nice-to-haves and an unverified salary. Counting ClickHouse as SQL, counting only literal "SQL", or changing the USD rate (400 / 500 / 600) does not change the pair.
4. **Blind spot:** 13 people were stopped only by missing or unclear data (empty test, "Intermediate" English). They are on a re-check list in the notebook, not rejected. In the best case 3 of them could beat the No. 2; the main one is `C1117` (test score missing).

## Cleaning decisions that protect good candidates

* The file is two exports with different delimiters and column order, page-break lines, a footer and multi-line comments: parsed record by record.
* One ID shared by two different people (`C1432`): both kept. Same person re-applying under a new ID with a new test score: the **latest** application is kept.
* `PostgreSQL` and `MySQL` count as SQL. A skill written only in the HR comment counts too.
* Broken text encoding (UTF-8 read as cp1251) repaired instead of dropping the row.
* Impossible values (negative experience, salary of 0 or 35,000,000, test score of 135, application date after the export date) become missing, not deleted rows.
* Never used: gender, age, city (relocation is paid), university, CV gaps, `hired_before`, HR flags such as "recommended by the director".

## Bias check

Applicants are only those who applied from 1 Sep to 21 Oct 2026 through these channels. About 13% have no test score and 16% no interview score, so complete records are favoured. All 5 candidates who passed every must-have are women, but gender was not used; the company-test gate passes more women than men (35% vs 24%, Fisher p = 0.008), the other gates do not differ, and CV gaps do not change the chance to pass (p = 0.66). Salary in USD is converted at 500 KZT/USD (my assumption).

## Data dictionary (`data_clean.csv`)

Missing-value code for every column: empty cell.

| Column | Meaning | Type | Unit / allowed values |
|---|---|---|---|
| candidate_id | candidate ID (`-B` suffix for a second person with the same ID) | text | C1000 to C1499 |
| gender | normalised gender, audit only | category | M, F, unknown |
| age | age at export date 29.10.2026, audit only | number | years, 21 to 46 |
| exp_years | work experience | number | years, 0 to 12 |
| exp_note | parsing status of experience | category | ok, missing, impossible, unparsed |
| skill_sql_literal | "SQL", "sql", "SQl", PostgreSQL or MySQL in the skills list | boolean | True / False |
| skill_clickhouse | ClickHouse in the skills list | boolean | True / False |
| skill_python | Python, "Пайтон", py or pandas | boolean | True / False |
| skill_bi | Power BI, Tableau or Looker | boolean | True / False |
| skill_stats_ab | statistics, stats, A/B tests, SPSS or R | boolean | True / False |
| sql_from_comment | SQL found only in the HR comment ("skills: SQL, Python") | boolean | True / False |
| skill_sql | must-have SQL: literal or from comment | boolean | True / False |
| english_cefr | English level mapped to CEFR; `B1*` = "Intermediate" (ambiguous B1/B2) | category | A2, B1, B1*, B2, C1, C2 |
| english_rank | numeric rank of English level | number | 2 to 6 (B2 = 4) |
| test_score | company test | number | points, 0 to 100 |
| test_note | parsing status of the test score | category | ok, missing, impossible |
| interview_score | interview score; words like "good" mapped to 1 to 5 | number | 1 to 5 |
| interview_note | parsing status of the interview | category | ok, qualitative, missing, impossible |
| salary_min_kzt | lower bound of salary expectation | number | KZT per month, 100,000 to 5,000,000 |
| salary_max_kzt | upper bound (same as min unless a range) | number | KZT per month |
| salary_note | status of salary | category | ok, negotiable, impossible |
| salary_in_usd | salary was written in USD (converted at 500) | boolean | True / False |
| applied_date | application date | date | 2026-09-01 to 2026-10-21 |
| applied_note | date parsing status | category | ok, resolved by window, ambiguous, impossible |
| cv_gap_reason | reason for a gap in the CV, audit only | text | empty = no gap |
| hired_before | worked here before, audit only | number | 0 / 1 |
| source_channel | where the person applied from | category | 8 channels |
| hr_comment | free text from HR (CV links removed) | text | - |
| export_block | which of the two exports the row came from | number | 1 or 2 |
| reapplied | person applied more than once, latest kept | boolean | True / False |
| passes_all_must_haves | passed all 6 must-have rules | boolean | True / False |
| weighted_score | score from the 4 weights, only for the 5 who passed | number | 0 to 100 |
| final_pick | one of the 2 chosen | boolean | True / False |
