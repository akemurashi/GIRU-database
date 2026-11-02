# Documentación Técnica: Tabla Satélite `public.documentotrazabilidad`

**Proyecto:** G.I.R.U. (Gestión Inteligente de Regulaciones Universitarias)  
**Institución:** Universidad Técnica Federico Santa María (UTFSM)  
**Motor de Base de Datos:** PostgreSQL 16 (AWS RDS) / PostgreSQL 18 (Local Devbox)  
**Patrón Arquitectónico:** Extensión Satélite 1:1 / Read Model CQRS  
**Fecha de Implementación:** Noviembre de 2026  
**Script DDL Oficial:** [`data/migracion_documento_trazabilidad.sql`](../data/migracion_documento_trazabilidad.sql)  
**Script ETL Asociado:** [`script/actualizar_trazabilidad.py`](../script/actualizar_trazabilidad.py)

---

## 1. Propósito y Justificación de Arquitectura

La tabla `public.documentotrazabilidad` almacena la **proyección genealógica pre-calculada (Clausura Transitiva)** de cada documento normativo del corpus universitario.

### ¿Por qué existe como tabla satélite separada?
1. **Preservación del Núcleo en 3NF (SSOT):** La tabla maestra `public.documento` y el grafo relacional `public.relaciondocumental` gestionan exclusivamente la verdad transaccional y las dependencias causales directas verificadas (Reducción Transitiva: ~96 aristas).
2. **Soporte Nativo de Normas Históricas Externas:** Permite registrar los **59 decretos históricos reales no digitalizados** (`nombre_archivo: null`, 165 menciones) sin violar las restricciones de clave foránea `UUID NOT NULL` de la tabla relacional.
3. **Rendimiento de Consulta para la Web:** Entrega la línea de tiempo completa a la interfaz de usuario en **menos de 1 milisegundo**, eliminando la necesidad de costosas consultas recursivas en tiempo de ejecución.
4. **Ciclo de Vida Desacoplado:** Puede vaciarse, reconstruirse o sincronizarse incrementalmente desde el ETL sin bloquear la tabla `documento` ni alterar sus marcas temporales maestras.

---

## 2. Definición DDL Oficial

```sql
CREATE TABLE public.documentotrazabilidad (
    iddocumento UUID NOT NULL,
    antes JSONB DEFAULT '[]'::jsonb NOT NULL,
    despues JSONB DEFAULT '[]'::jsonb NOT NULL,
    cant_antes INTEGER GENERATED ALWAYS AS (jsonb_array_length(antes)) STORED,
    cant_despues INTEGER GENERATED ALWAYS AS (jsonb_array_length(despues)) STORED,
    ultima_sincronizacion TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT documentotrazabilidad_pkey PRIMARY KEY (iddocumento),
    CONSTRAINT fk_documentotrazabilidad_documento 
        FOREIGN KEY (iddocumento) 
        REFERENCES public.documento(iddocumento) 
        ON DELETE CASCADE
);
```

---

## 3. Diccionario de Datos (Data Dictionary)

| Columna | Tipo de Dato | Nulable | Valor por Defecto | Restricciones / Reglas | Descripción |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **`iddocumento`** | `UUID` | **NO** | Ninguno | `PRIMARY KEY`, `FK -> public.documento(iddocumento) ON DELETE CASCADE` | Identificador único del documento maestro al que pertenece la trazabilidad. |
| **`antes`** | `JSONB` | **NO** | `'[]'::jsonb` | Arreglo JSONB validado | Lista ordenada de hitos normativos que antecedieron, modificaron o precedieron a esta norma hacia atrás en el tiempo. |
| **`despues`** | `JSONB` | **NO** | `'[]'::jsonb` | Arreglo JSONB validado | Lista ordenada de hitos normativos que sucedieron, reformaron o derogaron a esta norma hacia adelante en el tiempo. |
| **`cant_antes`** | `INTEGER` | **SÍ** | Computado | `GENERATED ALWAYS AS (jsonb_array_length(antes)) STORED` | Cantidad total de hitos en `antes`. Columna generada automáticamente por el motor y persistida en disco para búsquedas instantáneas. |
| **`cant_despues`**| `INTEGER` | **SÍ** | Computado | `GENERATED ALWAYS AS (jsonb_array_length(despues)) STORED`| Cantidad total de hitos en `despues`. Columna generada y persistida en disco. |
| **`ultima_sincronizacion`** | `TIMESTAMPTZ` | **SÍ** | `CURRENT_TIMESTAMP` | Marca temporal con zona horaria | Fecha y hora en la que el proceso ETL o el procedimiento almacenado actualizó la línea de tiempo por última vez. |

---

## 4. Estructura Interna de los Arreglos JSONB

Cada elemento contenido dentro de los arreglos `antes` y `despues` sigue estrictamente la siguiente firma canónica:

```typescript
interface HitoNormativo {
  decreto: string;                 // Código canónico del decreto (ej. "DECRETO 053/1992")
  fecha: string | null;            // Fecha de promulgación/efecto en formato ISO YYYY-MM-DD
  anio: number | null;             // Año del hito (ej. 1992)
  nombre_archivo: string | null;   // Nombre del archivo Markdown digitalizado o null
}
```

