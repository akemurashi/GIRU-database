# Guía de Integración Backend: API de Trazabilidad Normativa (Read Model CQRS)

**Proyecto:** G.I.R.U. (Gestión Inteligente de Regulaciones Universitarias)  
**Institución:** Universidad Técnica Federico Santa María (UTFSM)  
**Destinatarios:** Desarrolladores Backend, Frontend y Arquitectura de Software  
**Motor de Base de Datos:** PostgreSQL 16 (AWS RDS) / PostgreSQL 18 (Local)  
**Entidad Satélite:** `public.documentotrazabilidad`  
**Identificador Canónico:** UUIDv7 (`iddocumento`, RFC 9562)

---

## 1. Visión General y Propósito del Módulo

El módulo de trazabilidad normativa proporciona a la interfaz de usuario una **línea de tiempo genealógica desnormalizada (Clausura Transitiva)** de cada reglamento o decreto universitario.

A diferencia del núcleo transaccional en 3NF (`public.relaciondocumental`), que modela dependencias causales directas y rígidas entre documentos existentes en el corpus, `public.documentotrazabilidad` es una **proyección de lectura (Read Model CQRS)** precalculada que:
1. Permite visualizar qué normas estuvieron **antes** y qué normas vinieron **después** a lo largo de décadas.
2. Soporta nativamente normas promulgadas históricas que **no están digitalizadas en el corpus** (`nombre_archivo: null`).
3. Responde a la API web en **menos de 1.2 milisegundos** sin sobrecargar el motor con recursiones en tiempo de ejecución.

---

## 2. Contrato de Datos (Payload JSON de Trazabilidad)

Cada documento normativo expone el siguiente nodo JSON:

```json
{
  "trazabilidad": {
    "cant_antes": 5,
    "cant_despues": 9,
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
        "decreto": "DECRETO 362/2024",
        "fecha": "2024-08-19",
        "anio": 2024,
        "nombre_archivo": "362.2024 DECRETO - Promulga modificac ROAS_VF -. derogado DR057.2025.md"
      },
      {
        "decreto": "DECRETO 057/2025",
        "fecha": "2025-01-20",
        "anio": 2025,
        "nombre_archivo": null
      }
    ]
  }
}
```

### Reglas de Negocio Esenciales para Backend y Frontend
1. **Manejo de `nombre_archivo: null`**:
   - Significa que la norma existe institucionalmente pero **no tiene archivo Markdown/PDF digitalizado en el bucket S3**.
   - **Frontend**: Debe renderizar el hito en la línea de tiempo como un badge/etiqueta informativa o texto plano, **deshabilitando el botón de descarga o enlace a la ficha**.
   - **Backend**: No debe intentar resolver rutas en AWS S3 para objetos con `nombre_archivo === null`.
2. **Orden Cronológico**:
   - `antes[]`: Ordenado cronológicamente desde la norma más antigua a la más reciente que antecedió al documento.
   - `despues[]`: Ordenado cronológicamente desde la primera modificación hasta la más reciente.
3. **Documentos sin Historial**:
   - Si un documento nunca ha sido modificado ni modificó a otro, el nodo retorna arrays vacíos `[]` y contadores `cant_antes: 0`, `cant_despues: 0`.

---

## 3. Catálogo de Procedimientos y Funciones RPC

Para facilitar la construcción de endpoints REST o GraphQL, la base de datos expone tres funciones/procedimientos nativos:

---

### 3.1. `public.fn_obtener_documento_detalle(uuid)`
* **Propósito**: Retorna la ficha maestra completa del documento: metadatos, las 9 tablas puente satélite, relaciones del grafo DAG y el nodo de trazabilidad.
* **Uso en Backend**: Endpoint `GET /api/documentos/:id`
* **Tipo**: Función `STABLE` (Solo lectura, compatible con transacciones de réplica).
* **Invocación SQL**:
  ```sql
  SELECT public.fn_obtener_documento_detalle('01a13ca5-a64b-76e0-b638-06e4b74b3f5f'::uuid);
  ```

---

### 3.2. `public.fn_obtener_trazabilidad_documento(uuid)`
* **Propósito**: Retorna **únicamente** la línea de tiempo y metadatos clave (`numero`, `titulo`, `nommetadato`, `estadovigencia`, `trazabilidad`).
* **Uso en Backend**: Endpoint ligero `GET /api/documentos/:id/trazabilidad` (ideal para *lazy-loading* en pestañas del frontend o modales).
* **Tipo**: Función `STABLE`.
* **Invocación SQL**:
  ```sql
  SELECT public.fn_obtener_trazabilidad_documento('01a13ca5-a64b-76e0-b638-06e4b74b3f5f'::uuid);
  ```

