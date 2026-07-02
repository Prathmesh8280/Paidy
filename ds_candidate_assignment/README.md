# Spendy Loyalty Analysis — Reviewer Guide

This submission develops a business-driven definition of customer loyalty for Spendy and trains a machine learning model to identify customers likely to become loyal based on their first 60 days of activity.

## What is in this submission

| File | What it is |
|---|---|
| `spendy_loyalty_analysis.ipynb` | Full analysis notebook with all code, charts, and written commentary |
| `Spendy_Loyalty_Analysis_Business.html` | Business-friendly version — charts and findings only, no code |

---

## Which file should I look at?

**If you want to see the findings, charts, and conclusions without wading through code** — open `Spendy_Loyalty_Analysis_Business.html` directly in any browser. No setup or installation required.

**If you want to review the code, methodology, and full technical detail** — open `spendy_loyalty_analysis.ipynb` in Jupyter. Instructions below.

---

## How to run the notebook

### 1. Add the data file

The notebook requires the parquet file provided separately. Place it in the same folder as the notebook before running:

```
ds_candidate_assignment/
├── spendy_loyalty_analysis.ipynb
├── datascientist_candidate_assignment.parquet   <-- place here
└── README.md
```

The notebook reads it with a relative path, so no code changes are needed as long as both files are in the same folder and Jupyter is launched from that folder.

### 2. Install dependencies

```bash
pip install pandas numpy matplotlib seaborn scikit-learn pyarrow
```

### 3. Launch Jupyter and run

```bash
jupyter notebook spendy_loyalty_analysis.ipynb
```

Then in Jupyter: **Kernel > Restart & Run All**

The notebook runs top to bottom with no manual steps. Expected runtime is around 2 to 3 minutes, mostly due to the monthly loyalty tracking loop in Section 3.
