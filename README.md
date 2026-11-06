# The Data-Bridge — Documentation Site

Public documentation for both editions, at https://docs.tdb.jiracorp.co.in. MkDocs Material.

## Local development

```bash
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000
```

## Before pushing

```bash
mkdocs build --strict
python3 ../tdb-internal/scripts/check_docs_versions.py   # samples match the shipped releases
python3 ../tdb-internal/scripts/check_env_reference.py   # every TDB_* variable is documented
```

Text describing a release goes out only after that release is tagged.

## Deploy

Push to `main`: `.github/workflows/deploy.yml` builds with `--strict` and publishes to GitHub Pages.
