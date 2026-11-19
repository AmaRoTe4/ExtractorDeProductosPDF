# Extractor de tablas

Saca tablas de PDFs (listas de precios, catálogos) y las exporta a CSV. Se selecciona la tabla una vez con el mouse,
se guarda como **plantilla**, y los próximos PDF del mismo formato salen solos.

```
PDF ──▶ seleccionar y replicar ──▶ datos ──▶ mapear (estándar de salida) ──▶ resultado / CSV
        (motor Python)                         (campos, condiciones)         (plantilla guardada en MySQL)
```

- **Seleccionar:** dibujás un rectángulo sobre la tabla (como Tabula) y se replica en todas las páginas; las columnas se
  detectan solas (grillas de líneas o de rectángulos finos, o por espacios en blanco).
- **Estándares de salida:** definen los campos del CSV, qué lleva una celda vacía y el formato (separador, decimal, BOM).
  Viene de fábrica **Tienda Web Pro** (usa la receta real del ERP); se puede duplicar, crear uno libre y elegir el
  predeterminado.
- **Mapear:** cada campo sale de una columna, de varias juntas (`{DESCRIPCION} "{UNID}" "{GRUPO}"`) o de un valor fijo;
  condiciones para excluir filas; filas que se pueden revisar e ignorar.

## Requisitos

Bun ≥ 1.3, Python ≥ 3.12 con [uv](https://docs.astral.sh/uv/), MySQL 8 y [Flyway](https://flywaydb.org/) (Community 9.x).

## Primera vez

```bash
cd motor && uv sync && cd ..            # dependencias del motor
(cd backend && bun install)             # backend
(cd frontend && bun install)            # frontend
bash db/migrar.sh                       # crea las bases y aplica las migraciones de db/migrations
```

`db/migrar.sh` crea `extractor_tablas` (desarrollo) y `extractor_tablas_test` (tests). La primera vez arma el `.env`
tomando usuario y clave de `~/.my.cnf`; si no lo tenés, copiá `.env.example` a `.env` y completalo. El `.env` nunca se sube.
No hay usuarios ni login: es una herramienta local.

## Usarlo

```bash
bash scripts/dev.sh                     # motor :8001 · backend :3001 · app :5173
```

Abrí http://localhost:5173. Para probar sin PDFs propios: `cd motor && uv run python ../muestras/generar.py`
(crea PDFs sintéticos en `muestras/pdfs`).

## Formatos de lista que entiende

- **Productos de varias líneas** (arriba código y nombre, abajo marca y precio): se detecta solo («cada producto ocupa 2 líneas») y sale
  **una fila por producto**, con las columnas de la segunda línea como «… · L2». Se activa o ajusta a mano en el paso Datos.
- **Texto que se pasa a la celda de al lado** (una descripción larga sobre la columna del precio): vuelve a su columna. Se informa cuántos casos.
- **Títulos de sección** a la izquierda de la tabla («Agrupación: BOYAS»): se detectan y pasan a la columna `SECCION` (categoría).
- **Etiquetas dentro de la celda** («Marca: UNICA»): en Mapear se pueden quitar («Quitar este texto del valor»); si todas las celdas
  empiezan igual, se propone solo, y un campo «Marca» sin fuente toma esa columna.
- **IVA**: una columna de precio se toma como precio **con IVA** salvo que diga «sin IVA», «neto» o «P.N.». Con dos columnas (sin y con IVA)
  se propone inferir el porcentaje. El archivo exportado no lleva una marca de «con IVA»: se declara al importar la lista en el ERP.
- **Números**: las tablas los muestran como en el PDF (`5.830,80`). El formato de cada columna puede ser coma decimal, punto decimal o
  «Detectar por columna» (si una lista mezcla los dos, lee cada celda según cómo está escrita y avisa).

## Descargas

En el paso Resultado y en **Extraer** se elige el formato: **Excel (.xlsx)** (marcado por defecto; números reales, encabezado fijo, columnas del ancho
justo) o **CSV**. «Precios con 2 decimales» (`5830,80`) viene marcado. El CSV usa el formato del estándar y el fin de línea de tu sistema
(Windows: CRLF; Linux/Mac: LF). El archivo se llama `<PDF analizado>-<estándar o tabla>-<AAAA-MM-DD_HHMM>.xlsx|csv` (la hora es la de tu equipo).
El Excel no trae filtros automáticos (la librería `write-excel-file` no los soporta).

## Listas grandes

Probado con un PDF de 95 MB y 600 hojas (~37.000 filas). Un PDF grande no se sube ni se lee en un solo pedido:
- **Subida en tramos:** el navegador manda el archivo en pedazos de 8 MB (barra «Subiendo X de Y MB»), con reintentos; si un pedido
  falla o se repite, el servidor dice desde qué byte seguir. Límite: 300 MB. Hasta 8 MB va en un solo pedido.
- **Lectura por tramos:** apenas termina la subida aparece el visor; el texto se lee en segundo plano de a 10 hojas
  («Leyendo el texto: 490 de 600 hojas»). Cada tramo queda guardado en `data/motor/cache/<sha>.t/`: si el motor se cierra, retoma
  desde ahí. «Detectar» y «Replicar» andan enseguida (usan las primeras hojas); los datos de todas las páginas esperan a que termine.
- Un solo PDF se lee a la vez, con hasta 8 procesos (`MOTOR_PROCESOS` lo cambia). 600 hojas: ~45 s de lectura, ~1,8 GB de memoria.
- Los PDF se identifican por sha256: volver a subir el mismo archivo no lo vuelve a leer.

## Pruebas

```bash
cd motor && uv run pytest               # extracción, detección, huella, API del motor
cd backend && bun test                  # salida, estándares, API (necesita la base de pruebas)
cd frontend && bun test                 # lógica pura del editor
```

Los tests de MySQL se **omiten con un aviso** si la base de pruebas no está (`bash db/migrar.sh test`).

## Cómo está armado

| Carpeta | Qué hay |
|---|---|
| `motor/` | Python (FastAPI, pdfplumber, pypdfium2): lee el PDF, detecta tablas, extrae por zonas, renderiza páginas |
| `backend/` | Bun + TypeScript: API, plantillas y estándares en MySQL, salida (mapeo, condiciones, CSV) |
| `backend/src/catalogo_listas/` | **Copia** de la receta de Tienda Web Pro (cada archivo dice su origen); no se modifica |
| `frontend/` | React + Vite + Tailwind: el editor de 4 pasos |
| `db/migrations/` | Migraciones Flyway (`V1__…`): plantillas y versiones, estándares de salida, registro de usos |
| `muestras/` | Generador de PDFs sintéticos y su resultado esperado; `reales/` (ignorada por git) para los propios |

Una plantilla **versiona** cada guardado (las versiones nunca cambian). Más detalles de decisiones y reglas del proyecto
en [CLAUDE.md](CLAUDE.md).

## Estado

Hecho: extracción por zonas con replicación, detección de grillas, filas combinadas, estándares, campos compuestos,
condiciones, descargas con formato, plantillas versionadas en MySQL, buscador y «Ampliar» en las tablas, botón
«Replicar» pegado a la selección y buscador de texto dentro del PDF (resalta y salta entre coincidencias).
Pendiente: OCR de PDFs escaneados, texto girado como dato (marca), exportar XLSX, agrupar filas repetidas.
