# Dataframes — A Zarr Convention

A [Zarr Convention][spec] for storing dataframes.

This spec is inspired by the on-disk layout that [AnnData][anndata-df] uses for its dataframes, re-expressed, hardened, and extended using the [Zarr Conventions][spec] discovery mechanism instead of AnnData's bespoke `encoding-type` / `encoding-version` attributes.

- **Convention name:** `df:`
- **UUID:** `1f1b36d2-3185-6da8-91f6-9a0823af2d33`
- **Schema:** [`schema.json`](./schema.json)
- **Applies to:** Zarr **groups** (`node_type: "group"`)
- **Zarr format:** 3 (the `zarr_conventions` attribute is a Zarr v3 mechanism)

## Convention Maturity

This is at maturity 0 - at the current stage, it is simply to gather feedback.
It will be published once it has reached some degree of acceptance/stability.

Terms that need clarification are in bold in the folliwing text.

## Why a convention?

Dataframes, [according to wikipedia][], is "A tabular data structure common to many data processing libraries" of which it then goes on to list a few. We all know dataframe libraries, like [polars][] or [pandas][] to reach for the lowest-hanging fruit. We are probably also familiar with the plethora of file formats that may back these data strucutres, like [parquet][] or the more new-fangled formats like [vortex][].

However, for users of scientific data stored in zarr, the prospect of using a different file format for what is generally not a very big operation (relative to the scale of proper array data) is disheartening (at least for a file-format-freak like myself).

And sometimes, you just want that collection of **1d-interpreted arrays** of the **same length** zipped up and packaged nicely in a data struture that we are all familiar with without that extra hassle.

I thus put forth a convention for dataframes in zarr, which is little more than a(n optionally indexed) collection of **1d-interpreted arrays** (more on the quotations below). 

## Layout

A conforming node is a Zarr group whose children are all **1d-interpreted arrays**.
All member names below are literal. Column *names* are never stored as member names; they live in `columns`.

- `columns` (required): the column names of the dataframe, in order.
- `index` (optional): the index of the dataframe.
- `column_1`, `column_2`, …, `column_n`: the data columns, numbered consecutively from 1 with no gaps.

`columns` has one entry per column, and the index counts as a column if present:
if `index` is present, `columns[0]` is the name of the index and `columns[i]` is the name of `column_i`;
otherwise, `columns[i-1]` is the name of `column_i`.
Thus `len(columns)` is `n + 1` with an index and `n` without one.

All children except `columns` MUST have the **same length**.

| Member     | Kind        | Requirement |
| ---------- | ----------- | ----------- |
| `columns`  | array/group | The columns names in column order. The list MUST include the index name as first entry if an index is present. |
| `index`    | array/group | The column that should serve as an index if an implementation supports it (optional) |
| `column_i` | array/group | The *i*th "data" column |

### Semantics

Here we clearly define what we mean by:

**1d-interpreted arrays**: This term is used to indicate that a group can be interpreted as a 1d array, such as the [`zarr-convention-enum`] or a future convention for genomics data that might have separate 1d arrays for chromosome/position/variant, but could be interpreted as a 1d array via a [custom extension array for genomics][]. A concrete example of this interpretability is anndata's [nullable strings][].
**same length**: This means that the separate **1d-interpreted arrays** should all be of the same lenght i.e., be interpreted as 1d arrays of the same length whose entries' positions within the array match that of another **1d-interpreted array**.

## Convention attributes

The group's `attributes` MUST contain a `zarr_conventions` entry (per the
[spec][spec]) identifying this convention, plus the namespaced property below.

| Attribute       | Type        | Req. | Meaning |
| --------------- | ----------- | ---- | ------- |
| `df:shape`      | `number[2]` | yes  | data frame dimensions \[rows, columns\], useful for data frames with no column or index |

`df:shape` MUST be consistent with the group's children:

- `df:shape[1]` MUST equal the length of `columns` (note that `index` counts as a column).
- `df:shape[0]` MUST equal the length of every child except `columns`.

