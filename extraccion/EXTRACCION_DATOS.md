# Extracción, organización y mantenimiento de datos — Especificación formal

> Documento de referencia para cualquier IA o desarrollador que deba replicar,
> extender o auditar el flujo de datos del proyecto `horario-usm`.
>
> **Regla de oro:** si algo puede hacerse ejecutando un script o validando un
> esquema, se hace así (sin interpretación libre). Los tres artefactos de la
> "fuente de verdad" son:
>
> 1. `resources/export_carreras.py` — clasificación/ordenamiento ejecutable.
> 2. `docs/schemas/*.schema.json` — contratos de datos validables.
> 3. `docs/categorias_carreras.json` — mapeo exhaustivo nombre → categoría.

---

## 1. Visión general

- **Frontend**: SvelteKit + TypeScript (`src/`). Consume únicamente JSONs estáticos.
- **Backend de datos**: Python (`resources/`). Scrapea **SIGA** (`siga.usm.cl`).
- **Datos generados**: `src/lib/data/*.json`.
- **Automatización**: GitHub Actions (`.github/workflows/update_data.yml`).

No hay API pública. Todo el scraping es parseo de HTML de páginas JSP encadenadas.

---

## 2. Fuente de datos (SIGA)

Los endpoints se consumen en `resources/modules/scrapers.py`. El flujo es un
"wizard" de inscripción: cada página entrega las opciones del `<select>`
siguiente.

| Endpoint | Entrega | Usado por |
|---|---|---|
| `insc_plan_frame1.jsp` | Sedes y jornadas | `get_sedes_and_jornadas` |
| `insc_plan_frame2.jsp?sede&jornada` | Carreras de una sede+jornada | `get_carreras` |
| `insc_plan_frame3.jsp?carrera&sede&jornada` | Menciones/especialidades | `get_mencion_especializacion` |
| `insc_plan_frame4.jsp?mencion&sede&carrera` | Planes de la mención | `get_planes_carrera` |
| `insc_plan_frame5.jsp?plan&sede&carrera&mencion` | Info de carrera (duración, créditos) | `get_info_carrera` |
| `insc_ListPlanAsignatura.jsp` + `insc_plan_frame6.jsp` | Malla: asignaturas por semestre | `get_malla_carrera` |
| `insc_plan_requisito.jsp?plan&cod_asign` | Requisitos y equivalencias | `_fetch_requisitos_asignatura` |
| `insc_ListProgTodasAsign.jsp?sede&jornada&ano&semestre` | Horarios, paralelos, salas, profesores, cupos | `get_programacion_asignaturas` |
| `prog_oai/oai_academia.jsp` | Programas académicos | `get_programas_academicos` |

Autenticación: cookie `JSESSIONID` (obtenida en `resources/modules/auth.py` con
`SIGA_LOGIN`/`SIGA_PASSWD`/`SIGA_SERVER`, o pasada por argv/stdin).

---

## 3. Pipeline de extracción

Orquestador: `resources/generar.py`. Flags: `--ramos`, `--carreras`, `--programas`,
`--profesores`, `--reviews` (sin flags → solo ramos + profesores + reviews).

- **Paralelismo**: `multiprocessing.Pool`. Ramos por sede; mallas por carrera
  (`workers.py`).
- **Memorización**: `get_malla_carrera` recibe `memo_ramos` para no re-descargar
  requisitos de ramos ya vistos (clave: `sigla`).
- **Parsing de requisitos**: forma normal disyuntiva (DNF). Lista de listas:
  elementos dentro de un grupo = **Y**; grupos distintos = **O**
  (`_parse_siga_table`).

---

## 4. Contratos de datos

Los esquemas formales (JSON Schema, validados contra los datos reales) están en:

| Archivo generado | Schema |
|---|---|
| `planes_carreras.json` | `docs/schemas/planes_carreras.schema.json` |
| `horario_asignaturas.json` | `docs/schemas/horario_asignaturas.schema.json` |
| `programas_academicos.json` | `docs/schemas/programas_academicos.schema.json` |
| `metadata.json` | `docs/schemas/metadata.schema.json` |

### 4.1 `planes_carreras.json`

Array de carreras. Cada carrera:

