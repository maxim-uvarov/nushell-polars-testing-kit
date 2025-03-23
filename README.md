# nushell-polars-testing-kit

I assume that there are going to be several testing datasets. That is why we have the `archive` and `data` directories in this repo.

As of now, in the `archives` folder, there is the same file that can be downloaded from [here](https://www.stats.govt.nz/assets/Uploads/New-Zealand-business-demography-statistics/New-Zealand-business-demography-statistics-At-February-2020/Download-data/Geographic-units-by-industry-and-statistical-area-2000-2020-descending-order-CSV.zip):

```nu no-run
# unpack the file and save it to data folder with the name `nz.csv`
> unzip -p archives/Geographic-units-by-industry-and-statistical-area-2000-2020-descending-order-CSV.zip | save data/nz.csv -r --progress -f
```

```nushell
use tools/bench.nu
bench -n 10 --pretty {
    polars open data/nz.csv --eager
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
# 0.95 => 137ms 900µs ± 27.43%
```

Let's try `polars open`

```nushell
bench -n 10 --pretty {
    polars open data/nz.csv
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
# 0.95 => 43ms 120µs ± 7.58%
```

However, [dply 0.3.2](https://github.com/vincev/dply-rs/commit/13f5bab1132d39569ee183b22b2e6e9a679235f9) is built on polars 0.40.0 too but shows no problems at all:

```nushell
> dply --version
dply 0.3.2

> bench --rounds 10 --pretty {dply -c 'csv("data/nz.csv") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
# 0.95 => 98ms 260µs ± 13.65%
```

Let's save this file to parquet and measure this process time:

```nu
> timeit {polars open data/nz.csv | polars save data/nz.parquet}
# 0.95 => 336ms 731µs 542ns
```

Let's measure the parquet opening time:

```nu
bench -n 10 --pretty {
    polars open data/nz.parquet
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
# 0.95 => 39ms 670µs ± 4.28%
```

```nu
> timeit {polars open data/nz.parquet | polars save data/nz.jsonl}
# 0.95 => 1sec 67ms 45µs 708ns
```

```nu
bench -n 10 --pretty {
    polars open data/nz.jsonl
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
# 0.95 => 322ms 500µs ± 3.41%
```

while `dply` works with json quite fast.

```nu
> bench --rounds 10 --pretty {dply -c 'json("data/nz.jsonl") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
# 0.95 => 328ms 600µs ± 1.01%
```

```nu
> version
╭────────────────────┬──────────────────────────────────────────╮
│ version            │ 0.95.1                                   │
│ major              │ 0                                        │
│ minor              │ 95                                       │
│ patch              │ 1                                        │
│ branch             │ polars_0_41                              │
│ commit_hash        │ 35db34a404f98bf53b59aec6dcf87af97ad36537 │
│ build_os           │ macos-aarch64                            │
│ build_target       │ aarch64-apple-darwin                     │
│ rust_version       │ rustc 1.77.2 (25ef9e3d8 2024-04-09)      │
│ rust_channel       │ 1.77.2-aarch64-apple-darwin              │
│ cargo_version      │ cargo 1.77.2 (e52e36006 2024-03-26)      │
│ build_time         │ 2024-06-28 10:05:26 +00:00               │
│ build_rust_channel │ release                                  │
│ allocator          │ mimalloc                                 │
│ features           │ default, sqlite, system-clipboard, trash │
│ installed_plugins  │ polars 0.95.1                            │
╰────────────────────┴──────────────────────────────────────────╯
```
