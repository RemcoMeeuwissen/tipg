
`Tipg` is designed to be fully customizable, in respect to the OGC standard. This page aims to show some example of customizations.


### HTML Templates

The default `HTML` responses are generated using [Jinja](https://jinja.palletsprojects.com) HTML [templates](https://github.com/developmentseed/tipg/tree/main/tipg/templates).

You can override part or a complete list of templates and then provide `TIPG_TEMPLATE_DIRECTORY` environment to tell the `tipg` application to first look in the provided directory for HTML templates.

When building custom `tipg` application you can set the `templates` attribute of the `Endpoints` Factory.

```python
from fastapi import FastAPI
import jinja2

from tipg.factory import Endpoints

app = FastAPI(openapi_url="/api", docs_url="/api.html")

templates_location = (
    [
        jinja2.FileSystemLoader("{PATH TO YOUR CUSTOM TEMPLATE DIRECTORY}"),
        jinja2.PackageLoader("tipg", "templates"),  # Fallback to default's tipg templates
    ]
)

templates = Jinja2Templates(
    directory="",
    loader=jinja2.ChoiceLoader(templates_location),
)  # type: ignore

ogc_api = Endpoints(templates=templates)
app.include_router(ogc_api.router)
```

Example:

In [`eoAPI`](https://github.com/developmentseed/eoAPI), we use a custom logo by overriding the `header.html` : https://github.com/developmentseed/eoAPI/blob/8a3b3de4e82499994fec022229ac3be70bbc1388/runtime/eoapi/vector/eoapi/vector/templates/header.html

![](https://github.com/developmentseed/tipg/assets/10407788/8c79e668-252b-464c-a50b-8efe7a99d931)


### SQL Functions

`tipg` support SQL functional layers (see [Functions](/advanced/functions/)).

`Functions` will be either found by `tipg` at startup within the specified schemas or by registering them dynamically to the [`pg_temp`](https://www.postgresql.org/docs/current/runtime-config-client.html) schema when creating the [Database connection](https://github.com/developmentseed/tipg/blob/2543707238a97a0527effff710a83f9bea66440f/tipg/db.py#L63-L65).

To `register` custom SQL functions, user can set `TIPG_CUSTOM_SQL_DIRECTORY` environment variable when using `tipg` demo application or set `user_sql_files` option in [tipg.db.connect_to_db](https://github.com/developmentseed/tipg/blob/2543707238a97a0527effff710a83f9bea66440f/tipg/main.py#L90-L109).

```python
from tipg.db import connect_to_db, register_collection_catalog
from tipg.settings import PostgresSettings

postgres_settings = PostgresSettings()

app = FastAPI()

@app.on_event("startup")
async def startup_event() -> None:
    """Connect to database on startup."""
    await connect_to_db(
        app,
        settings=postgres_settings,
        schemas=["public"],
        user_sql_files="tests/fixtures/functions",  # <----
    )
    await register_collection_catalog(
        app,
        schemas=["public"],
        exclude_function_schemas=["public"],
    )
```

```bash
TIPG_DB_EXCLUDE_FUNCTION_SCHEMAS='["public"]' TIPG_CUSTOM_SQL_DIRECTORY=tests/fixtures/functions  uvicorn tipg.main:app --port 8000 --reload
```

```
curl -s http://127.0.0.1:8000/collections\?f\=json | jq -r '.collections[].id' | grep "pg_temp"
pg_temp.landsat_centroids
pg_temp.hexagons_g
pg_temp.hexagons
pg_temp.squares
pg_temp.landsat
```