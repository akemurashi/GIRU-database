# Documentación G.I.R.U. Database
## Universidad Técnica Federico Santa María (UTFSM)

Este directorio centraliza los manuales técnicos, diagramas y especificaciones de la base de datos institucional.

---

## 📑 Índice de Recursos

1. [**Procedimientos Almacenados (`PROCEDIMIENTOS_ALMACENADOS.md`)**](PROCEDIMIENTOS_ALMACENADOS.md):
   * Especificación exhaustiva de firmas, tipos de parámetros, transaccionalidad ACID y ejemplos de ejecución para los procedimientos modulares y maestros (`sp_modificar_documento.sql`).
   * Trazabilidad temporal mediante `ultima_modificacion TIMESTAMPTZ` con `CURRENT_TIMESTAMP`.
   * Estructura JSONB retornada por la función de alto rendimiento `fn_obtener_documento_detalle`.

2. [**Diagrama de Entidad-Relación (`DB_optv1_clear.drawio.svg`)**](DB_optv1_clear.drawio.svg):
   * Visualización vectorial del modelo relacional en Tercera Forma Normal (3NF), incluyendo las 33 tablas, claves foráneas, restricciones de integridad y las 9 tablas puente N:M.

3. [**Manual Consolidado de Base de Datos (`documentacionDB.pdf`)**](documentacionDB.pdf):
   * Documento formal con la arquitectura global de datos del proyecto G.I.R.U.

4. [**Tabla Satélite de Trazabilidad (`TABLA_DOCUMENTOTRAZABILIDAD.md`)**](TABLA_DOCUMENTOTRAZABILIDAD.md):
   * Diccionario de datos, definición DDL, columnas generadas (`cant_antes`, `cant_despues`) y estrategia de índices GIN (`jsonb_path_ops`).
   * Almacenamiento de normas históricas externas con `nombre_archivo: null` y reglas de integridad referencial en cascada.

5. [**Guía de Integración Backend (`TRAZABILIDAD_API_BACKEND.md`)**](TRAZABILIDAD_API_BACKEND.md):
   * Contrato del payload JSON de la línea de tiempo (`antes[]` y `despues[]`).
   * Especificación de endpoints ligeros (`fn_obtener_trazabilidad_documento`) y procedimientos administrativos (`sp_modificar_o_crear_trazabilidad`).
   * Ejemplos de implementación listos para copiar y pegar en Node.js (TypeScript) y Python (FastAPI).

