# Out-of-core NYC Cab Data Handling
An out-of-core data pipeline that analyzes ~100 million NYC taxi trips without loading it all into memory.

## Problem
Yellow-taxi data alone from the publicly available NYC Taxi & Limousine Commission (TLC) trip dataset is ~1.5 billion rows and ~50GB, and just a two years subset of this is ~100 million rows and ~12.9 GB, which still surpasses Google Colab's 12.7 GB of RAM. Since standard data analysis requires loading the entire dataset into memory, performing standard analysis on the the yellow-taxi trip data isn't possible.  

## Approach
This project handles that data anyway using **out-of-core** techniques: processing data on the disk and pulling through only the pieces that are needed at any given moment, so the full dataset never has to fit in memory at once.

There are four main layers behind this pipeline, each one attacking the problem differently:
| Layer | Role | What it does |
|-------|------|--------------|
| **Parquet** | efficient *format* | reads on the necessary columns |
| **DuckDB** | out-of-core *engine* | streams the data as needed, doesn't load it all |
| **Partitioning** | efficient *layout* | excludes the data we don't need |
| **DVC** | *governance* | versions the data reproducibly |

## Pipeline stages
 
1. **Acquire** — Download 24 monthly Parquet files (yellow taxi, 2023–2024) from the TLC. The year range is chosen deliberately for a consistent schema across all files; mismatched schemas are a common cause of multi-file processing failures.
2. **Establish the constraint** — Measure one month's in-memory footprint in pandas and extrapolate to show the full 24-month load would exceed Colab's RAM.
3. **Columnar reads (Parquet)** — Parquet stores data by column rather than by row, so the engine reads only the requested columns off disk (*column pushdown*) and uses per-chunk statistics to skip rows that can't match a filter (*predicate pushdown*). Demonstrated by reading two columns at a fraction of the full-file memory cost.
4. **Streaming queries (DuckDB)** — DuckDB reads the Parquet files directly and executes aggregations in a streaming fashion, pulling data through in batches and spilling to disk as needed. A single SQL query over a `*.parquet` glob aggregates all ~100M rows while RAM usage stays essentially flat — the core demonstration of out-of-core processing.
5. **Partitioning** — The data is rewritten into a Hive-partitioned layout (`year=/month=/`). Filtered queries then read only the relevant partitions instead of scanning everything (*partition pruning*), the standard organization for data lakes.
6. **Versioning (DVC)** — Git is built for code, not gigabytes of data. DVC replaces the data folder with a small pointer file (hash + metadata) that Git tracks, while the actual bytes live in a separate cache/remote. This ties each version of the data to each version of the code without bloating the repository.
7. **Feature extraction (capstone)** — The same streaming engine distills the raw trips into a compact, model-ready `features.parquet` (selected fields, computed tip-percentage, filtered invalid rows), reframing the pipeline as ML data preparation: turning a huge, messy raw dataset into a clean training table.
## Results / what this demonstrates
 
- Aggregated ~100M rows on a machine with ~12.7 GB RAM, with memory usage staying flat during execution.
- Column-pruned reads using a small fraction of full-file memory.
- Partitioned layout enabling filtered queries that scan only relevant partitions.
- Dataset versioned via a lightweight DVC pointer rather than committing raw data.
*(Replace with your actual query outputs — e.g. total trips, average fare, tip-by-hour table — and a note on the RAM stayed flat during the full-dataset aggregation.)*
 
## Reproduce
 
Runs in **Google Colab** on a standard **CPU runtime** (no GPU needed).
 
1. Open the notebook in Colab:
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1drOsq61kXwmoVSgrEo7owOFy1hnSzjVG?usp=sharing)
2. Set the runtime to CPU: **Runtime → Change runtime type → CPU**.
3. Run all cells top to bottom. 
 
## Tech stack
 
Python · DuckDB · Apache Parquet (PyArrow) · pandas · DVC · Google Colab
 
## Next steps
 
- Push the DVC-tracked data to a real remote (S3 / GCS / Google Drive) for full reproducibility.
- Add a lightweight model trained on `features.parquet` (e.g. tip-percentage regression) to close the loop from raw data to trained model.
- Benchmark query time before vs. after partitioning to quantify the partition-pruning speedup.