```
{ "código": "13-1", "jornada": "Diurna", "nombre": "Arquitectura",
  "sede": "Casa Central Valparaíso",
  "menciones/especialidades": { "<idx>": { "nombre": "...", "planes": {
       "<plan_id>": { "plan": "<etiqueta>", "malla": [ semestre... ] } } } } }
```

Ramo de malla (`RamoCarrera`):

```
{ "nombre": str, "creditos": int, "departamento": str,
  "requisito_licenciatura": bool,
  "horas": { "teoricas": int, "practicas": int, "laboratorios": int, "ayudantias": int },
  "requisitos": [[ { "sigla": str, "tipo": "PRE"|"CO"? } ]],
  "equivalencias": [[ { "sigla": str } ]] }
```

### 4.2 `horario_asignaturas.json`

Anidamiento: `sede → jornada → periodo ("YYYY-S") → sigla → paralelo → ramo`.

Ramo (por paralelo): `{ nombre, sigla, paralelo, cupo: int, departamento,
profesor: [str], horario: [ bloque ] }`.

Bloque: `{ bloque: int, tipo, sala, campus, profesor, dia: 0..6 }`
(`dia` 0=Lunes … 6=Domingo; ver `DIAS` en `resources/modules/utils.py`).

### 4.3 `programas_academicos.json`

Anidamiento: `sede → departamento → tipo → sigla → ramo`.
`tipo ∈ { IMPAR, PAR, AMBOS, ELECTIVO }`.
Ramo: `{ nombre: str, creditos: str, programa: str }` (⚠ `creditos` es **string**).

### 4.4 `metadata.json`

`{ version, status, generatedAt{unix,iso}, system{scraperVersion,environment,executionTimeSeconds}, stats{totalAsignaturas,totalParalelos}, files{<name>:{hash,updatedAt,size?,cambiosUltimaEjecucion?}} }`.

---

## 5. Invariantes (no negociables)

1. **`plan_id` (clave del diccionario `planes`) es numérico y monotónico**.
   Es el `value` del `<select>` de SIGA y crece con el tiempo. **Es el único
   criterio válido para ordenar planes.**
2. **`plan` (campo interno) es SOLO una etiqueta** (ej. `"7310"`, `"3 (2019)"`,
   `"PPT15"`). **No usar para ordenar ni comparar.**
3. Los planes cuya etiqueta contiene **`"No Vigente"` se descartan** en
   `workers.py` (no llegan al JSON).
4. Los requisitos/equivalencias siempre están en DNF (nunca una lista plana).
5. Las escrituras son atómicas y condicionales por hash MD5
   (`utils.write_if_modified`).

---

## 6. Clasificación de carreras (reglas ejecutables)

Implementada en `resources/export_carreras.py::clasificar`. Evaluación en orden
de precedencia (la primera coincidencia gana):

| # | Condición (regex, case-insensitive) | Categoría |
|---|---|---|
| 1 | nombre ∈ `EXCLUIDAS_EXPLICITAS` | **descartar** |
| 2 | `^(Doc\.\|Doctorado)` | Doctorado |
| 3 | `^(Mag\.\|Magíster\|Magister\|Master)` | Magíster |
| 4 | `^(Lic\.\|Licenciatura)` | Licenciatura |
| 5 | `^(Téc\.\|Tec\.\|Técnico)` | Técnica |
| 6 | `^(Ing\.\|Ingeniería\|Ingenieria\|I\.Civil\|I\.Ejec)` | Profesional |
| 7 | nombre ∈ `PROFESIONALES_EXACTAS` | Profesional |
| 8 | (ninguna) | **descartar** |

Constantes:

```
EXCLUIDAS_EXPLICITAS = { "Formación de Piloto Comercial" }
PROFESIONALES_EXACTAS = { "Arquitectura", "Construcción Civil", "Químico", "Piloto Comercial" }
```

Todo lo que **no** cae en una categoría se descarta (cursos, diplomados,
diplomas, programas, nivelación, talleres, workshops, postítulos, propedéutico,
"Primer Año", "Especial", "Subtécnico", "Ing. Civil Común" ya re-incluida, etc.).

