# data-quality-forge

Plantilla de enseñanza para la **Semana 9, Laboratorio 4 — limpieza y perfilado de datos**. Los estudiantes perfilan un lote pequeño de pedidos intencionalmente sucio en PySpark y practican casting seguro, enrutamiento a cuarentena (dead-letter) y deduplicación con ventanas. El notebook es un laboratorio, no una clave de respuestas.

## Objetivos de aprendizaje

Cada dimensión se mide antes de “arreglarla”:

| Dimensión | Qué deberías poder hacer |
| --- | --- |
| **Completitud** (completeness) | Distinguir un valor real de null, texto vacío y sentinelas como `N/A` o `null`. `isNotNull()` solo no basta. |
| **Validez** (validity) | Parsear fechas, montos y cantidades con `try_cast` o un `when` / null explícito. Rechazar un `.cast` silencioso que convierte basura en null sin una regla. Conservar solo valores dentro del dominio del lab (monto positivo, cantidad ≥ 1, status conocido). |
| **Unicidad** (uniqueness) | Colapsar valores duplicados de `order_id` con `row_number` sobre un `Window.partitionBy` y declarar qué copia sobrevive (el `order_date` más reciente). |
| **Consistencia** (consistency) | Verificar acuerdo entre campos. En este lab, el país ISO-2 y la moneda deben coincidir (`ES`/`FR`→EUR, `MX`→MXN, `US`→USD). |
| **Precisión** (accuracy) | Comparar un monto parseado con su referencia dentro de una tolerancia absoluta de `0.01`. Los números faltantes no son precisos. |

## Estructura

```
.github/workflows/deploy_databricks.yml   # pytest → importar → upsert/ejecutar Job
notebooks/Week09_Lab04_DataCleansing_and_Profiling.py  # lab del estudiante (TODOs)
notebooks/Pipeline_Bronze_Quarantine_Silver.py         # notebook ejecutable de CI / Job
databricks/jobs/data_quality_pipeline.json
databricks/jobs/README.md
scripts/upsert_databricks_job.py
data_quality/validation.py                # reglas en Python puro (sin Spark)
tests/test_data_validation_rules.py
pyproject.toml                            # pone la raíz del repo en el path de pytest
requirements-dev.txt
```

`data_quality/validation.py` es el contrato sin cluster para las mismas ideas (null / completitud, rangos de validez, unicidad, consistencia país-moneda, tolerancia absoluta). El notebook de Databricks las reimplementa en Spark detrás de prompts `# TODO(estudiante):`.

## Ejecutar las pruebas

Python 3.10 o superior. No hace falta instalar Spark ni tener credenciales de Databricks.

```bash
python -m pip install -r requirements-dev.txt
pytest -q
```

## Cómo despliega el CI

Workflow: [`.github/workflows/deploy_databricks.yml`](.github/workflows/deploy_databricks.yml).

Se ejecuta en cada **push a `main`** y cuando alguien lo inicia con **workflow_dispatch**.

1. **Pruebas unitarias** — hace checkout del repo, configura Python, instala pytest y `requirements-dev.txt`, y corre `pytest -q`.
2. **Importación** — solo si las pruebas pasan (`needs: test`). El job:
   - falla de inmediato si faltan `DATABRICKS_HOST` o `DATABRICKS_TOKEN`;
   - crea `/Shared/Week09_Lab04_DataCleansing` con la API Workspace `mkdirs` si aún no existe;
   - importa cada archivo `notebooks/*.py` con la API Workspace `import` como **SOURCE / PYTHON**, con **overwrite**.
3. **Upsert + ejecución del Job** — `python3 scripts/upsert_databricks_job.py` con `RUN_JOB=true` crea o actualiza el Job de Databricks `data-quality-forge-pipeline` y arranca una ejecución del notebook de pipeline (no el lab del estudiante).

Los notebooks quedan en:

```
/Shared/Week09_Lab04_DataCleansing/Week09_Lab04_DataCleansing_and_Profiling
/Shared/Week09_Lab04_DataCleansing/Pipeline_Bronze_Quarantine_Silver
```

`DATABRICKS_HOST` debe ser solo la URL del workspace, por ejemplo `https://adb-xxxx.azuredatabricks.net` o `https://adb-xxxx.cloud.databricks.com`, sin token en el valor.

## Secretos

Guarda `DATABRICKS_HOST` y `DATABRICKS_TOKEN` como secretos del repositorio de GitHub Actions (Settings → Secrets and variables → Actions). Agrégalos en este repositorio antes de que el workflow de deploy pueda funcionar.

Nunca subas tokens, archivos `.env` ni `.databrickscfg`. No pegues un personal access token en el notebook, el workflow ni un pull request. El workflow lee los secretos desde el entorno de Actions y no los imprime.

## Cómo trabajar el laboratorio

Abre el notebook importado en Databricks y ejecútalo desde arriba. El setup y el lote sintético ya están listos. Cada sección posterior lanza `NotImplementedError` hasta que reemplaces el `# TODO(estudiante):` con tu propio código Spark. Cuando una fila pueda normalizarse o ir a cuarentena (decimales europeos, `USA` vs `US`, fechas con `/`), escribe la decisión en un comentario corto junto al código.

## Job del pipeline de datos (bronze → quarantine → silver)

El CI también mantiene un Job de Databricks llamado **`data-quality-forge-pipeline`**.

| Pieza | Detalle |
| --- | --- |
| Task | `bronze_to_silver` |
| Notebook | `/Shared/Week09_Lab04_DataCleansing/Pipeline_Bronze_Quarantine_Silver` |
| Definición | [`databricks/jobs/data_quality_pipeline.json`](databricks/jobs/data_quality_pipeline.json) |
| Script de upsert | [`scripts/upsert_databricks_job.py`](scripts/upsert_databricks_job.py) |

Flujo: pedidos sintéticos en **bronze** → perfil rápido → casts seguros → filas inválidas a **quarantine** (códigos de razón) → dedupe con ventana sobre `order_id` → **silver**. Tras una importación exitosa del notebook, Actions actualiza el Job por nombre y dispara `run-now`.

El notebook del Lab 4 del estudiante sigue basado en TODOs; el Job nunca apunta a él. Consulta [`databricks/jobs/README.md`](databricks/jobs/README.md) para notas de Free Edition / serverless.
