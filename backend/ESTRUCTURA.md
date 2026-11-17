# Estructura del backend

Guía rápida de dónde va cada cosa para que el código no acabe siendo un desastre.

## Árbol

```
backend/
├── pyproject.toml      # Dependencias y config del proyecto (gestionado con uv)
├── uv.lock             # Versiones exactas (NO editar a mano, sí subir a git)
├── .python-version
├── ESTRUCTURA.md       # Este archivo
└── app/                # Todo el código vive aquí
    ├── __init__.py
    ├── main.py         # Punto de entrada: crea la app y registra los routers
    ├── db/             # Conexión y sesión con la base de datos
    ├── models/         # Modelos de datos (tablas y esquemas)
    ├── routers/        # Endpoints, un archivo por recurso
    └── utils/          # Funciones auxiliares reutilizables
```

## Imports

**Siempre absolutos desde `app`**:

```python
from app.utils.security import hash_password
from app.models.user import User
from app.db.session import get_session
```

**Nunca** relativos (`from ..utils import algo`) ni sin el prefijo (`from utils import algo`).

Para que funcionen, todo se ejecuta **desde la carpeta `backend/`**:

```bash
cd backend
uv run python -m app.main
# con FastAPI:
uv run uvicorn app.main:app --reload
```

## Qué va en cada carpeta

### `app/db/`
Solo lo relacionado con la **conexión** a la base de datos.
- `session.py` → engine, sesión y la dependencia `get_session()`.
- `init_db.py` → creación de tablas / datos iniciales (si hace falta).

No hay lógica de negocio ni endpoints aquí.

### `app/models/`
Definición de los **datos**. Un archivo por entidad (`user.py`, `product.py`...).
- Modelos de tabla (ORM).
- Esquemas de entrada/salida (Pydantic): `UserCreate`, `UserRead`, etc. Pueden ir en el mismo archivo que su entidad.

Los modelos **no** importan de `routers`.

### `app/routers/`
Los **endpoints**. Un archivo por recurso, cada uno con su propio `router`:

```python
# app/routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
def list_users(): ...
```

Y se registran en `main.py`:

```python
from app.routers import users
app.include_router(users.router)
```

Los endpoints deben ser **finos**: reciben la petición, llaman a lo que haga falta y devuelven la respuesta. Si una función crece mucho o se repite, sácala a `utils/`.

### `app/utils/`
Funciones **genéricas y reutilizables** que no encajan en otro sitio: hashing de contraseñas, formateo de fechas, validaciones, etc. Agrúpalas por tema (`security.py`, `dates.py`...), **no** un `utils.py` gigante con todo.

## Reglas de dependencias

Para evitar imports circulares, el flujo va en una sola dirección:

```
main  →  routers  →  models / db / utils
```

- `routers` puede importar de `models`, `db` y `utils`.
- `models`, `db` y `utils` **nunca** importan de `routers` ni de `main`.
- `utils` no debería depender de nada del proyecto salvo, como mucho, `models`.

## Normas generales

- Nombres de archivos y funciones en `snake_case`, clases en `PascalCase`.
- Un archivo = una responsabilidad. Si pasa de ~300 líneas, probablemente haya que dividirlo.
- Los `__init__.py` se quedan vacíos (salvo que haya un motivo claro).
- Nada de credenciales en el código: van en un `.env` (que **no** se sube a git).
- Dependencias siempre con `uv add <paquete>`, nunca con `pip install`.