> El mapeo **exhaustivo** (347 nombres reales → categoría o descarte) está en
> `docs/categorias_carreras.json`. Es generado y no debe editarse a mano:
> `python resources/export_carreras.py` es la fuente; el JSON es la instantánea.

---

## 7. Determinación "Malla Nueva" vs "Malla Antigua"

Dentro de una carrera (y por cada `plan_id` distinto):

1. Ordenar las claves `plan_id` numéricamente **ascendente**.
2. La **clave mayor** = **Malla Nueva**; el resto = **Malla Antigua**.
3. Un único plan = **Malla Nueva**.
4. Sin planes = **Sin dato**.

Casos de prueba que validan el criterio (clave, no etiqueta):

| Carrera | plan_id | etiqueta | Resultado |
|---|---|---|---|
| Ing. Civil Informática | `10730000010` | `7310` | Malla Nueva |
| Ing. Civil Informática | `10730000009` | `7313` | Malla Antigua |
| Arquitectura | `10130000005` | `1325` | Malla Nueva |
| Arquitectura | `10130000004` | `1311` | Malla Antigua |
| Téc. Univ. Informática (Viña) | `30920000003` | `3 (2019)` | Malla Nueva |
| Téc. Univ. Informática (Viña) | `30920000001` | `1 (2003 - 2020)` | Malla Antigua |

⚠ Ordenar por la **etiqueta** (`7310` vs `7313`) produce el resultado invertido.

---

## 8. Manejo de actualizaciones de la fuente

1. **Cron + manual**: `update_data.yml` corre `generar.py` por cron (3 niveles
   de prioridad según mes) y `workflow_dispatch` (manual, con `--args`).
2. **Enumeración total**: cada corrida recorre **todas** las sedes × jornadas ×
   carreras × menciones × planes. Si SIGA agrega una sede, carrera, mención,
   plan o paralelo **nuevo**, aparece solo en la siguiente corrida.
3. **Escritura condicional**: se escribe únicamente si cambió el hash MD5
   (`os.replace` atómico) → evita commits vacíos.
4. **Diff incremental**: `utils.calcular_diff_ramos` compara viejo vs nuevo a 5
   niveles y emite eventos tipados (`NUEVA_SEDE`, `NUEVA_ASIGNATURA`,
   `NUEVO_PARALELO`, `CAMBIO_CUPO`, `CAMBIO_PROFESOR`, `CAMBIO_HORARIO`, …) que
   se agregan a `src/lib/data/historial_cambios.jsonl`.
5. **Aborto seguro**: ante sesión expirada o inconsistencia de parseo, aborta
   **sin escribir** (protege la integridad de los datos).

---

## 9. Organización/consumo en el frontend

`src/lib/data/data.svelte.ts`:

- Construye lookups por sigla y **hidrata** cada asignatura con créditos y
  requisitos específicos de la carrera del usuario.
- `normalizeDepto()` unifica nombres de departamentos (ej.
  `electrotecnia → Electrónica`, `DEFIDER`, acentos).
- Tipos en `src/lib/types/horario.ts`; validación Zod en `src/lib/data/schemas.ts`.

---

## 10. Cómo ejecutar la verificación completa

```bash
# 1) Validar esquemas contra los datos reales
python3 - <<'PY'
import json, jsonschema, glob
for s in sorted(glob.glob('docs/schemas/*.schema.json')):
    d = s.replace('docs/schemas/','src/lib/data/').replace('.schema','')
    jsonschema.validate(json.load(open(d)), json.load(open(s)))
    print('OK', s)
PY

# 2) Regenerar la clasificación (CSV + opcional XLSX)
python resources/export_carreras.py --csv carreras_usm.csv --xlsx carreras_usm.xlsx
```

---

## 11. Advertencias para quien extienda esto

- La clasificación por prefijo es **heurística**. Si SIGA agrega una carrera con
  un nombre que no calza con ningún patrón, caerá en "descartado" en silencio.
  Revisar `docs/categorias_carreras.json` (los `null`) tras cada actualización.
- Si aparece una sede o jornada nueva, agregarla a `SEDE_ORDER`/`JORNADA_ORDER`
  en `resources/export_carreras.py` para conservar el orden determinista.
- `programas_academicos.json` guarda `creditos` como **string**; no asumir int.
