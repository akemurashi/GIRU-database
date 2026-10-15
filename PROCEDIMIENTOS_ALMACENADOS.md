# Especificación y Documentación de Procedimientos Almacenados

**Proyecto:** G.I.R.U. (Gestor Institucional de Regulaciones Universitarias)  
**Institución:** Universidad Técnica Federico Santa María (UTFSM)  
**Motor de Base de Datos:** PostgreSQL 16 (AWS RDS) / PostgreSQL 18 (Local)  
**Identificador de Entidad:** UUIDv7 (`iddocumento`, RFC 9562)

---

## 1. Visión General y Filosofía de Diseño

Este módulo procedural centraliza las operaciones de **mutación y actualización** de documentos normativos en la base de datos institucional.

### Principios Arquitectónicos
1. **SSOT (Single Source of Truth):** Todas las modificaciones normativas pasan por una capa procedural canónica que aplica las mismas reglas de negocio, integridad referencial y validaciones de taxonomía, evitando desincronizaciones entre tablas maestras y tablas puente.
2. **Atomicidad Transaccional:** Cada procedimiento opera dentro del contexto transaccional de PostgreSQL. En caso de error o violación de restricciones, la operación se revierte en su totalidad (`ROLLBACK`), garantizando que ningún documento quede en estado corrupto o inconsistente.
3. **Concurrencia Segura:** La asignación de correlativos por categoría (`correlativocategoria`) y la construcción de códigos (`nommetadato`) implementan bloqueos pesimistas a nivel de fila (`FOR UPDATE`) en la tabla `categoria`, previniendo condiciones de carrera (*race conditions*) en ejecuciones paralelas.
4. **Idempotencia y Flexibilidad de Interfaz:** Los parámetros permiten especificar entidades tanto por sus identificadores enteros como por sus nombres o códigos canónicos (ej. `'GI-01'`, `'Decreto'`, `'Vigente'`, `'Informática'`), resolviendo dinámicamente las claves foráneas correspondientes.
5. **Auditoría y Trazabilidad Temporal (`ultima_modificacion`):** La tabla `documento` incorpora el atributo de auditoría `ultima_modificacion TIMESTAMPTZ`. Cada mutación ejecutada por los procedimientos almacenados (modificaciones escalares, categorización y tablas puente satélite) actualiza de forma atómica este campo con `CURRENT_TIMESTAMP`, contando además con el respaldo de un trigger procedural a nivel de motor (`trg_documento_actualizar_ultima_modificacion`).

---

## 2. Catálogo de Procedimientos Almacenados

A continuación se detalla la especificación técnica de cada procedimiento almacenado:

---

### 2.1. `sp_modificar_documento_tipo`

Actualiza el tipo de documento normativo en la tabla `documento`.

