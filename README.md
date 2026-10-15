# IPL Data Analysis

This project analyzes Indian Premier League ball-by-ball data on Databricks. CSV files live in Amazon S3, Spark reads them through a Unity Catalog external location, and the notebook turns those tables into player, bowling, venue, dismissal, and toss results.

It is a learning record of that full path: storage setup, typed schemas, DataFrame transformations, Spark SQL, and charts in Matplotlib and Seaborn.

## What the analysis asks

- Who scored the most runs in each season?
- Which bowlers conceded the fewest runs per ball in the powerplay?
- Does winning the toss line up with winning the match, and which teams convert the toss most often?
- Which venues produce the highest combined scores?
- How are batters usually dismissed?
- Which players score the most runs per ball in matches their team wins?

## Data

Each file is one grain of the IPL dataset.

| File | One row represents |
| --- | --- |
| `Ball_By_Ball.csv` | A single delivery |
| `Match.csv` | A match |
| `Player.csv` | A player |
| `Player_match.csv` | A player in a specific match |
| `Team.csv` | A team |

`Ball_By_Ball.csv` is read from `s3://ipl-analyst-data-project/`. The other four files are read from `s3://ipl-data-analysis-project/`. Point both paths at the bucket you created in the setup guide before re-running the notebook.

## Stack

- Databricks and Unity Catalog
- Apache Spark (PySpark DataFrames and Spark SQL)
- Amazon S3
- pandas, Matplotlib, and Seaborn

## Guides

Read them in this order.

1. [S3 and Databricks setup](docs/s3-databricks-setup.md) connects the bucket to Unity Catalog.
2. [Loading the tables](docs/loading-data.md) starts a Spark session and applies a schema to each file.
3. [Transformations](docs/transformations.md) filters deliveries, adds running totals, and cleans players and matches.
4. [SQL analysis](docs/sql-analysis.md) registers temporary views and runs the questions above.
5. [Visualizations](docs/visualizations.md) turns those results into bar charts.

Screenshots from the Databricks notebook are stored as PNG files in [`docs/images`](docs/images).

The original cell-by-cell notebook export is still in [`commands.txt`](commands.txt). The guides above are the formatted version of that work, with notes on results that are easy to misread.
