Ejemplo base de partida FastAPI + SQLModel + Pytest

## Separar main:
Con SQLModel puedes mantener los modelos de base de datos y los modelos de entrada/salida juntos en `models.py`. Para tu CRUD actual es lo más simple.

Yo lo dejaría así:

```text
app/
├── __init__.py
├── main.py
├── database.py
├── models.py
├── api/
│   ├── __init__.py
│   └── heroes.py
├── web/
│   ├── __init__.py
│   └── heroes.py
├── templates/
│   ├── base.html
│   └── heroes/
│       ├── list.html
│       └── detail.html
└── tests/
    ├── __init__.py
    └── test_heroes.py
```

## 1. `models.py`

Aquí colocas tus modelos SQLModel:

```python
from sqlmodel import Field, SQLModel


class HeroBase(SQLModel):
    name: str = Field(index=True)
    age: int | None = Field(default=None, index=True)


class Hero(HeroBase, table=True):
    id: int | None = Field(default=None, primary_key=True)
    secret_name: str


class HeroPublic(HeroBase):
    id: int
    secret_name: str


class HeroCreate(HeroBase):
    secret_name: str


class HeroUpdate(HeroBase):
    name: str | None = None
    age: int | None = None
    secret_name: str | None = None
```

Esto ya cumple la función que normalmente tendría:

```text
schemas/
├── hero.py
```

No necesitas duplicar estructuras innecesariamente.

---

## 2. `database.py`

```python
from sqlmodel import Session, SQLModel, create_engine


sqlite_file_name = "database.db"
sqlite_url = f"sqlite:///{sqlite_file_name}"

connect_args = {"check_same_thread": False}

engine = create_engine(
    sqlite_url,
    connect_args=connect_args,
)


def create_db_and_tables():
    SQLModel.metadata.create_all(engine)


def get_session():
    with Session(engine) as session:
        yield session
```

---

## 3. `api/heroes.py`

Aquí van los endpoints JSON.

```python
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlmodel import Session, select

from app.database import get_session
from app.models import Hero, HeroCreate, HeroPublic, HeroUpdate


router = APIRouter()

SessionDep = Annotated[Session, Depends(get_session)]


@router.post("/heroes/", response_model=HeroPublic)
def create_hero(hero: HeroCreate, session: SessionDep):
    db_hero = Hero.model_validate(hero)

    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)

    return db_hero


@router.get("/heroes/", response_model=list[HeroPublic])
def read_heroes(
    session: SessionDep,
    offset: int = 0,
    limit: Annotated[int, Query(le=100)] = 100,
):
    heroes = session.exec(
        select(Hero).offset(offset).limit(limit)
    ).all()

    return heroes


@router.get("/heroes/{hero_id}", response_model=HeroPublic)
def read_hero(hero_id: int, session: SessionDep):
    hero = session.get(Hero, hero_id)

    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")

    return hero


@router.patch("/heroes/{hero_id}", response_model=HeroPublic)
def update_hero(
    hero_id: int,
    hero: HeroUpdate,
    session: SessionDep,
):
    hero_db = session.get(Hero, hero_id)

    if not hero_db:
        raise HTTPException(status_code=404, detail="Hero not found")

    hero_data = hero.model_dump(exclude_unset=True)

    hero_db.sqlmodel_update(hero_data)

    session.add(hero_db)
    session.commit()
    session.refresh(hero_db)

    return hero_db


@router.delete("/heroes/{hero_id}")
def delete_hero(hero_id: int, session: SessionDep):
    hero = session.get(Hero, hero_id)

    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")

    session.delete(hero)
    session.commit()

    return {"ok": True}
```

---

## 4. `web/heroes.py`

Aquí empezamos a servir Jinja2.

```python
from fastapi import APIRouter, Request
from fastapi.templating import Jinja2Templates


router = APIRouter()

templates = Jinja2Templates(directory="app/templates")


@router.get("/heroes")
def heroes_page(request: Request):
    return templates.TemplateResponse(
        request=request,
        name="heroes/list.html",
        context={},
    )
```

Por ahora solo estamos mostrando la página. Después conectamos la sesión y obtenemos los héroes.

---

## 5. `main.py`

Aquí queda la composición de toda la aplicación:

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.api.heroes import router as api_heroes
from app.database import create_db_and_tables
from app.web.heroes import router as web_heroes


@asynccontextmanager
async def lifespan(app: FastAPI):
    create_db_and_tables()
    yield


app = FastAPI(lifespan=lifespan)

app.include_router(api_heroes)
app.include_router(web_heroes)
```

Esto es justamente lo que quieres conseguir: **`main.py` pequeño**.

---

## 6. `templates/base.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Mi aplicación{% endblock %}</title>
</head>

<body>
    {% block content %}{% endblock %}
</body>
</html>
```

---

## 7. `templates/heroes/list.html`

```html
{% extends "base.html" %}

{% block title %}Heroes{% endblock %}

{% block content %}

<h1>Heroes</h1>

<p>Lista de heroes</p>

{% endblock %}
```

---

## 8. Tests

Tus imports cambian de:

```python
from app.main import Hero, app, get_session
```

a:

```python
from app.database import get_session
from app.main import app
from app.models import Hero
```

El resto de tus tests puede permanecer prácticamente igual.

---

# ¿Y `schemas/`?

**No lo agregaría ahora.**

SQLModel te permite hacer esto:

```text
models.py
│
├── Hero          → tabla BD
├── HeroCreate    → entrada API
├── HeroUpdate    → actualización API
└── HeroPublic    → salida API
```

Por tanto, crear:

```text
schemas/
    heroes.py
```

solo para mover `HeroCreate`, `HeroUpdate` y `HeroPublic` sería más archivos y más imports sin darte una ventaja real en este proyecto.

### ¿Cuándo sí separaría `schemas/`?

Cuando tu aplicación crezca y tengas modelos bastante diferentes:

```text
models/
├── hero.py
├── product.py
└── order.py

schemas/
├── hero.py
├── product.py
└── order.py
```

Ahí sí empieza a tener sentido.

Para **tu catálogo actual**, yo mantendría:

```text
models.py
database.py
api/
web/
templates/
tests/
```