#### Firma
```sql
CALL public.sp_modificar_documento_tipo(
    p_iddocumento UUID,
    p_idtipodocumento INTEGER DEFAULT NULL,
    p_nombretipodocumento VARCHAR DEFAULT NULL
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento a modificar.
- `p_idtipodocumento` *(INTEGER, Opcional)*: Clave foránea directa a `tipodocumento`.
- `p_nombretipodocumento` *(VARCHAR, Opcional)*: Nombre canónico del tipo (ej. `'Reglamento'`, `'Decreto'`, `'Resolución'`). Si no se provee el ID, se resuelve por nombre de forma insensible a mayúsculas/minúsculas.

#### Tablas Afectadas
- `public.documento` (`idtipodocumento`)
- `public.tipodocumento` (Consulta de catálogo)

#### Ejemplo
```sql
CALL public.sp_modificar_documento_tipo(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_nombretipodocumento => 'Reglamento'
);
```

---

### 2.2. `sp_modificar_documento_metadatos`

Modifica la categoría temática y macrocategoría institucional del documento. Gestiona automáticamente el recalculo de `correlativocategoria` y el código estandarizado `nommetadato`.

#### Firma
```sql
CALL public.sp_modificar_documento_metadatos(
    p_iddocumento UUID,
    p_idcategoria INTEGER DEFAULT NULL,
    p_nombrecategoria VARCHAR DEFAULT NULL,
    p_numcategoria VARCHAR DEFAULT NULL,
    p_nommetadato VARCHAR DEFAULT NULL,
    p_recalcular_nommetadato BOOLEAN DEFAULT TRUE
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento.
- `p_idcategoria` *(INTEGER, Opcional)*: ID numérico de la categoría destino.
- `p_nombrecategoria` *(VARCHAR, Opcional)*: Nombre de la categoría (ej. `'Gobernanza normativa'`).
- `p_numcategoria` *(VARCHAR, Opcional)*: Código alfanumérico (ej. `'GI-01'`, `'DC-02'`).
- `p_nommetadato` *(VARCHAR, Opcional)*: Código personalizado manual. Si no se envía y la categoría cambia, se recalcula automáticamente.
- `p_recalcular_nommetadato` *(BOOLEAN, Default `TRUE`)*: Indica si se debe regenerar el código con el patrón oficial (`[TEST-]SIGLA-NUM-CORRELATIVO`).

#### Reglas de Negocio
- Al cambiar de categoría, bloquea la fila en `categoria` con `FOR UPDATE`.
- Calcula el siguiente correlativo: `MAX(correlativocategoria) + 1` dentro de la nueva categoría.
- Verifica que el nuevo `nommetadato` no colisione con otro documento existente (`UNIQUE`).

#### Ejemplo
```sql
CALL public.sp_modificar_documento_metadatos(
    p_iddocumento => 'TU-UUID-AQUI',
    p_idmacrocategoria => 1,
    p_idcategoria => 8
);
```

---

### 2.3. `sp_modificar_documento_descripcion`

Actualiza el resumen o descripción sintética del contenido del documento normativo.

#### Firma
```sql
CALL public.sp_modificar_documento_descripcion(
    p_iddocumento UUID,
    p_descripcion TEXT
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento.
- `p_descripcion` *(TEXT, Requerido)*: Nuevo texto descriptivo.

#### Ejemplo
```sql
CALL public.sp_modificar_documento_descripcion(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_descripcion => 'Reglamento general sobre derechos, deberes y disciplina del estudiantado.'
);
```

---

### 2.4. `sp_modificar_documento_numero`

Modifica el identificador numérico o código oficial del acto normativo (ej. `'056/2024'`).

#### Firma
```sql
CALL public.sp_modificar_documento_numero(
    p_iddocumento UUID,
    p_nuevo_numero VARCHAR(50)
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento.
- `p_nuevo_numero` *(VARCHAR(50), Requerido)*: Nuevo número institucional.

#### Reglas de Negocio
- No permite valores nulos ni cadenas vacías tras aplicar `TRIM`.
- Valida que ningún otro documento posea el mismo número (`documento_numero_key`).

#### Ejemplo
```sql
CALL public.sp_modificar_documento_numero(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_nuevo_numero => '056/2024'
);
```

---

### 2.5. `sp_modificar_documento_sesion`

Actualiza la información de cuerpos colegiados: sesión ordinaria/extraordinaria, número de sesión, número de acuerdo y tipo de decisión adoptada.

#### Firma
```sql
CALL public.sp_modificar_documento_sesion(
    p_iddocumento UUID,
    p_idtiposesion INTEGER DEFAULT NULL,
    p_nombretiposesion VARCHAR DEFAULT NULL,
    p_numsesion INTEGER DEFAULT NULL,
    p_numacuerdo INTEGER DEFAULT NULL,
    p_idtipodecision INTEGER DEFAULT NULL,
    p_nombretipodecision VARCHAR DEFAULT NULL,
    p_limpiar_sesion BOOLEAN DEFAULT FALSE
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento.
- `p_idtiposesion` *(INTEGER, Opcional)*: Clave foránea a `tiposesion`.
- `p_nombretiposesion` *(VARCHAR, Opcional)*: Nombre de tipo de sesión (`'Ordinaria'`, `'Extraordinaria'`, `'Solemne'`).
- `p_numsesion` *(INTEGER, Opcional)*: Número ordinal de la sesión.
- `p_numacuerdo` *(INTEGER, Opcional)*: Número correlativo del acuerdo adoptado.
- `p_idtipodecision` *(INTEGER, Opcional)*: Clave foránea a `tipodecision`.
- `p_nombretipodecision` *(VARCHAR, Opcional)*: Decisión tomada (`'Aprobación'`, `'Promulgación'`, `'Modificación'`, etc.).
- `p_limpiar_sesion` *(BOOLEAN, Default `FALSE`)*: Si es `TRUE`, establece todos los campos de sesión en `NULL` (útil si un documento no proviene de un cuerpo colegiado).

#### Ejemplo
```sql
CALL public.sp_modificar_documento_sesion(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_nombretiposesion => 'Ordinaria',
    p_numsesion => 415,
    p_numacuerdo => 12,
    p_nombretipodecision => 'Aprobación'
);
```

---

### 2.6. `sp_modificar_documento_vigencia`

Actualiza el estado de vigencia jurídica del documento, garantizando la consistencia de los flags `isactive` y de la fecha de `derogacion`.

#### Firma
```sql
CALL public.sp_modificar_documento_vigencia(
    p_iddocumento UUID,
    p_idestadovigencia INTEGER DEFAULT NULL,
    p_nombreestadovigencia VARCHAR DEFAULT NULL,
    p_fecha_derogacion DATE DEFAULT NULL,
    p_isactive BOOLEAN DEFAULT NULL
);
```

#### Parámetros
- `p_iddocumento` *(UUID, Requerido)*: Clave primaria del documento.
- `p_idestadovigencia` *(INTEGER, Opcional)*: Clave foránea a `estadovigencia`.
- `p_nombreestadovigencia` *(VARCHAR, Opcional)*: Nombre del estado (`'Vigente'`, `'Derogado'`, `'Modificado'`, `'No Vigente'`, `'En Revisión'`).
- `p_fecha_derogacion` *(DATE, Opcional)*: Fecha en que pierde vigencia.
- `p_isactive` *(BOOLEAN, Opcional)*: Visibilidad para consultas y recuperación RAG.

#### Reglas de Negocio
- Si el estado pasa a `'Derogado'` o `'No Vigente'` y `p_isactive` no se especifica, se establece en `FALSE`.
- Si el estado pasa a `'Derogado'` y no hay fecha de derogación previa ni explícita, se asigna `CURRENT_DATE`.
- Si el estado pasa a `'Vigente'`, `isactive` se activa (`TRUE`) y la fecha de derogación se limpia a `NULL` (salvo indicación explícita).

#### Ejemplo
```sql
CALL public.sp_modificar_documento_vigencia(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_nombreestadovigencia => 'Derogado',
    p_fecha_derogacion => '2026-03-01'::date
);
```

---

### 2.7. `sp_modificar_documento_fechas`

Actualiza las fechas cronológicas de promulgación/creación y derogación del documento.

#### Firma
```sql
CALL public.sp_modificar_documento_fechas(
    p_iddocumento UUID,
    p_creacion DATE DEFAULT NULL,
    p_derogacion DATE DEFAULT NULL,
    p_limpiar_derogacion BOOLEAN DEFAULT FALSE
);
```

#### Reglas de Negocio
- Valida que `derogacion` no sea anterior a `creacion` (`derogacion >= creacion`).
- Permite limpiar expresamente la fecha de derogación fijando `p_limpiar_derogacion => TRUE`.

#### Ejemplo
```sql
CALL public.sp_modificar_documento_fechas(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_creacion => '2024-02-27'::date,
    p_limpiar_derogacion => TRUE
);
```

---

### 2.8. `sp_modificar_documento_urls3`

Actualiza el enlace URI / clave de almacenamiento del PDF o documento original en el bucket de Amazon S3.

#### Firma
```sql
CALL public.sp_modificar_documento_urls3(
    p_iddocumento UUID,
    p_urlarchivooriginals3 VARCHAR(500)
);
```

#### Ejemplo
```sql
CALL public.sp_modificar_documento_urls3(
    '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    's3://giru-normativas/2024/056_2024_decreto.pdf'
);
```

---

### 2.9. Procedimientos de Tablas Puente N:M (Filtros Multicriterio)

Todos los procedimientos de tablas puente aceptan el parámetro `p_modo` con tres comportamientos posibles:
1. `'REEMPLAZAR'` (Default): Borra las asociaciones actuales del documento y registra las nuevas.
2. `'AGREGAR'`: Agrega únicamente los nuevos registros sin eliminar los ya existentes (`ON CONFLICT DO NOTHING`).
3. `'ELIMINAR'`: Desvincula únicamente los elementos especificados en los arreglos.

#### Resumen de Procedimientos de Filtros:

```
┌──────────────────────────────────────────────┬────────────────────────────┬─────────────────────────────┐
│ Procedimiento                                │ Tabla Puente               │ Catálogo Asociado           │
├──────────────────────────────────────────────┼────────────────────────────┼─────────────────────────────┤
│ sp_modificar_documento_areas_emisoras        │ areaemisora                │ subarea                     │
│ sp_modificar_documento_niveles               │ documentonivel             │ nivelacademico              │
│ sp_modificar_documento_jornadas              │ documentojornada           │ jornada                     │
│ sp_modificar_documento_sedes                 │ documentosedecampus        │ sedecampus                  │
│ sp_modificar_documento_carreras              │ documentocarrera           │ carrera                     │
│ sp_modificar_documento_departamentos         │ documentodepartamento      │ departamento                │
│ sp_modificar_documento_roles                 │ documentorol               │ rolinstitucional            │
│ sp_modificar_documento_beneficios            │ documentobeneficio         │ beneficiointerno            │
│ sp_modificar_documento_nombramientos         │ documentonombramiento      │ tiponombramiento            │
└──────────────────────────────────────────────┴────────────────────────────┴─────────────────────────────┘
```

#### Firmas Representativas

```sql
-- Áreas Emisoras
CALL public.sp_modificar_documento_areas_emisoras(
    p_iddocumento UUID,
    p_ids_subareas INTEGER[] DEFAULT NULL,
    p_nombres_subareas TEXT[] DEFAULT NULL,
    p_modo VARCHAR DEFAULT 'REEMPLAZAR'
);

-- Roles Institucionales
CALL public.sp_modificar_documento_roles(
    p_iddocumento UUID,
    p_ids_roles INTEGER[] DEFAULT NULL,
    p_nombres_roles TEXT[] DEFAULT NULL,
    p_modo VARCHAR DEFAULT 'REEMPLAZAR'
);

-- Beneficios Internos
CALL public.sp_modificar_documento_beneficios(
    p_iddocumento UUID,
    p_ids_beneficios INTEGER[] DEFAULT NULL,
    p_nombres_beneficios TEXT[] DEFAULT NULL,
    p_modo VARCHAR DEFAULT 'REEMPLAZAR'
);

-- Nombramientos
CALL public.sp_modificar_documento_nombramientos(
    p_iddocumento UUID,
    p_ids_nombramientos INTEGER[] DEFAULT NULL,
    p_nombres_nombramientos TEXT[] DEFAULT NULL,
    p_modo VARCHAR DEFAULT 'REEMPLAZAR'
);
```

#### Agrupador Académico: `sp_modificar_documento_filtros_academicos`
Permite modificar en una sola instrucción los 5 filtros del ámbito académico:

```sql
CALL public.sp_modificar_documento_filtros_academicos(
    p_iddocumento => '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_nombres_sedes => ARRAY['Casa Central (Valparaíso)', 'Campus San Joaquín'],
    p_nombres_departamentos => ARRAY['Informática'],
    p_nombres_carreras => ARRAY['Ingeniería Civil Informática'],
    p_nombres_jornadas => ARRAY['Diurno'],
    p_nombres_niveles => ARRAY['Pregrado']
);
```

---

### 2.10. `sp_modificar_o_crear_relacion_documental`

Gestiona los vínculos jurídicos dirigidos entre documentos (`RelacionDocumental`), permitiendo modelar acciones como **Modifica**, **Deroga**, **Complementa** o **Reemplaza**.

#### Firma
```sql
CALL public.sp_modificar_o_crear_relacion_documental(
    p_iddocumentoorigen UUID,
    p_iddocumentodestino UUID,
    p_idtiporelacion INTEGER DEFAULT NULL,
    p_nombretiporelacion VARCHAR DEFAULT NULL,
    p_detallemodificacion VARCHAR(255) DEFAULT NULL,
    p_fechaefecto DATE DEFAULT NULL,
    p_verificada BOOLEAN DEFAULT TRUE,
    p_confianza NUMERIC(3,2) DEFAULT 1.00,
    p_textoevidencia TEXT DEFAULT NULL,
    p_idfuentedeteccion INTEGER DEFAULT NULL,
    p_idrespaldolegal INTEGER DEFAULT NULL,
    p_idrelacion INTEGER DEFAULT NULL
);
```

#### Triggers del Sistema que se Disparan
1. **`trg_relacion_documental_prevencion_ciclos` (`fn_prevenir_ciclos_documentales`):**
   - Ejecuta una consulta recursiva CTE (*Common Table Expression*) sobre el grafo.
   - Si la relación genera un ciclo cerrado directo ($A \to A$) o transitivo ($A \to B \to C \to A$), **aborta inmediatamente la transacción** con una excepción controlada.
2. **`trg_relacion_documental_vigencia` (`fn_sincronizar_vigencia_derogacion`):**
   - Si la relación es de tipo `'Deroga'` y `verificada = TRUE`, actualiza en tiempo real el documento destino:
     - `idEstadoVigencia = 'Derogado'`
     - `derogacion = NEW.fechaEfecto`
     - `isActive = FALSE`

#### Ejemplo
```sql
CALL public.sp_modificar_o_crear_relacion_documental(
    p_iddocumentoorigen => '01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid,
    p_iddocumentodestino => '01a12d3c-994a-7a59-89ef-0667e6e94635'::uuid,
    p_nombretiporelacion => 'Deroga',
    p_detallemodificacion => 'Derogación expresa de los artículos 1 al 20.'
);
```

---

### 2.11. `sp_eliminar_relacion_documental`

Elimina un vínculo del grafo relacional, permitiendo buscarlo por su `idrelacion` numérico o por el par `(origen, destino)`.

#### Firma
```sql
CALL public.sp_eliminar_relacion_documental(
    p_idrelacion INTEGER DEFAULT NULL,
    p_iddocumentoorigen UUID DEFAULT NULL,
    p_iddocumentodestino UUID DEFAULT NULL,
    p_idtiporelacion INTEGER DEFAULT NULL
);
```

---

## 3. Procedimiento Maestro Unificado (SSOT): `sp_modificar_documento_completo`

Constituye la **interfaz principal para backend y microservicios**. Recibe el identificador UUIDv7 y cualquier subconjunto o la totalidad de campos y tablas satélite, aplicando todas las mutaciones en una **única transacción atómica**.

### Firma Completa

```sql
CREATE OR REPLACE PROCEDURE public.sp_modificar_documento_completo(
    p_iddocumento UUID,
    -- Campos escalares
    p_numero VARCHAR(50) DEFAULT NULL,
    p_titulo VARCHAR(255) DEFAULT NULL,
    p_descripcion TEXT DEFAULT NULL,
    p_urlarchivooriginals3 VARCHAR(500) DEFAULT NULL,
    p_idtipodocumento INTEGER DEFAULT NULL,
    p_nombretipodocumento VARCHAR DEFAULT NULL,
    p_idcategoria INTEGER DEFAULT NULL,
    p_nombrecategoria VARCHAR DEFAULT NULL,
    p_numcategoria VARCHAR DEFAULT NULL,
    p_nommetadato VARCHAR DEFAULT NULL,
    p_recalcular_nommetadato BOOLEAN DEFAULT TRUE,
    p_idestadovigencia INTEGER DEFAULT NULL,
    p_nombreestadovigencia VARCHAR DEFAULT NULL,
    p_creacion DATE DEFAULT NULL,
    p_derogacion DATE DEFAULT NULL,
    p_limpiar_derogacion BOOLEAN DEFAULT FALSE,
    p_isactive BOOLEAN DEFAULT NULL,
    p_idtiposesion INTEGER DEFAULT NULL,
    p_nombretiposesion VARCHAR DEFAULT NULL,
    p_numsesion INTEGER DEFAULT NULL,
    p_numacuerdo INTEGER DEFAULT NULL,
    p_idtipodecision INTEGER DEFAULT NULL,
    p_nombretipodecision VARCHAR DEFAULT NULL,
    p_limpiar_sesion BOOLEAN DEFAULT FALSE,
    -- Tablas puente multivalor (N:M)
    p_areas_emisoras_ids INTEGER[] DEFAULT NULL,
    p_areas_emisoras_nombres TEXT[] DEFAULT NULL,
    p_niveles_ids INTEGER[] DEFAULT NULL,
    p_niveles_nombres TEXT[] DEFAULT NULL,
    p_jornadas_ids INTEGER[] DEFAULT NULL,
    p_jornadas_nombres TEXT[] DEFAULT NULL,
    p_sedes_ids INTEGER[] DEFAULT NULL,
    p_sedes_nombres TEXT[] DEFAULT NULL,
    p_carreras_ids INTEGER[] DEFAULT NULL,
    p_carreras_nombres TEXT[] DEFAULT NULL,
    p_departamentos_ids INTEGER[] DEFAULT NULL,
    p_departamentos_nombres TEXT[] DEFAULT NULL,
    p_roles_ids INTEGER[] DEFAULT NULL,
    p_roles_nombres TEXT[] DEFAULT NULL,
    p_beneficios_ids INTEGER[] DEFAULT NULL,
    p_beneficios_nombres TEXT[] DEFAULT NULL,
    p_nombramientos_ids INTEGER[] DEFAULT NULL,
    p_nombramientos_nombres TEXT[] DEFAULT NULL,
    p_modo_puentes VARCHAR DEFAULT 'REEMPLAZAR',
    -- Relación normativa (opcional)
    p_relacion_destino_id UUID DEFAULT NULL,
    p_relacion_tipo_id INTEGER DEFAULT NULL,
    p_relacion_tipo_nombre VARCHAR DEFAULT NULL,
    p_relacion_detalle VARCHAR DEFAULT NULL,
    p_relacion_fecha DATE DEFAULT NULL,
    p_relacion_verificada BOOLEAN DEFAULT TRUE
);
```

---

## 4. Función de Auditoría: `fn_obtener_documento_detalle`

Retorna la representación canónica del documento en un objeto estructurado `JSONB`.

```sql
SELECT public.fn_obtener_documento_detalle('01a12d3c-994a-7a59-89ef-0667e6e94634'::uuid);
```

### Estructura de Salida JSONB
```json
{
  "documento": {
    "iddocumento": "01a12d3c-994a-7a59-89ef-0667e6e94634",
    "numero": "056/2026",
    "titulo": "DECRETO DE RECTORÍA N° 056/2026",
    "nommetadato": "GI-01-08",
    "correlativocategoria": 8,
    "descripcion": "Reglamento general actualizado.",
    "urlarchivooriginals3": "s3://giru-normativas/2026/056_2026.pdf",
    "cant_paginas": 14,
    "creacion": "2026-03-15",
    "derogacion": null,
    "isactive": true,
    "aplicacioninmediata": true,
    "ultima_modificacion": "2026-03-15T18:30:00-03:00",
    "tipodocumento": { "id": 1, "nombre": "Reglamento" },
    "categoria": { "id": 5, "num": "GI-01", "nombre": "Gobernanza normativa" },
    "macrocategoria": { "id": 2, "sigla": "GI", "nombre": "Gestión Institucional" },
    "estadovigencia": { "id": 1, "nombre": "Vigente" },
    "sesion": {
      "idtiposesion": 1,
      "tiposesion": "Ordinaria",
      "numsesion": 415,
      "numacuerdo": 12,
      "idtipodecision": 1,
      "tipodecision": "Aprobación"
    }
  },
  "areas_emisoras": [
    { "id": 1, "nombre": "Secretaría General", "sigla": "SECGEN" }
  ],
  "niveles_academicos": [],
  "jornadas": [{ "id": 1, "nombre": "Diurno" }],
  "sedes_campus": [
    { "id": 1, "nombre": "Casa Central (Valparaíso)" }
  ],
  "carreras": [],
  "departamentos": [
    { "id": 1, "nombre": "Informática" }
  ],
  "roles_institucionales": [
    { "id": 1, "nombre": "Docente" },
    { "id": 2, "nombre": "Alumno" }
  ],
  "beneficios_internos": [],
  "nombramientos": [],
  "relaciones_como_origen": [
    {
      "idrelacion": 42,
      "destino_uuid": "01a12d3c-994a-7a59-89ef-0667e6e94635",
      "destino_numero": "120/2020",
      "tipo_relacion": "Deroga",
      "detalle": "Derogación expresa.",
      "fecha_efecto": "2026-03-15",
      "verificada": true
    }
  ],
  "relaciones_como_destino": []
}
```

---

## 5. Manejo de Excepciones y Errores Comunes

| Código / Excepción | Causa | Solución |
| :--- | :--- | :--- |
| `Integridad: No se encontró el documento con ID %` | El UUID proporcionado no existe en `documento`. | Verificar el UUIDv7 del documento consultado. |
| `Unicidad: El número institucional "%" ya está asignado` | Se intentó asignar un `numero` duplicado. | Utilizar un número unívoco oficial. |
| `Unicidad: nommetadato "%" ya existe` | Colisión con clave única `documento_nommetadato_key`. | Recalcular correlativo o especificar un código manual unívoco. |
| `Catálogo: [...] no existe` | El nombre o ID enviado no existe en la tabla catálogo. | Verificar los nombres contra los catálogos en `config.py` o tablas base. |
| `Violación de Grafo Acíclico (DAG)` | La relación genera un ciclo cerrado ($A \to B \to A$). | Corregir la dirección del vínculo jurídico entre normas. |
| `Validación: derogacion < creacion` | La fecha de derogación es previa a la fecha de creación. | Validar la consistencia cronológica de los decretos. |


---

## 6. Integración y Ejemplo Completo en Python con FastAPI

A continuación se muestra una implementación canónica y lista para producción utilizando **FastAPI** y **AsyncPG** (o SQLAlchemy Async / Psycopg 3). Este patrón consume el procedimiento maestro `sp_modificar_documento_completo` y retorna inmediatamente el estado auditado mediante `fn_obtener_documento_detalle`.

### 6.1. Esquema de Validación (Pydantic v2)

Define el payload parcial de actualización (`PATCH`). Todos los campos son opcionales (`None` por defecto), permitiendo modificar solo los atributos deseados sin pisar el resto.

```python
# schemas/documento.py
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import date
from uuid import UUID

class DocumentoPatchRequest(BaseModel):
    # Campos escalares
    numero: Optional[str] = Field(None, max_length=50, description="Número oficial institucional (ej: '056/2026')")
    titulo: Optional[str] = Field(None, max_length=255, description="Título del documento")
    descripcion: Optional[str] = Field(None, description="Resumen descriptivo del contenido")
    url_archivo_original_s3: Optional[str] = Field(None, max_length=500, description="URI o key en Amazon S3")
    tipo_documento_nombre: Optional[str] = Field(None, description="Ej: 'Decreto', 'Reglamento', 'Resolución'")
    categoria_num: Optional[str] = Field(None, description="Código de categoría (ej: 'GI-01', 'DC-02')")
    nom_metadato: Optional[str] = Field(None, description="Override manual para nommetadato si aplica")
    estado_vigencia_nombre: Optional[str] = Field(None, description="Ej: 'Vigente', 'Derogado', 'Modificado'")
    creacion: Optional[date] = Field(None, description="Fecha de promulgación o creación (YYYY-MM-DD)")
    derogacion: Optional[date] = Field(None, description="Fecha de derogación oficial (YYYY-MM-DD)")
    limpiar_derogacion: bool = Field(False, description="Fijar en TRUE si vuelve a estar vigente para limpiar fecha")
    is_active: Optional[bool] = Field(None, description="Visibilidad RAG")

    # Sesión y cuerpos colegiados
    tipo_sesion_nombre: Optional[str] = Field(None, description="Ej: 'Ordinaria', 'Extraordinaria'")
    num_sesion: Optional[int] = Field(None, ge=1, description="Número ordinal de sesión")
    num_acuerdo: Optional[int] = Field(None, ge=1, description="Número ordinal de acuerdo")
    tipo_decision_nombre: Optional[str] = Field(None, description="Ej: 'Aprobación', 'Promulgación'")
    limpiar_sesion: bool = Field(False, description="Fijar en TRUE para remover datos de sesión")

    # Tablas puente N:M (nombres canónicos)
    areas_emisoras: Optional[List[str]] = Field(None, description="Nombres de subáreas (ej: ['Rectoría', 'Secretaría General'])")
    sedes_campus: Optional[List[str]] = Field(None, description="Nombres de sedes (ej: ['Casa Central (Valparaíso)'])")
    departamentos: Optional[List[str]] = Field(None, description="Departamentos (ej: ['Informática'])")
    carreras: Optional[List[str]] = Field(None, description="Nombres o códigos de carreras")
    roles: Optional[List[str]] = Field(None, description="Roles regulados (ej: ['Docente', 'Alumno'])")
    jornadas: Optional[List[str]] = Field(None, description="Jornadas académicas (ej: ['Diurno'])")
    niveles: Optional[List[str]] = Field(None, description="Niveles académicos (ej: ['Pregrado', 'Postgrado'])")
    beneficios: Optional[List[str]] = Field(None, description="Beneficios internos asociados")
    nombramientos: Optional[List[str]] = Field(None, description="Cargos o nombramientos regulados")
    modo_puentes: str = Field("REEMPLAZAR", pattern="^(REEMPLAZAR|AGREGAR|ELIMINAR)$")

    # Relación jurídica opcional
    relacion_destino_id: Optional[UUID] = Field(None, description="UUID del documento afectado")
    relacion_tipo_nombre: Optional[str] = Field(None, description="Ej: 'Deroga', 'Modifica', 'Complementa'")
    relacion_detalle: Optional[str] = Field(None, description="Detalle del cambio articulado")
    relacion_fecha: Optional[date] = Field(None, description="Fecha de efecto del cambio")
```

---

### 6.2. Controlador / Endpoint de API (FastAPI)

Invoca atómicamente el procedimiento almacenado y captura las excepciones de PostgreSQL mapeándolas a códigos HTTP estándar (`400`, `404`, `409`, `422`).

```python
# routers/documentos.py
import json
import asyncpg
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, status
from schemas.documento import DocumentoPatchRequest
from db.connection import get_db_pool  # Dependencia del pool asyncpg

router = APIRouter(prefix="/api/v1/documentos", tags=["Documentos Normativos"])

@router.patch(
    "/{documento_id}",
    summary="Actualizar documento normativo de forma integral",
    response_description="Detalle canónico del documento modificado en formato JSONB"
)
async def actualizar_documento(
    documento_id: UUID,
    payload: DocumentoPatchRequest,
    pool: asyncpg.Pool = Depends(get_db_pool)
):
    """
    Actualiza atómicamente un documento normativo identificado por su UUIDv7.
    Consume el procedimiento almacenado canónico `sp_modificar_documento_completo`
    y retorna la estructura consolidada mediante `fn_obtener_documento_detalle`.
    """
    query_call_sp = """
        CALL public.sp_modificar_documento_completo(
            p_iddocumento => $1,
            p_numero => $2,
            p_titulo => $3,
            p_descripcion => $4,
            p_urlarchivooriginals3 => $5,
            p_nombretipodocumento => $6,
            p_numcategoria => $7,
            p_nommetadato => $8,
            p_nombreestadovigencia => $9,
            p_creacion => $10,
            p_derogacion => $11,
            p_limpiar_derogacion => $12,
            p_isactive => $13,
            p_nombretiposesion => $14,
            p_numsesion => $15,
            p_numacuerdo => $16,
            p_nombretipodecision => $17,
            p_limpiar_sesion => $18,
            p_areas_emisoras_nombres => $19,
            p_niveles_nombres => $20,
            p_jornadas_nombres => $21,
            p_sedes_nombres => $22,
            p_carreras_nombres => $23,
            p_departamentos_nombres => $24,
            p_roles_nombres => $25,
            p_beneficios_nombres => $26,
            p_nombramientos_nombres => $27,
            p_modo_puentes => $28,
            p_relacion_destino_id => $29,
            p_relacion_tipo_nombre => $30,
            p_relacion_detalle => $31,
            p_relacion_fecha => $32
        );
    """

    query_get_detail = "SELECT public.fn_obtener_documento_detalle($1) AS detalle;"

    async with pool.acquire() as conn:
        async with conn.transaction():
            try:
                # 1. Ejecutar el procedimiento maestro
                await conn.execute(
                    query_call_sp,
                    documento_id,
                    payload.numero,
                    payload.titulo,
                    payload.descripcion,
                    payload.url_archivo_original_s3,
                    payload.tipo_documento_nombre,
                    payload.categoria_num,
                    payload.nom_metadato,
                    payload.estado_vigencia_nombre,
                    payload.creacion,
                    payload.derogacion,
                    payload.limpiar_derogacion,
                    payload.is_active,
                    payload.tipo_sesion_nombre,
                    payload.num_sesion,
                    payload.num_acuerdo,
                    payload.tipo_decision_nombre,
                    payload.limpiar_sesion,
                    payload.areas_emisoras,
                    payload.niveles,
                    payload.jornadas,
                    payload.sedes_campus,
                    payload.carreras,
                    payload.departamentos,
                    payload.roles,
                    payload.beneficios,
                    payload.nombramientos,
                    payload.modo_puentes,
                    payload.relacion_destino_id,
                    payload.relacion_tipo_nombre,
                    payload.relacion_detalle,
                    payload.relacion_fecha
                )

                # 2. Consultar el estado resultante consolidado en JSONB
                row = await conn.fetchrow(query_get_detail, documento_id)
                if not row or not row["detalle"]:
                    raise HTTPException(
                        status_code=status.HTTP_404_NOT_FOUND,
                        detail=f"Documento con ID {documento_id} no encontrado."
                    )

                # asyncpg parsea columnas json/jsonb como string o dict
                detalle = row["detalle"]
                return json.loads(detalle) if isinstance(detalle, str) else detalle

            except asyncpg.exceptions.RaiseException as ex:
                msg = str(ex)
                # Mapeo semántico de excepciones según reglas de negocio
                if "No se encontró el documento" in msg:
                    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=msg)
                elif "Unicidad:" in msg:
                    raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail=msg)
                elif "Violación de Grafo Acíclico (DAG)" in msg:
                    raise HTTPException(status_code=status.HTTP_422_UNPROCESSABLE_ENTITY, detail=msg)
                elif "Catálogo:" in msg or "Validación:" in msg:
                    raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=msg)
                else:
                    raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=msg)

            except Exception as e:
                raise HTTPException(
                    status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                    detail=f"Error inesperado al procesar la actualización: {str(e)}"
                )
