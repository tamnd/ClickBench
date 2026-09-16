# RuDB ClickBench development ladder

These are development results over strided samples of the real ClickBench `hits` dataset. They
are not official ClickBench scores and must not be compared with full 99,997,497-row results on
the public board.

## Result

Every engine completed all 43 queries at every size. RuDB and DuckDB were loaded into their native
single-file formats using the same schema and projection. The sum below is the sum of the 43
per-query medians from five hot runs after one first run, using each engine's own query timer.

| Rows | RuDB 0.3.22 | DuckDB 2.0.0-dev84237 | ClickHouse 26.9.1.1162 | RuDB / DuckDB | RuDB / ClickHouse |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 1,000 | 0.014503 s | 0.122000 s | 0.090000 s | 0.12x | 0.16x |
| 10,000 | 0.049601 s | 0.146000 s | 0.102000 s | 0.34x | 0.49x |
| 99,998 | 0.162220 s | 0.332000 s | 0.194000 s | 0.49x | 0.84x |
| 999,975 | 0.607559 s | 0.547000 s | 0.395000 s | 1.11x | 1.54x |
| 9,999,750 | 4.769067 s | 3.179000 s | 1.552000 s | 1.50x | 3.07x |

The ratio is elapsed RuDB query time divided by the comparison engine, so values below one favor
RuDB. At small sizes process and query startup dominate. At 10M rows, RuDB is 1.50x DuckDB and
3.07x ClickHouse by the engines' reported query clocks.

## Load and storage

| Rows | RuDB load | RuDB bytes | DuckDB load | DuckDB bytes | ClickHouse load | ClickHouse size |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1,000 | 0.009677 s | 590,212 | 0.045498 s | 1,060,864 | 0.171392 s | 356.92 KiB |
| 10,000 | 0.034828 s | 5,783,938 | 0.105939 s | 3,944,448 | 0.207809 s | 2.61 MiB |
| 99,998 | 0.264164 s | 56,604,295 | 0.638857 s | 30,158,848 | 0.529772 s | 23.93 MiB |
| 999,975 | 2.576274 s | 561,185,011 | 1.985030 s | 525,611,008 | 2.885 s | 143.66 MiB |
| 9,999,750 | 24.310180 s | 5,510,815,258 | 8.280287 s | 3,991,678,976 | 25.402 s | 1.22 GiB |

Load time is wall time. The ClickHouse size is the active MergeTree parts reported by
`system.parts`; the other sizes are database file lengths.

## Correctness

The native audit ran the original 43 DuckDB queries through DuckDB native, RuDB native, DuckDB
Parquet, and RuDB Parquet. Across all 215 size/query pairs, the first-run answers were either an
exact match, the same rows in a different order, or a different tied `LIMIT` selection that matched
after an untimed deterministic tie-breaker was added. No deterministic retest remained unresolved,
and no rewritten query replaced a timed query.

## Method

The run used `gamingpc-wsl`: Linux 6.18.33.2 under WSL2, an Intel Core i9-13900K, 32 logical CPUs,
31.34 GiB RAM, and ext4 storage. The native audit started at a one-minute load average of 0.09.
The retained dataset row counts were 1,000, 10,000, 99,998, 999,975, and 9,999,750, sampled at
strides of 99,998, 10,000, 1,000, 100, and 10 respectively.

The native measurements used [`tamnd/rudb-bench` commit
`1340195`](https://github.com/tamnd/rudb-bench/commit/1340195). Every repetition started a fresh
process. The first run was not disk-cold because the OS page cache was not dropped. Query time came
from the CLI timer and exact-child wall time, CPU, and peak RSS came from Linux `wait4`. The commands
were:

```sh
python3 scripts/clickbench-audit.py \
  --duckdb /usr/local/bin/duckdb \
  --rudb /usr/local/bin/rudb \
  --data data \
  --output run-20260916 \
  --sizes 1k 10k 100k 1m 10m \
  --hot 5 --timeout 600
python3 scripts/verify-clickbench-audit.py run-20260916
```

ClickHouse was measured by the same harness's server path, once per size:

```sh
rudb-bench run clickbench --rows SIZE \
  --engines clickhouse-server --runs 5 --report
```

ClickHouse stayed up across its suite, so its internal caches were warm; DuckDB and RuDB used a
fresh embedded process for every repetition. The query-time column follows the public ClickBench
convention, but ClickHouse's process and cache lifecycle differs from the embedded engines and the
ratios must be read with that limitation. The 1K through 1M ClickHouse runs started below a 3.0
one-minute load average; the 10M run started at 3.04 on the 32-thread host.
