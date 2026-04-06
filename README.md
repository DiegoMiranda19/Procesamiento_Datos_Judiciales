# 🔵 Procesamiento de Datos Judiciales — Pipeline ETL + Modelo Estrella

<div align="center">

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?style=flat-square&logo=jupyter&logoColor=white)](https://jupyter.org/)
[![pandas](https://img.shields.io/badge/pandas-2.x-150458?style=flat-square&logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![openpyxl](https://img.shields.io/badge/openpyxl-3.x-217346?style=flat-square)](https://openpyxl.readthedocs.io/)
[![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?style=flat-square&logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Universidad Externado](https://img.shields.io/badge/Universidad-Externado%20de%20Colombia-C8102E?style=flat-square)](https://www.uexternado.edu.co/)

**Pipeline de extracción, transformación y carga (ETL) de reportes policiales de salas de detenidos,**  
**con construcción de un modelo estrella dimensional orientado a análisis en Power BI.**

[Ver notebooks](#-notebooks) · [Arquitectura](#-arquitectura) · [Instalación](#-instalación) · [Modelo estrella](#-modelo-estrella) · [Power BI](#-conexión-a-power-bi)

</div>

---

## 📋 Tabla de contenidos

1. [Contexto del proyecto](#-contexto-del-proyecto)
2. [Problema que resuelve](#-problema-que-resuelve)
3. [Arquitectura](#-arquitectura)
4. [Estructura del repositorio](#-estructura-del-repositorio)
5. [Notebooks](#-notebooks)
6. [Instalación](#-instalación)
7. [Uso rápido](#-uso-rápido)
8. [Modelo estrella](#-modelo-estrella)
9. [Conexión a Power BI](#-conexión-a-power-bi)
10. [Stack tecnológico](#-stack-tecnológico)
11. [Hallazgos técnicos clave](#-hallazgos-técnicos-clave)
12. [Contribución](#-contribución)
13. [Licencia](#-licencia)

---

## 🎯 Contexto del proyecto

Este proyecto es desarrollado como trabajo final de la asignatura **Big Data** del **Tercer Semestre** de la [Universidad Externado de Colombia](https://www.uexternado.edu.co/).

El dataset fuente consiste en **22 archivos Excel** de reportes diarios del sistema **"Sala de Detenidos"** de la Policía Nacional de Colombia, con información sobre personas detenidas en instalaciones policiales y URI (Unidades de Reacción Inmediata) a nivel nacional, con cobertura temporal de **enero 2024 a abril 2025**.

| Atributo | Detalle |
|---|---|
| Fuente | Policía Nacional de Colombia — Dirección de Seguridad Ciudadana |
| Período | Enero 2024 – Abril 2025 |
| Archivos fuente | 22 archivos `.xlsx` |
| Registros consolidados | 23.367 filas |
| Cobertura geográfica | 9 Regionales (RG) · 55 Unidades · ~1.000 salas |
| Operadores | AYALA, DÁVILA, ACOSTA, GARCIA, TOBÓN |

---

## 🔧 Problema que resuelve

Los archivos fuente presentan múltiples desafíos de ingesta que hacen imposible procesarlos con `pd.read_excel` directamente:

| Problema | Impacto sin tratamiento | Solución implementada |
|---|---|---|
| **Celdas combinadas** en encabezados (240 merged cells por archivo) | `NaN` masivo en columnas de título | Expansión completa con `openpyxl` antes de leer |
| **Celdas combinadas verticales** en datos (RG, UNIDAD: hasta 139 filas) | Solo 469 filas leídas de 1.079 reales | Propagación de valores + `ffill` post-`dropna` |
| **Múltiples hojas** por archivo (hasta 5: `DÍA`, `BASE`, `NEUTRO.`, `DÍA (2)`)  | Se leía la hoja auxiliar incorrecta | Selección por doble criterio: F4=RG+UNIDAD & merged>0 |
| **Vocabulario heterogéneo** entre operadores | Columnas duplicadas tras concatenar | Diccionario de homologación centralizado |
| **3 niveles de encabezado** multinivel | Nombres de columna vacíos o incorrectos | Construcción jerárquica BLOQUE\_SUBGRUPO\_CAMPO |
| **Formato `join='inner'`** excluía columnas | `KeyError: None` en etapas posteriores | Búsqueda por nombre con fallback, nunca por índice |

---

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ARQUITECTURA DEL PIPELINE                           │
├──────────────┬──────────────────────────────────┬───────────────────────────┤
│   FUENTE     │        PIPELINE (NB 1)           │      DWH (NB 2)           │
├──────────────┼──────────────────────────────────┼───────────────────────────┤
│              │  ETI → Lectura openpyxl           │  dim_tiempo  (22 × 15)    │
│  22 archivos │  ODC → Selección de hoja          │  dim_ubicacion (1083 × 6) │
│  Excel .xlsx │  ETL 1 → Expansión merged cells   │  dim_instalacion (2 × 5)  │
│              │  ETL 2 → Homologación vocabulario │  dim_reporte  (22 × 5)    │
│              │  ↓                                │  fact_detenidos           │
│              │  Staging CSV                      │  (23.367 × 47 métricas)   │
│              │  23.367 filas · 63 cols           │                           │
└──────────────┴──────────────────────────────────┴───────────────────────────┘
                                                           ↓
                                                      Power BI
                                                      Dashboard
```

---

## 📁 Estructura del repositorio

```
Procesamiento_Datos_Judiciales/
│
├── 📓 notebooks/
│   ├── 01_pipeline_etl.ipynb          # Pipeline principal ETL
│   └── 02_modelo_estrella.ipynb       # Construcción del DWH dimensional
│
├── 📊 DWH/                            # Tablas del modelo estrella (generadas)
│   ├── dim_tiempo.csv
│   ├── dim_ubicacion.csv
│   ├── dim_instalacion.csv
│   ├── dim_reporte.csv
│   └── fact_detenidos.csv
│
├── 📈 powerbi/
│   └── sala_detenidos.pbix            # Archivo Power BI (opcional)
│
├── 📄 requirements.txt                # Dependencias del proyecto
├── 📄 LICENSE                         # MIT License
└── 📄 README.md                       # Este archivo
```

> **Nota:** Los archivos `.xlsx` fuente no se incluyen en el repositorio por contener datos sensibles de carácter judicial. La carpeta `DWH/` se genera automáticamente al ejecutar el Notebook 2.

---

## 📓 Notebooks

### Notebook 1 — Pipeline ETL (`01_pipeline_etl.ipynb`)

Procesa los 22 archivos Excel fuente y genera el CSV consolidado de staging.

| Celda | Función |
|---|---|
| 1 | Configuración de rutas (única celda a editar) |
| 2 | Importaciones y verificación de versiones |
| 3 | Funciones auxiliares (`limpiar`, `deduplicar`, `extraer_fecha`, `extraer_operador`) |
| 4 | `seleccionar_hoja()` + `transformar_archivo()` — motor principal del pipeline |
| 5 | Prueba unitaria con un archivo antes de procesar todos |
| 6 | Fase 1: Extracción y transformación individual |
| 7 | Auditoría de estructura entre archivos |
| 8 | Fase 2: Homologación de vocabulario |
| 9 | Fase 3: Consolidación con `join='inner'` |
| 10 | Fase 4: Tipado de columnas (`Int64` nullable) |
| 11 | Resumen y vista previa del resultado |
| 12 | Diagnóstico de calidad de datos |
| 13 | Exportación a CSV con `encoding='utf-8-sig'` |
| 14 | Modo incremental: agregar nuevos archivos sin reprocesar |

**Output:** `sala_detenidos_consolidado_YYYYMMDD_HHMM.csv`

---

### Notebook 2 — Modelo Estrella (`02_modelo_estrella.ipynb`)

Lee el CSV de staging y construye las 5 tablas del modelo dimensional.

| Celda | Función |
|---|---|
| 1 | Configuración de rutas fuente y destino DWH |
| 2 | Importaciones y carga del staging |
| 3 | `dim_tiempo` — 15 atributos temporales en español |
| 4 | `dim_ubicacion` — jerarquía RG → Unidad → Sala |
| 5 | `dim_instalacion` — PONAL vs URI |
| 6 | `dim_reporte` — metadatos de cada reporte diario |
| 7 | `fact_detenidos` — 47 métricas + 6 KPIs calculados |
| 8 | Validación del modelo (integridad referencial) |
| 9 | Exportación de las 5 tablas a `DWH/` |
| 10 | Modo incremental para nuevas fechas |

**Output:** 5 archivos CSV en la carpeta `DWH/`

---

## ⚙️ Instalación

### Requisitos previos

- Python 3.10 o superior
- pip o conda
- Jupyter Notebook / JupyterLab / VS Code con extensión Jupyter

### Clonar el repositorio

```bash
git clone https://github.com/DiegoMiranda19/Procesamiento_Datos_Judiciales.git
cd Procesamiento_Datos_Judiciales
```

### Instalar dependencias

```bash
pip install -r requirements.txt
```

O instalación manual:

```bash
pip install pandas>=2.0 openpyxl>=3.1 numpy>=1.24 jupyter
```

### Contenido de `requirements.txt`

```
pandas>=2.0.0
openpyxl>=3.1.0
numpy>=1.24.0
jupyter>=1.0.0
```

---

## 🚀 Uso rápido

### 1. Configurar la ruta de los archivos fuente

En la **Celda 1** del Notebook 1, editar únicamente esta variable:

```python
RUTA_BASE = r'C:\ruta\a\tu\carpeta\con\archivos_excel'
```

### 2. Ejecutar el pipeline ETL

```bash
jupyter notebook notebooks/01_pipeline_etl.ipynb
```

Ejecutar todas las celdas en orden. El proceso tarda aproximadamente 2-5 minutos dependiendo del equipo.

**Output esperado:**
```
✅ Ruta base: C:\...
📖 SALA DE DETENIDOS DÍA 23012024 IT. AYALA.xlsx
   Hoja seleccionada : 'DÍA'
   Merged cells expandidas  : 240
   ✅ Resultado: 1069 filas × 89 columnas
...
✅ Dataset limpio: 23.367 filas × 63 columnas
✅ Archivo guardado. Tamaño: X.XX MB
```

### 3. Construir el modelo estrella

En la **Celda 1** del Notebook 2, actualizar las rutas:

```python
RUTA_CSV_STAGING = r'C:\ruta\al\staging.csv'
RUTA_DWH        = r'C:\ruta\destino\DWH'
```

```bash
jupyter notebook notebooks/02_modelo_estrella.ipynb
```

**Output esperado:**
```
✅ dim_tiempo            →  22 filas × 15 cols
✅ dim_ubicacion         → 1.083 filas × 6 cols
✅ dim_instalacion       →   2 filas × 5 cols
✅ dim_reporte           →  22 filas × 5 cols
✅ fact_detenidos        → 23.367 filas × 47 cols
FKs con NaN: 0 ✅
```

---

## ⭐ Modelo estrella

El modelo sigue un esquema estrella clásico con una tabla de hechos central y cuatro dimensiones.

```
              dim_tiempo
              (id_tiempo)
                   │
dim_ubicacion ─── fact_detenidos ─── dim_reporte
(id_ubicacion)    (id_hecho)         (id_reporte)
                   │
              dim_instalacion
              (referencia semántica)
```

### Dimensiones

#### `dim_tiempo`
| Campo | Descripción |
|---|---|
| `id_tiempo` | PK surrogate |
| `fecha` | Fecha en formato `YYYY-MM-DD` |
| `anio` | Año numérico |
| `trimestre` | 1-4 |
| `trimestre_label` | `Q1-2024`, `Q2-2025`, etc. |
| `mes_num` | 1-12 |
| `mes_nombre` | Nombre en español (Enero, Febrero…) |
| `mes_anio` | `2024-01`, `2025-03`, etc. |
| `semana_anio` | Semana ISO del año |
| `dia_mes` | Día del mes |
| `dia_semana_nombre` | Lunes, Martes… |
| `es_fin_semana` | Flag 0/1 |
| `es_fin_mes` | Flag 0/1 (día ≥ 28) |
| `periodo` | Semestre: `Enero-Junio 2024`, etc. |

#### `dim_ubicacion`
| Campo | Descripción |
|---|---|
| `id_ubicacion` | PK surrogate |
| `rg_codigo` | Código del Regional (REMSA, RG 1…RG 8) |
| `rg_descripcion` | Nombre completo del Regional |
| `unidad_codigo` | Código de la unidad (MEBOG, DEBOY, etc.) |
| `sala_nombre` | Nombre de la sala/ubicación |
| `tipo_unidad` | Metropolitana / Departamental / Comando |

#### `dim_instalacion`
| Campo | Descripción |
|---|---|
| `id_instalacion` | PK (1=PONAL, 2=URI) |
| `codigo` | `PONAL` / `URI` |
| `nombre` | Nombre completo |
| `entidad_cargo` | Policía Nacional / Fiscalía General |

#### `dim_reporte`
| Campo | Descripción |
|---|---|
| `id_reporte` | PK surrogate |
| `archivo_origen` | Nombre del archivo Excel fuente |
| `fecha_reporte` | Fecha del reporte |
| `operador` | Apellido del oficial responsable |
| `operador_rango` | Rango policial completo |

### Tabla de hechos — `fact_detenidos`

27 métricas PONAL + 20 métricas URI + 6 KPIs calculados:

| KPI calculado | Fórmula |
|---|---|
| `ponal_tasa_hacinamiento` | `personas_salas / capacidad_salas` |
| `uri_tasa_hacinamiento` | `uri_personas / uri_capacidad` |
| `ponal_ratio_guardian_detenido` | `personas_salas / personal_total` |
| `total_detenidos_consolidado` | `ponal_total + uri_total` |
| `ponal_pct_imputados` | `imputados / (imputados + condenados) × 100` |
| `alerta_mas_36h` | Flag 1 si hay detenidos >36h |
| `alerta_hacinamiento` | Flag 1 si tasa > 1.0 |

---

## 📊 Conexión a Power BI

### Paso 1 — Cargar las tablas

1. Abrir **Power BI Desktop**
2. `Inicio` → `Obtener datos` → `Texto/CSV`
3. Cargar los 5 archivos de la carpeta `DWH/` uno a uno

### Paso 2 — Crear las relaciones en la vista Modelo

| Tabla origen | Campo | Tabla destino | Campo |
|---|---|---|---|
| `fact_detenidos` | `id_tiempo` | `dim_tiempo` | `id_tiempo` |
| `fact_detenidos` | `id_ubicacion` | `dim_ubicacion` | `id_ubicacion` |
| `fact_detenidos` | `id_reporte` | `dim_reporte` | `id_reporte` |

Todas las relaciones: cardinalidad **Muchos a uno** (`*:1`), dirección de filtro **Unidireccional**.

### Paso 3 — Visualizaciones sugeridas

| Visual | Campos |
|---|---|
| Mapa de hacinamiento por sala | `sala_nombre`, `ponal_tasa_hacinamiento` |
| Evolución temporal de detenidos | `mes_anio`, `total_detenidos_consolidado` |
| Distribución por género | `ponal_genero_m`, `ponal_genero_f`, `ponal_genero_lgbti` |
| Semáforo de alertas | `alerta_hacinamiento`, `alerta_mas_36h` |
| Composición condenados vs imputados | `ponal_condenados`, `ponal_imputados` |
| Ranking de unidades por detenidos | `unidad_codigo`, suma de `ponal_total_personas_salas` |
| Extranjeros venezolanos por regional | `rg_descripcion`, `ponal_venezolanos` + `uri_venezolanos` |

---

## 🛠️ Stack tecnológico

| Componente | Tecnología | Versión |
|---|---|---|
| Lenguaje | Python | ≥ 3.10 |
| Manipulación de datos | pandas | ≥ 2.0 |
| Lectura Excel | openpyxl | ≥ 3.1 |
| Computación numérica | numpy | ≥ 1.24 |
| Entorno de desarrollo | Jupyter Notebook | ≥ 1.0 |
| Visualización | Power BI Desktop | Última versión |
| Control de versiones | Git + GitHub | — |

---

## 🔍 Hallazgos técnicos clave

Durante el desarrollo del pipeline se identificaron y resolvieron los siguientes problemas no documentados en la literatura estándar de ingesta Excel:

**1. `pd.read_excel` trunca la lectura con merged cells masivas.**  
Archivos con merged cells verticales de 100+ filas (columnas RG y UNIDAD) hacen que pandas detecte fin de datos prematuramente, devolviendo 469 filas de 1.079 reales. Solución: `openpyxl` puro con expansión previa.

**2. `wb.active` apunta a la hoja equivocada en 4 de 22 archivos.**  
Algunos operadores guardaban el archivo con una hoja auxiliar activa (`DÍA (2)`, `DÍA (3)`). Solución: selección por doble criterio estructural en lugar de usar `wb.active`.

**3. `join='inner'` en `pd.concat` puede devolver solo columnas de metadatos.**  
Si una columna tiene nombre ligeramente distinto en un solo archivo, el join la elimina y desplaza todos los índices. Solución: búsqueda de columnas siempre por nombre, nunca por posición.

**4. `dropna` debe ejecutarse antes del `ffill`, no después.**  
El orden incorrecto propaga valores de ID a filas fantasma que luego sobreviven el filtro, inflando artificialmente el conteo de registros.

**5. Los archivos tienen 3 niveles de encabezado en filas 4, 5 y 6, no en 1, 2 y 3.**  
Las primeras 3 filas son metadata institucional. El ancla real siempre está en la fila 4 con 240 merged cells, lo que requiere detección por contenido en lugar de posición fija.

---

## 🤝 Contribución

Este es un proyecto académico individual. Si encuentras un error o tienes una sugerencia, puedes abrir un [Issue](https://github.com/DiegoMiranda19/Procesamiento_Datos_Judiciales/issues).

Para contribuir:

```bash
# 1. Fork del repositorio
# 2. Crear una rama para tu cambio
git checkout -b fix/nombre-del-fix

# 3. Hacer commit
git commit -m "fix: descripción del cambio"

# 4. Push y Pull Request
git push origin fix/nombre-del-fix
```

Se usa la convención **Conventional Commits** para los mensajes:
- `feat:` nueva funcionalidad
- `fix:` corrección de error
- `docs:` cambios en documentación
- `refactor:` refactorización sin cambio de comportamiento
- `data:` actualización de datos o configuración

---

## 📄 Licencia

Este proyecto está bajo la licencia **MIT**. Consulta el archivo [LICENSE](LICENSE) para más detalles.

```
MIT License — Copyright (c) 2025 Diego Miranda
```

---

## 👤 Autor

**Diego Miranda**  
Estudiante de Big Data — Universidad Externado de Colombia  
GitHub: [@DiegoMiranda19](https://github.com/DiegoMiranda19)

---

<div align="center">

Desarrollado como proyecto final de **Big Data** · Universidad Externado de Colombia · 2025

</div>