```

---

### 6.3. Ejemplo de Solicitud HTTP (`cURL`) y Respuesta

#### Petición `PATCH`
```bash
curl -X 'PATCH' \
  'http://localhost:8000/api/v1/documentos/01a12d3c-994a-7a59-89ef-0667e6e94634' \
  -H 'Content-Type: application/json' \
  -d '{
    "numero": "056/2026",
    "titulo": "DECRETO DE RECTORÍA N° 056/2026 - Reglamento Actualizado",
    "descripcion": "Reglamento de régimen académico y disciplina estudiantil.",
    "tipo_documento_nombre": "Decreto",
    "categoria_num": "GI-01",
    "estado_vigencia_nombre": "Vigente",
    "tipo_sesion_nombre": "Ordinaria",
    "num_sesion": 415,
    "areas_emisoras": ["Rectoría", "Secretaría General"],
    "sedes_campus": ["Casa Central (Valparaíso)", "Campus San Joaquín"],
    "departamentos": ["Informática"],
    "roles": ["Docente", "Alumno"]
  }'
```

#### Respuesta `200 OK`
```json
{
  "documento": {
    "iddocumento": "01a12d3c-994a-7a59-89ef-0667e6e94634",
    "numero": "056/2026",
    "titulo": "DECRETO DE RECTORÍA N° 056/2026 - Reglamento Actualizado",
    "nommetadato": "GI-01-08",
    "correlativocategoria": 8,
    "descripcion": "Reglamento de régimen académico y disciplina estudiantil.",
    "urlarchivooriginals3": "s3://giru-normativas/2026/056_2026_reglamento.pdf",
    "cant_paginas": 12,
    "creacion": "2026-03-15",
    "derogacion": null,
    "isactive": true,
    "aplicacioninmediata": true,
    "ultima_modificacion": "2026-03-15T18:30:00-03:00",
    "tipodocumento": { "id": 2, "nombre": "Decreto" },
    "categoria": { "id": 5, "num": "GI-01", "nombre": "Gobernanza normativa" },
    "macrocategoria": { "id": 2, "sigla": "GI", "nombre": "Gestión Institucional" },
    "estadovigencia": { "id": 1, "nombre": "Vigente" },
    "sesion": {
      "idtiposesion": 1,
      "tiposesion": "Ordinaria",
      "numsesion": 415,
      "numacuerdo": null,
      "idtipodecision": null,
      "tipodecision": null
    }
  },
  "areas_emisoras": [
    { "id": 1, "nombre": "Rectoría", "sigla": "REC" },
    { "id": 2, "nombre": "Secretaría General", "sigla": "SECGEN" }
  ],
  "sedes_campus": [
    { "id": 1, "nombre": "Casa Central (Valparaíso)" },
    { "id": 2, "nombre": "Campus San Joaquín" }
  ],
  "departamentos": [
    { "id": 1, "nombre": "Informática" }
  ],
  "roles_institucionales": [
    { "id": 1, "nombre": "Docente" },
    { "id": 2, "nombre": "Alumno" }
  ]
}
```
