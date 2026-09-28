# Arquitectura de la solución

Fecha de inicio: 2026-09-28  
Estado: línea base documental; los procesos de negocio descritos como objetivo todavía no están implementados.

## 1. Propósito y alcance

Este documento describe la arquitectura y los procesos de la solución de stock, recepciones, fabricación y remitos, con preparación para una futura integración con ARCA. Se actualizará a medida que las decisiones se confirmen y el sistema evolucione.

La memoria [`MEMORIA_REUTILIZABLE_ARCA_STOCK.md`](MEMORIA_REUTILIZABLE_ARCA_STOCK.md) conserva aprendizajes y criterios técnicos reutilizables. El [manual de usuario](../manual_usuario/README.md) explica únicamente las funciones disponibles y verificadas.

## 2. Estado actual verificado

- La aplicación usa Laravel 12 y PHP 8.2 o superior.
- La ruta web `/` muestra la vista de bienvenida de Laravel.
- En esta línea base no se encontraron rutas ni flujos de negocio de stock, compras, fabricación, remitos o facturación.
- Las migraciones presentes corresponden al esqueleto inicial de Laravel (usuarios, caché y trabajos).

Por lo tanto, las secciones de procesos objetivo son **Propuesta**, no describen funciones disponibles.

## 3. Principios de diseño

- Mantener el historial de stock mediante movimientos auditables; el saldo es una lectura derivada o de consulta, no un reemplazo del historial.
- Confirmar las operaciones de negocio y sus movimientos asociados dentro de transacciones de base de datos.
- Diseñar confirmaciones para que sean idempotentes y no dupliquen efectos ante reintentos.
- Conservar trazabilidad de usuario, fecha, documento de origen y motivo en las operaciones relevantes.
- Separar documentos de inventario y logística de los comprobantes fiscales.
- Ejecutar integraciones externas fuera de transacciones SQL largas y guardar tanto el resultado como los intentos.
- Proteger secretos y credenciales fuera del repositorio y de la carpeta pública.

## 4. Componentes lógicos propuestos

| Componente | Responsabilidad | Estado |
|---|---|---|
| Catálogos | Productos, insumos, unidades de medida y depósitos | Propuesta |
| Inventario | Movimientos, saldos, ajustes y auditoría | Propuesta |
| Compras y recepción | Órdenes, recepciones parciales y anulaciones | Propuesta |
| Recetas y fabricación | Versiones de recetas, órdenes, consumos y producción | Propuesta |
| Remitos | Documentos de entrada/salida y sus detalles | Propuesta |
| Usuarios y permisos | Identidad, autorización y actor de cada operación | Propuesta |
| Integración ARCA futura | Autenticación, emisión, estados y reintentos | Preparación futura; no implementada |

La organización concreta en servicios, modelos, controladores, colas y tablas se definirá junto con cada módulo, siguiendo las convenciones existentes en Laravel.

## 5. Procesos de negocio objetivo

### 5.1 Recepción de mercadería

**Propuesta.** Registrar una orden, recibir cantidades totales o parciales y confirmar cada recepción. La confirmación actualiza cantidades acumuladas y genera movimientos de stock en una operación consistente. Una anulación conserva el documento y genera el movimiento inverso correspondiente; no borra el historial.

### 5.2 Ajuste e inventario

**Propuesta.** Registrar el motivo, el usuario y las cantidades anterior/nueva. Cada ajuste genera un movimiento auditable; no se modifica silenciosamente el saldo.

### 5.3 Fabricación

**Propuesta.** Tomar una receta versionada, validar disponibilidad de componentes y confirmar consumos e ingresos de producto terminado de forma atómica. Una misma orden no puede generar movimientos duplicados.

### 5.4 Remito de salida

**Propuesta.** Registrar encabezado y detalle como documento logístico independiente de la factura y de la política de precios. La confirmación de salida debe reflejarse en el historial de stock.

### 5.5 Facturación electrónica ARCA

**Futura.** Persistir el comprobante pendiente, procesar la llamada externa mediante un servicio/job y guardar respuesta, CAE, vencimiento e intentos. Los reintentos deben conservar correlación e idempotencia. No invocar ARCA dentro de una transacción SQL larga.

## 6. Datos, consistencia y auditoría

El modelo de datos se definirá mediante migraciones incrementales. Las cantidades e importes usarán tipos decimales adecuados. Las claves internas se mantendrán separadas de códigos regulatorios. Cada operación que afecte existencias deberá poder rastrearse hasta su documento de origen y actor.

Los detalles de tablas, relaciones, restricciones e índices se agregarán aquí cuando estén definidos e implementados; no se anticipan nombres de tablas como si fueran decisiones aprobadas.

## 7. Seguridad e integración

- Aplicar autorización en servidor para cada acción que modifique datos.
- Validar entradas y registrar errores sin exponer credenciales ni datos sensibles en logs.
- Mantener certificados, claves privadas y tokens fuera del control de versiones y del directorio público.
- Definir por separado configuración no sensible, credenciales por ambiente y permisos de acceso.
- Para integraciones externas, persistir estado e intentos de forma que un timeout no se interprete automáticamente como rechazo o ausencia de emisión.

## 8. Política para cambios y estados

Ante cada avance o modificación, actualizar esta arquitectura si cambia un proceso, componente, dato, integración, control de seguridad o decisión técnica. Actualizar también el manual si cambia un flujo visible para el usuario y la memoria reutilizable si surge una lección o criterio transferible. Registrar la fecha y el resumen en la bitácora de cada documento modificado.

Usar estos estados en la documentación:

- **Propuesta:** decisión o proceso objetivo pendiente de implementación.
- **Implementado:** existe en el código, pero aún no se declara verificado funcionalmente.
- **Verificado:** comprobado con una prueba o validación reproducible.

## 9. Bitácora de cambios

| Fecha | Cambio | Estado |
|---|---|---|
| 2026-09-28 | Se crea la línea base de arquitectura, se documenta el estado Laravel actual y se clasifican los procesos objetivo como propuesta/futuros. | Estado actual verificado; procesos propuestos |