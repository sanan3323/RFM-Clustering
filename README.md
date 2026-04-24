# Customer Segmentation: RFM, K-Means, CLV

Segmentation of 4,338 retail customers using RFM features, k-means clustering,
and BG/NBD plus Gamma-Gamma for expected customer lifetime value. The notebook
goes from raw transactions to four named segments with dollar values and
recommended actions.

Data: UCI Online Retail, one year of transactions from a UK online gift
retailer (Dec 2010 to Dec 2011).

## Results

| Segment | Customers | Share | Avg CLV (12m) | Total CLV (12m) |
|---|---|---|---|---|
| Champions | 720 | 16.6% | $7,488 | $5.39M |
| At Risk / Cannot Lose | 1,174 | 27.1% | $2,006 | $2.36M |
| New Customers | 856 | 19.7% | $1,181 | $1.01M |
| Hibernating | 1,588 | 36.6% | $431 | $684K |

Champions are 17% of the customer base and carry roughly 58% of expected
12-month revenue. 44 of the 720 Champions are whales (top 1% of Monetary,
threshold $19,881), and they account for a large share of the Champions total
on their own.

Practical read: most of the marketing budget should be split between
protecting Champions and reactivating At Risk before they turn into
Hibernating. Hibernating is a third of the customer list and only 7% of
forecast revenue, so it earns low-cost email treatment at most.

## Method notes

- Snapshot date is pinned to the day after the last transaction in the data,
  not `datetime.now()`. Analysis should give the same answer next month.
- Monetary is **not** IQR-filtered. The top 1% (44 customers) is flagged as
  `is_whale` but stays in the clustering. Standard outlier removal on
  Monetary deletes most of the Champions segment.
- R, F, M are log1p-transformed and standard-scaled before k-means. K-means on
  raw RFM is dominated by the Monetary tail and produces a "rich vs not rich"
  split that isn't useful.
- `k=4` chosen on business grounds. Silhouette peaks at k=2 (0.43) and
  Calinski-Harabasz is also maximal at k=2, which is the usual shape when the
  data has one dominant active-vs-lapsed split. k=2 collapses the segmentation
  into something marketing already knows. k=4 is the smallest k that
  separates Champions from Loyal-like behavior and At Risk from Hibernating.
  Silhouette at k=4 is 0.34, which still indicates real structure.
- Seed stability across 10 random seeds: mean ARI 0.986, min 0.972. The
  partition is not seed-dependent.
- Classical 5-quintile RFM run alongside and cross-tabbed with k-means. ARI
  0.326, NMI 0.486. Moderate agreement, which is expected: classical RFM has
  9+ segments against 4 k-means clusters, so a 1-to-1 mapping isn't possible.
  The k-means Champions cluster concentrates in classical Champions and
  Loyal; the Hibernating cluster concentrates in classical Lost and
  Hibernating. That's the check.
- Cluster names come from centroid R/F/M ranks, not hand-labeling. Same rule
  applied to retrained data produces the same names for equivalent behavior.
- CLV via `lifetimes` (BG/NBD for purchase frequency, Gamma-Gamma for
  transaction value). Frequency-Monetary correlation is 0.016 on this data,
  so the Gamma-Gamma independence assumption holds cleanly. 2,790 of 4,338
  customers have `frequency >= 1` and make it into the CLV fit; the 1,548
  one-time buyers get backfilled with segment median CLV at the end.

## Repo layout

```
customer-segmentation-rfm/
├── README.md
├── requirements.txt
├── notebooks/
│   └── customer-segmentation-RFM.ipynb
└── data/
    └── Online_Retail.csv 
```

## Run it

```bash
pip install -r requirements.txt
# download Online_Retail.xlsx from https://archive.ics.uci.edu/dataset/352/online+retail,
# convert to CSV, and drop it at data/raw/Online_Retail.csv
jupyter lab notebooks/customer-segmentation-RFM.ipynb
```

The notebook reads from `~/Online_Retail.csv` by default; change the path in
section 1 if your file lives elsewhere.

## Dependencies

```
numpy
pandas
matplotlib
seaborn
scikit-learn
lifetimes
jupyterlab
```

Tested on Python 3.11.

## Limitations

- K-means assumes spherical, equal-variance clusters. After log+scale that's
  defensible, but a GMM or hierarchical clustering run would be worth adding
  as a robustness check on a real engagement. The seed-stability result makes
  me comfortable skipping it for this writeup.
- BG/NBD is non-contractual and continuous-time. It fits retail. Don't use it
  on a subscription business.
- The `lifetimes` package is in maintenance mode. For production work,
  `pymc-marketing` is the actively-developed successor and implements the
  same models with better diagnostics.
- RFM ignores product mix, margin, and acquisition channel. A second layer of
  clustering inside each segment on category behavior would probably split
  Champions into useful sub-groups (wholesale bulk buyers vs retail gift
  buyers, most likely).
- Online Retail mixes wholesale and retail customers in a single file. A
  serious analysis would split those first.

## Source

Online Retail dataset, UCI Machine Learning Repository:
https://archive.ics.uci.edu/dataset/352/online+retail

Roughly 542k transactions raw, 398k after cleaning.
