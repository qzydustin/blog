+++
title = "Converting .sas7bdat to CSV Without SAS"
date = 2023-08-27T15:16:21+00:00

[taxonomies]
categories = ["CS"]
tags = ["sas", "python", "pyreadstat"]
+++

`.sas7bdat` is the binary format SAS uses to store data sets. SAS is commercial statistical software with a licensing cost that prices out most individual users. The format is proprietary and undocumented by SAS, so every reader outside it, including pyreadstat, is the product of reverse engineering.

A single `.sas7bdat` file holds the table itself, the column metadata, and enough type information to reconstruct rows. What it does not hold is the display layer. SAS stores human-readable value labels and formats in a separate catalog file, `.sas7bcat`. Converting to CSV throws that layer away, because CSV has no place to put it.

<!--more-->

## Converting with pyreadstat

pyreadstat is a Python binding around ReadStat, a reverse-engineered C parser by Evan Miller. It reads the data set and reattaches value labels from the catalog.

```python
import pyreadstat

df, meta = pyreadstat.read_sas7bdat(
    'input.sas7bdat',
    catalog_file='formats.sas7bcat',
    apply_value_formats=True,
)
df.to_csv('output.csv', index=False)
```

`read_sas7bdat` returns the DataFrame and a `meta` object. With `apply_value_formats=True`, a column coded `1=Yes, 2=No` arrives in the CSV as `Yes` and `No` rather than `1` and `2`. `meta` holds the variable labels and format definitions, which you can inspect or write to a sidecar file to keep alongside the CSV.

pyreadstat also decodes SAS dates correctly. SAS numbers days from 1960-01-01, not the 1970 Unix epoch. ReadStat applies that offset internally, so date columns arrive as proper datetimes.

## Encoding

Older `.sas7bdat` files from Windows SAS are often `wlatin1`, not UTF-8. If you see mojibake in character columns, pass the encoding explicitly.

```python
df, meta = pyreadstat.read_sas7bdat(
    'input.sas7bdat', encoding='latin-1',
    catalog_file='formats.sas7bcat', apply_value_formats=True,
)
```

Guessing wrong leaves the data intact but garbles accented characters. There is no penalty for trying a different encoding.

## Beyond Python

R users reach the same result with `haven`, which also wraps ReadStat, so the two agree on output. For batch conversion of many files, `sas7bdat-converter` wraps a reader behind a command-line interface.

## What the conversion loses

CSV cannot represent value labels, column formats, or SAS missing-value sentinels (`.`, `.A` through `.Z`). A conversion is lossy by design, even with pyreadstat applying labels. If the missing-value distinctions matter for your analysis, keep the original `.sas7bdat` and the catalog, and treat the CSV as a transport format rather than an archive.