---

### 3.3. `public.sp_modificar_o_crear_trazabilidad(uuid, jsonb, jsonb)`
* **Propósito**: Inserta o actualiza atómicamente la proyección de trazabilidad de un documento.
* **Uso en Backend**: Endpoint administrativo `PUT /api/documentos/:id/trazabilidad`
* **Firma**:
  ```sql
  CALL public.sp_modificar_o_crear_trazabilidad(
      p_iddocumento UUID,
      p_antes JSONB DEFAULT '[]'::jsonb,
      p_despues JSONB DEFAULT '[]'::jsonb
  );
  ```
* **Comportamiento**:
  - Si el documento no existe en `public.documento`, aborta con excepción SQL `23503`.
  - Si ya existe en `public.documentotrazabilidad`, ejecuta `UPDATE` sobre `antes`, `despues` y refresca `ultima_sincronizacion`.

---

### 3.4. `public.sp_eliminar_trazabilidad_documento(uuid)`
* **Propósito**: Limpia o resetea el historial de trazabilidad de un documento sin alterar la norma maestra.
* **Uso en Backend**: Endpoint administrativo `DELETE /api/documentos/:id/trazabilidad`
* **Invocación SQL**:
  ```sql
  CALL public.sp_eliminar_trazabilidad_documento('01a13ca5-a64b-76e0-b638-06e4b74b3f5f'::uuid);
  ```

---

## 4. Ejemplos de Implementación en Código de Backend

### Ejemplo A: Node.js / TypeScript (Express + `pg`)

```typescript
import { Router, Request, Response } from 'express';
import { Pool } from 'pg';

const router = Router();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// GET /api/documentos/:id/trazabilidad
router.get('/:id/trazabilidad', async (req: Request, res: Response) => {
  const { id } = req.params;
  try {
    const query = 'SELECT public.fn_obtener_trazabilidad_documento($1::uuid) AS data;';
    const result = await pool.query(query, [id]);

    if (!result.rows[0]?.data) {
      return res.status(404).json({ error: 'Documento no encontrado' });
    }

    return res.json(result.rows[0].data);
  } catch (err: any) {
    console.error('Error al consultar trazabilidad:', err);
    return res.status(500).json({ error: 'Error interno del servidor' });
  }
});

// PUT /api/documentos/:id/trazabilidad (Panel de Administración)
router.put('/:id/trazabilidad', async (req: Request, res: Response) => {
  const { id } = req.params;
  const { antes, despues } = req.body;

  try {
    const query = 'CALL public.sp_modificar_o_crear_trazabilidad($1::uuid, $2::jsonb, $3::jsonb);';
    await pool.query(query, [id, JSON.stringify(antes || []), JSON.stringify(despues || [])]);

    return res.json({ message: 'Trazabilidad actualizada exitosamente' });
  } catch (err: any) {
    if (err.code === '23503') {
      return res.status(404).json({ error: 'El documento especificado no existe' });
    }
    return res.status(500).json({ error: 'Error al actualizar trazabilidad' });
  }
});

export default router;
```

---

### Ejemplo B: Python (FastAPI + `asyncpg` / `psycopg`)

```python
from fastapi import APIRouter, HTTPException, Depends
from uuid import UUID
from pydantic import BaseModel
from typing import List, Optional

router = APIRouter(prefix="/api/documentos", tags=["Trazabilidad"])

class HitoTrazabilidad(BaseModel):
    decreto: str
    fecha: Optional[str] = None
    anio: Optional[int] = None
    nombre_archivo: Optional[str] = None

class TrazabilidadPayload(BaseModel):
    antes: List[HitoTrazabilidad] = []
    despues: List[HitoTrazabilidad] = []

@router.get("/{documento_id}/trazabilidad")
async def get_trazabilidad(documento_id: UUID, db = Depends(get_db_conn)):
    query = "SELECT public.fn_obtener_trazabilidad_documento(%s::uuid);"
    async with db.cursor() as cur:
        await cur.execute(query, (str(documento_id),))
        row = await cur.fetchone()
        if not row or not row[0]:
            raise HTTPException(status_code=404, detail="Documento no encontrado")
        return row[0]
```

---

## 5. Pruebas y Rendimiento

La función `fn_obtener_trazabilidad_documento` utiliza un `LEFT JOIN` indexado por clave primaria B-Tree sobre `iddocumento`:
- **Tiempo medio de respuesta en RDS (us-east-1)**: `~0.8 ms` a `1.2 ms`.
- **Uso de memoria**: Prácticamente nulo en backend; la serialización JSONB se realiza de forma nativa en el motor de PostgreSQL.