### Ejemplo de Fila en PostgreSQL:
```json
{
  "iddocumento": "01a13ca5-a64b-76e0-b638-06e4b74b3f5f",
  "cant_antes": 2,
  "cant_despues": 1,
  "ultima_sincronizacion": "2026-11-02T22:05:55.079949+00:00",
  "antes": [
    {
      "decreto": "DECRETO 053/1992",
      "fecha": "1992-11-05",
      "anio": 1992,
      "nombre_archivo": "053.1992 DECRETO - Promulga instructivo sobre Contraloria.md"
    },
    {
      "decreto": "DECRETO 005/1995",
      "fecha": "1995-03-09",
      "anio": 1995,
      "nombre_archivo": "Decreto_05-95 ROAS.md"
    }
  ],
  "despues": [
    {
      "decreto": "DECRETO 057/2025",
      "fecha": "2025-01-20",
      "anio": 2025,
      "nombre_archivo": null
    }
  ]
}
```

### Regla Semántica de `nombre_archivo: null`
* Si `nombre_archivo` es un **`string`**: La norma está digitalizada en el corpus. El frontend puede generar un hipervínculo para abrir el PDF o la ficha del documento.
* Si `nombre_archivo` es **`null`**: La norma existe formalmente pero **no está digitalizada**. El frontend debe mostrarla como un hito informativo o badge desactivando el enlace de descarga para evitar errores HTTP 404.

---

## 5. Estrategia de Índices y Rendimiento en Disco

Para garantizar tiempos de respuesta sub-milisegundo tanto en búsquedas escalares como en operaciones JSONB avanzadas, la tabla cuenta con los siguientes índices especializados:

| Nombre del Índice | Tipo | Columnas / Operador | Propósito |
| :--- | :--- | :--- | :--- |
| **`documentotrazabilidad_pkey`** | B-Tree (Único) | `iddocumento` | Búsqueda por clave primaria; utilizado por los `JOIN` de las funciones RPC. |
| **`idx_doctraza_cantidades`** | B-Tree Compuesto | `(cant_antes, cant_despues)` | Acelera auditorías y filtros de frontend (ej. *"mostrar reglamentos con más de 5 modificaciones"*). |
| **`idx_doctraza_gin_antes`** | GIN | `antes jsonb_path_ops` | Búsqueda rápida por contención JSONB (`@>`) dentro del historial de antecedentes. |
| **`idx_doctraza_gin_despues`** | GIN | `despues jsonb_path_ops` | Búsqueda rápida por contención JSONB (`@>`) dentro del historial de sucesores. |

### Almacenamiento TOAST
Las columnas `antes` y `despues` utilizan la estrategia de almacenamiento estándar de PostgreSQL para JSONB (`EXTENDED`). Si el arreglo supera los 2 KB, PostgreSQL lo comprime de forma transparente en el espacio TOAST, evitando el sobrecosto de I/O en lecturas de la tabla base.

---

## 6. Procedimientos y Funciones Asociadas

### 6.1. Funciones de Lectura (Queries)
1. **`public.fn_obtener_documento_detalle(uuid)`**:
   * Incorpora el nodo `'trazabilidad'` dentro de la respuesta global consolidada de la ficha normativa (junto a las 9 tablas puente y las relaciones del grafo).
2. **`public.fn_obtener_trazabilidad_documento(uuid)`**:
   * Diseñada específicamente para endpoints de lazy-loading (`GET /api/documentos/:id/trazabilidad`).
   * Retorna exclusivamente: `iddocumento`, `numero`, `titulo`, `nommetadato`, `estadovigencia` y el objeto `trazabilidad`.

### 6.2. Procedimientos de Modificación (Mutaciones)
1. **`public.sp_modificar_o_crear_trazabilidad(p_iddocumento, p_antes, p_despues)`**:
   * Inserta o actualiza atómicamente la trazabilidad.
   * Valida previamente que el `iddocumento` exista en `public.documento`.
   * Realiza un `UPSERT` seguro refrescando `ultima_sincronizacion`.
2. **`public.sp_eliminar_trazabilidad_documento(p_iddocumento)`**:
   * Elimina la fila de trazabilidad de un documento específico sin tocar la tabla maestra `public.documento`.

---

## 7. Consultas Útiles de Auditoría y Administración

### 7.1. Verificar cobertura del corpus (debe retornar 381)
```sql
SELECT COUNT(*) AS total_trazabilidad FROM public.documentotrazabilidad;
```

### 7.2. Listar documentos con más de 5 sucesores históricos (normas muy reformadas)
```sql
SELECT 
    d.numero,
    d.titulo,
    dt.cant_despues AS total_reformas,
    dt.ultima_sincronizacion
FROM public.documentotrazabilidad dt
JOIN public.documento d ON dt.iddocumento = d.iddocumento
WHERE dt.cant_despues >= 5
ORDER BY dt.cant_despues DESC;
```

### 7.3. Buscar qué documentos mencionan a una norma externa específica (ej. Decreto 057/2025)
Aprovecha el índice GIN `jsonb_path_ops` de forma ultra-rápida:
```sql
SELECT 
    d.numero, 
    d.titulo, 
    d.nommetadato
FROM public.documentotrazabilidad dt
JOIN public.documento d ON dt.iddocumento = d.iddocumento
WHERE dt.despues @> '[{"decreto": "DECRETO 057/2025"}]';
```

### 7.4. Verificar consistencia de eliminación en cascada
Garantiza que no existan registros huérfanos:
```sql
SELECT COUNT(*) AS huerfanos
FROM public.documentotrazabilidad dt
LEFT JOIN public.documento d ON dt.iddocumento = d.iddocumento
WHERE d.iddocumento IS NULL; -- Debe retornar estrictamente 0
```
