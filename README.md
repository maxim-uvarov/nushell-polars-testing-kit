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
    polars open data/nz.csv
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
1sec 561ms ± 15.61%
```

Let's try `polars open --lazy`

```nushell
bench -n 10 --pretty {
    polars open data/nz.csv --lazy
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
42ms 160µs ± 5.98%
```

However, [dply 0.3.2](https://github.com/vincev/dply-rs/commit/13f5bab1132d39569ee183b22b2e6e9a679235f9) is built on polars 0.40.0 too but shows no problems at all:

```nushell
> dply --version
dply 0.3.2

> bench --rounds 10 --pretty {dply -c 'csv("data/nz.csv") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
94ms 150µs ± 10.62%
```

Let's save this file to parquet and measure this process time:

```nu
> timeit {polars open --lazy data/nz.csv | polars to-parquet data/nz.parquet}
1sec 677ms 431µs 83ns
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
39ms 700µs ± 12.9%
```

```nu
> timeit {polars open data/nz.parquet --lazy | polars to-jsonl data/nz.jsonl}
1sec 58ms 119µs
```

```nu
bench -n 10 --pretty {
    polars open data/nz.jsonl --lazy
    | polars group-by year
    | polars agg (polars col geo_count | polars sum)
    | polars collect
}
```

Output:

```
3sec 863ms ± 0.38%
```

while `dply` works with json quite fast.

```nu
> bench --rounds 10 --pretty {dply -c 'json("data/nz.jsonl") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
346ms 500µs ± 4.48%
```

```nu
> version
╭────────────────────┬─────────────────────────────────────────────────╮
│ version            │ 0.94.2                                          │
│ major              │ 0                                               │
│ minor              │ 94                                              │
│ patch              │ 2                                               │
│ branch             │ main                                            │
│ commit_hash        │ be8c1dc0066cd1034a6b110a622f47b516bfe029        │
│ build_os           │ macos-aarch64                                   │
│ build_target       │ aarch64-apple-darwin                            │
│ rust_version       │ rustc 1.77.2 (25ef9e3d8 2024-04-09)             │
│ rust_channel       │ 1.77.2-aarch64-apple-darwin                     │
│ cargo_version      │ cargo 1.77.2 (e52e36006 2024-03-26)             │
│ build_time         │ 2024-06-04 08:20:37 +00:00                      │
│ build_rust_channel │ release                                         │
│ allocator          │ mimalloc                                        │
│ features           │ default, sqlite, system-clipboard, trash, which │
│ installed_plugins  │ explore, polars, regex                          │
╰────────────────────┴─────────────────────────────────────────────────╯
```
