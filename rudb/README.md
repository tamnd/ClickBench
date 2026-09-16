# RuDB

[RuDB](https://github.com/tamnd/rudb) is an embedded analytical database compatible with DuckDB.

This entry intentionally follows the DuckDB setup in this repository. It downloads the same
single Parquet file, creates the same `hits` schema, converts the four integer-encoded date and
timestamp columns with the same expressions, loads the rows once into the engine's native
single-file columnar format, and runs the same 43 queries through the CLI. RuDB is embedded, so
the concurrent-QPS test is disabled for the same reason as DuckDB's.

`install` pins the source archive and checksum for RuDB 0.3.22. That release is built from source
because the native storage format is newer than the last release that published Linux binaries.
The result is installed as `/usr/local/bin/rudb`.

Run the standard full ClickBench benchmark on Ubuntu 24.04 or newer with:

```sh
cd rudb
./benchmark.sh
```

The [development benchmark report](BENCHMARK.md) records the 1K through 10M validation ladder
against DuckDB and ClickHouse. Those sampled runs validate the setup; they are not official
ClickBench scores.
