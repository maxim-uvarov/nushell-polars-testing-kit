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
# => 189ms 100µs ± 1.99%
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
# => 173ms 200µs ± 5.26%
```

However, [dply 0.3.2](https://github.com/vincev/dply-rs/commit/13f5bab1132d39569ee183b22b2e6e9a679235f9) is built on polars 0.40.0 too but shows no problems at all:

```nushell
> dply --version
# => dply 0.3.5

> bench --rounds 10 --pretty {dply -c 'csv("data/nz.csv") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
# => 496µs ± 81.73%
```

Let's save this file to parquet and measure this process time:

```nu
> timeit {polars open data/nz.csv | polars save data/nz.parquet}
# => 936ms 168µs 42ns
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
# => 31ms 540µs ± 8.26%
```

```nu
> timeit {polars open data/nz.parquet | polars save data/nz.jsonl}
# => 836ms 274µs 750ns
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
# => 261ms 500µs ± 4.53%
```

while `dply` works with json quite fast.

```nu
> bench --rounds 10 --pretty {dply -c 'json("data/nz.jsonl") | group_by(year) | summarize(count = n(), sum = sum(geo_count)) | show()' | null}
# => 688µs 900ns ± 84.53%
```

```nu
> version
# => ╭────────────────────┬─────────────────────────────────────╮
# => │ version            │ 0.103.0                             │
# => │ major              │ 0                                   │
# => │ minor              │ 103                                 │
# => │ patch              │ 0                                   │
# => │ branch             │                                     │
# => │ commit_hash        │                                     │
# => │ build_os           │ macos-aarch64                       │
# => │ build_target       │ aarch64-apple-darwin                │
# => │ rust_version       │ rustc 1.83.0 (90b35a623 2024-11-26) │
# => │ rust_channel       │ stable-aarch64-apple-darwin         │
# => │ cargo_version      │ cargo 1.83.0 (5ffbef321 2024-10-29) │
# => │ build_time         │ 2025-03-19 07:32:54 -03:00          │
# => │ build_rust_channel │ release                             │
# => │ allocator          │ standard                            │
# => │ features           │ default, sqlite, trash              │
# => │ installed_plugins  │ explore 0.100.0, polars 0.103.0     │
# => ╰────────────────────┴─────────────────────────────────────╯
```
