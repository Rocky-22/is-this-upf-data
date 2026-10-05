# Is this UPF? — product database

The SQLite database behind the **Is this UPF?** iOS app: UK packaged foods from [Open Food Facts](https://world.openfoodfacts.org), each with a [NOVA](https://doi.org/10.1017/S1368980018003762) group (1–4, from unprocessed to ultra-processed) and the ingredients that decided it.

Download it from **[Releases](../../releases)**. Each release is one build, tagged `db-<version>`. The version is the Open Food Facts snapshot date, the version of the classification rules, and a pipeline revision (e.g. `db-20261004-0.4.0-r2`). The revision goes up when the processing code changes the database built from the same snapshot and rules, for example better detection of non-English ingredient lists. It is never reset. The first release, `db-20261004-0.4.0`, came before the revision was added; `-r2` replaces it.

| File | What it is |
|---|---|
| `upf.sqlite.gz` | The database, gzipped |
| `SHA256SUMS` | SHA-256 of `upf.sqlite.gz` and of the uncompressed `upf.sqlite` |

```sh
gunzip upf.sqlite.gz && shasum -a 256 -c SHA256SUMS --ignore-missing
sqlite3 upf.sqlite "SELECT name, brand, nova_group FROM products WHERE barcode = '5000157024671'"
```

The tables are described in [SCHEMA.md](SCHEMA.md).

## What's in it

- About 179,000 products sold in the UK, from an Open Food Facts snapshot. Names and brands are cleaned and de-duplicated.
- For each product with an English ingredients list, a NOVA group computed from the ingredients text alone. The ingredients that make a product ultra-processed ("markers", such as emulsifiers, flavourings or glucose syrup) are recorded with their position in the text and a plain-English reason.
- Products without a usable ingredients list are included with no group (shown in the app as "Unknown"), with the reason.
- A full-text search index and a spelling vocabulary over product names and brands.

NOVA groups here are computed by the Is this UPF? project, not by Open Food Facts, and can differ from the NOVA group shown on Open Food Facts. They are information about how a food is processed, **not health or medical advice**.

## Licence and attribution

This database is a derivative of the Open Food Facts database and is made available under the **[Open Database License (ODbL) 1.0](LICENSE)**. Individual contents of the database are licensed under the **[Database Contents License (DbCL) 1.0](LICENSE-DbCL.txt)**.

If you use or share it, credit Open Food Facts and keep it under the same licence:

> Contains data from Open Food Facts (https://world.openfoodfacts.org), available under the Open Database License (ODbL) 1.0.

Product images aren't included. On Open Food Facts they're licensed separately (CC BY-SA).

To correct a product, edit it on [Open Food Facts](https://world.openfoodfacts.org). Fixes there reach this database in the next build.