Properties are namespaced with the `df:` prefix to avoid collisions
with other conventions present on the same node, as recommended by the spec.

### Minimal `zarr.json` (group)

```json
{
    "zarr_format": 3,
    "node_type": "group",
    "attributes": {
        "zarr_conventions": [
            {
                "uuid": "1f1b36d2-3185-6da8-91f6-9a0823af2d33",
                "schema_url": "https://raw.githubusercontent.com/DOES_NOT_EXIST/zarr-dataframe/refs/tags/v1/schema.json",
                "spec_url": "https://github.com/DOES_NOT_EXIST/zarr-dataframe/blob/v1/README.md",
                "name": "df:",
                "description": "A collection of 1d-interpreted arrays to be used as a dataframe"
            }
        ],
        "df:shape": [300, 10]
    }
}
```

> The `schema_url` / `spec_url` above point at a `v1` tag of a yet-to-be-named-organization-owned
> repository as a worked default. This will be udpated when we publish this
> convention for the first time; per the spec, `uuid` is the stable identifier and the URLs are
> resolution hints.

## Reading a dataframe

A convention-aware reader:

1. Reads the group's `attributes.zarr_conventions` array and matches an entry by
   `uuid == "1f1b36d2-3185-6da8-91f6-9a0823af2d33"` (falling back to
   `schema_url`, then `spec_url`, per the spec's identity precedence).
2. Reads the `columns` child array/group and asserts that it has length `df:shape[1]`.
3. Opens all child arrays/groups referred to by each entry of `columns`.
4. Constructs a 1d array out of each, and asserts their lengths each match `df:shape[0]` in their in-memory representation.
5. Assembles a dataframe with the column order in `columns` if applicable to the in-memory data structure.

A reader that does **not** know this convention still sees an ordinary group with one or more readable arrays and groups with easily inferable meanings so this convention is **safely ignorable**.

## Versioning

This is **v1**. Backwards-incompatible changes will bump the major version and
the `v1` segment of `schema_url`. The `uuid` is permanent and does not change
across versions. See the [spec's versioning guidance][spec].

## Relationship to `zarr-enum-convention`

Categorical columns (as potentially represented on-disk in [`zarr-convention-enum`]) are some of the most common features of modern dataframes.

Furthermore, this spec highlights the opportunity for a group to be interpreted as a 1d array if `codes` is 1d.  *We thus seek a way to formalize a group to indicate that it can be interpreted as 1d*.

## Relationship to AnnData

| AnnData                          | This convention                                            |
| -------------------------------- | ---------------------------------------------------------- |
| `encoding-type: "dataframe"`     | `zarr_conventions[].uuid == 1f1b36d2-...`                  |
| `encoding-version: "0.2.0"`      | the convention version (`v1`) via `schema_url` tag         |
| `column-order: <array>`          | `columns`                                                  |
| `_index`                         | `index`                                                    |

The byte-level layout of the arrays/group could be identical to
AnnData's , so existing data can be made conformant by renaming changing/adding json, and potentially hardening some of the **1d-interpreted array** specifications currently in anndata, like [nullable strings][].

## License

BSD-3-Clause. See [LICENSE](./LICENSE).

[spec]: https://github.com/zarr-conventions/zarr-conventions-spec
[anndata-df]: https://anndata.scverse.org/en/stable/fileformat-prose.html#dataframes
[according to wikipedia]: https://en.wikipedia.org/wiki/Dataframe
[polars]: https://pola.rs
[pandas]: https://pandas.pydata.org
[vortex]: https://vortex.dev
[parquet]: https://parquet.apache.org
[custom extension array for genomics]: https://pandas-genomics.readthedocs.io/en/latest/index.html
[nullable strings]: https://anndata.scverse.org/en/stable/fileformat-prose.html#nullable-integers-booleans-and-strings
[`zarr-convention-enum`]: https://github.com/ilan-gold/zarr-convention-enum