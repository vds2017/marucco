# Memoria reutilizable: stock, recepciones y factura electrónica ARCA

Fecha de consolidación: 2026-09-28  
Proyecto de origen: `pos_arca`

## 1. Propósito

Esta memoria concentra los aprendizajes técnicos y operativos que conviene reutilizar en el próximo sistema de control de stock, productos compuestos, fabricación y remitos.

El documento es deliberadamente independiente del framework. Las rutas del proyecto actual se conservan como referencias históricas; en el nuevo sistema deberán convertirse en configuraciones, servicios, comandos y migraciones de Laravel, sin copiar la estructura legacy.

## 2. Decisión recomendada para el próximo proyecto

Crear un proyecto nuevo en Laravel con una base de datos nueva y un dominio más pequeño:

1. productos e insumos;
2. composiciones o recetas;
3. órdenes de fabricación;
4. movimientos de stock;
5. recepciones y remitos;
6. comprobantes de salida/remitos;
7. usuarios y permisos;
8. una capa fiscal preparada para incorporar factura electrónica más adelante.

No implementar inicialmente precios, márgenes, listas de precios ni AFIP si el cliente no los necesita. Sí conviene dejar separados los conceptos de documento comercial, detalle, cliente y numeración para que la incorporación futura de factura no obligue a rediseñar el stock.

## 3. Principios que deben conservarse

### 3.1 Stock como movimientos, no solo como saldo

El saldo de un producto es una lectura rápida. La fuente de auditoría debe ser `stock_movimiento`, con:

- producto;
- depósito, aunque inicialmente exista uno solo;
- tipo de movimiento;
- referencia al documento origen;
- cantidad delta;
- stock anterior y posterior;
- usuario y fecha;
- observación.

Toda recepción, fabricación, ajuste, anulación o salida debe generar un movimiento. No se debe corregir stock editando directamente el saldo salvo mediante un ajuste registrado.

### 3.2 Recepción parcial

La recepción debe actualizar únicamente lo efectivamente recibido. Una orden puede tener varias recepciones y cada ítem debe conservar:

- cantidad ordenada;
- cantidad recibida acumulada;
- cantidad pendiente;
- estado del ítem.

Al confirmar una recepción se actualizan la recepción, los acumulados de la orden y los movimientos de stock dentro de una transacción corta. Si una recepción se anula, debe generarse el movimiento inverso y conservarse la evidencia de la anulación.

### 3.3 Productos compuestos y fabricación

No modelar un producto compuesto como una descripción de texto. Usar una receta versionable:

- producto terminado;
- insumo o componente;
- cantidad requerida;
- unidad de medida;
- versión y vigencia;
- estado activa/inactiva.

Una orden de fabricación confirmada debe descontar componentes e ingresar productos terminados en una sola transacción. La orden debe ser idempotente: no puede confirmarse dos veces ni volver a generar movimientos por un reintento de la pantalla.

## 4. Factura electrónica ARCA: conocimiento reutilizable

### 4.1 Requisitos fiscales que no deben quedar implícitos

La configuración debe contemplar, como mínimo:

- CUIT y razón social del emisor;
- condición frente al IVA;
- domicilio fiscal;
- ambiente de ARCA: homologación o producción;
- certificado y clave privada por ambiente;
- punto de venta autorizado;
- tipo de comprobante;
- numeración independiente por punto de venta y tipo;
- datos fiscales del receptor;
- alícuotas y detalle de IVA;
- moneda, condición de venta y datos de vencimiento cuando correspondan.

Los datos regulatorios no deben mezclarse con IDs internos sin una decisión explícita. En el proyecto actual se detectó una inconsistencia entre `tipos_comprobantes.id` y el código AFIP guardado en `ventas.tipo_comprobante`; en el nuevo sistema se recomienda separar siempre `id` interno de `codigo_afip`.

### 4.2 Certificados y autorización

Flujo comprobado para homologación:

1. Generar clave privada y CSR con OpenSSL.
2. Usar el CUIT titular en el `serialNumber` del CSR.
3. Cargar el CSR en WSASS de ARCA.
4. Descargar el certificado PEM y guardarlo como `.crt`.
5. Crear la autorización del servicio `wsfe` para el alias del certificado.
6. Verificar el comprobante técnico de autorización.
7. Generar y firmar el TRA para WSAA.
8. Obtener Token y Sign, cuya validez operativa es temporal.

La clave privada, certificados, tokens y credenciales nunca deben versionarse. Deben vivir fuera de la carpeta pública o estar bloqueados por configuración del servidor.

### 4.3 WSAA, WSFE y estado de emisión

La emisión debe dividirse en tres etapas:

1. **Local:** guardar el documento en estado `PENDIENTE_CAE`, junto con un identificador único y el payload que se pretende enviar.
2. **Externa:** invocar WSFE fuera de una transacción SQL larga.
3. **Persistencia:** guardar CAE, vencimiento, respuesta completa y estado final en una transacción corta.

Estados recomendados:

`BORRADOR`, `PENDIENTE_CAE`, `ENVIANDO_AFIP`, `FACTURADO`, `ERROR_REINTENTABLE`, `ERROR_FINAL` y `REVERSADO_LOGICO`.

Nunca borrar un comprobante emitido o intentado. Ante una emisión incorrecta se utiliza el mecanismo fiscal correspondiente, normalmente una nota de crédito, y se conserva el historial.

### 4.4 Datos que deben conservarse

Por cada intento de emisión guardar:

- `request_id` o correlación;
- hash del payload;
- request enviado;
- respuesta cruda de ARCA;
- código y mensaje de error;
- inicio, fin y duración;
- cantidad de intento;
- usuario o proceso que lo ejecutó;
- estado resultante.

El retry debe operar sobre el mismo documento y ser idempotente. Un timeout no permite asumir que ARCA no emitió; antes de repetir hay que consultar el estado o el último comprobante cuando el caso lo requiera.

### 4.5 Condición IVA del receptor

En WSFE, `CondicionIVAReceptorId` debe enviarse siempre según la normativa y la condición registrada. No se debe eliminar el campo como fallback ante errores de validación: los errores 10243/10246 deben resolverse corrigiendo datos, catálogo o mapeo.

## 5. Arquitectura recomendada en Laravel

La traducción de la experiencia actual al nuevo proyecto debería ser:

| Necesidad | Diseño recomendado |
|---|---|
| Configuración | `.env` y tabla de configuración no sensible |
| Stock | `StockMovementService` + transacciones |
| Fabricación | `ManufacturingOrderService` |
| Recepción | `ReceiptService` con auditoría |
| Remitos | Documento y detalle independientes de factura |
| ARCA | `AfipAuthService` y `ElectronicInvoiceService` |
| Reintentos | Jobs/queues y tabla de intentos |
| Numeración | secuencia por punto de venta y tipo |
| Diagnóstico | comandos Artisan y logs estructurados |
| Auditoría | eventos de negocio y actor |

Las llamadas a ARCA no deben ocurrir dentro de `DB::transaction()`. El patrón equivalente al sistema aprendido es: persistir pendiente, despachar un job, llamar al servicio externo y persistir el resultado.

## 6. Mapa de rutas del proyecto actual

Estas rutas sirven para consultar el trabajo realizado y localizar ejemplos. No deben convertirse automáticamente en la estructura del nuevo proyecto.

### Documentación

- `ANALISIS_FACTURA_ELECTRONICA_ARCA.md`: diagnóstico inicial y datos fiscales requeridos.
- `GUIA_INSTALACION_FACTURA_ELECTRONICA.md`: instalación, configuración y pruebas iniciales.
- `Cert/Homologacion/Guía de Configuración Certificados.md`: CSR, WSASS y autorización `wsfe`.
- `ENTIDADES_RELACIONES.md`: entidades y relaciones del modelo actual.
- `Docs/Desarrollo/REPORTE_AUDITORIA_TABLAS_AFIP.md`: inconsistencias reales detectadas en catálogos y FKs.
- `Docs/Desarrollo/PROCEDIMIENTO_PRUEBA_FACTURACION_DOS_PASOS.md`: prueba manual de venta, stock, CAE y reintento.
- `Docs/Desarrollo/IMPLEMENTACION_FACTURACION_DOS_PASOS_CON_ROLLBACK.md`: estados, outbox, idempotencia y rollback lógico.
- `Docs/Desarrollo/DEPLOY_DONWEB_MULTI_CLIENTE.md`: aislamiento por cliente y checklist de despliegue.
- `Docs/Desarrollo/compra_orden_modelo.md`: órdenes, recepciones parciales y movimientos de stock.
- `Docs/Desarrollo/PROPUESTA_MODULO_COMPRAS_STOCK_PARCIAL.md`: modelo ampliado de compras y recepciones.

### Código y scripts de referencia

- `extensiones/GeneradorFacturaAFIP.php`: generador/integración fiscal del proyecto actual.
- `vendor/afipsdk/afip.php/`: dependencia AFIP instalada en el proyecto actual.
- `scripts/test_conexion_afip.php`: prueba de conexión.
- `scripts/test_cae_homologacion.php`: prueba de CAE en homologación.
- `scripts/test_afip_con_certs.php`: prueba con certificados.
- `scripts/diagnostico_afip_soap.php`: diagnóstico SOAP.
- `scripts/emitir_cae_venta.php`: emisión o reintento de una venta puntual.
- `scripts/cron_worker_afip.php`: worker de procesamiento AFIP.
- `scripts/actualizar_certificados_por_ambiente.php`: actualización de certificados por ambiente.
- `scripts/verificar_certificados.php`: verificación de certificados.
- `scripts/inspeccionar_schema.php` y `scripts/inspect_tables.php`: inspección de base.
- `scripts/migracion_afip_dos_pasos.sql`: migración del flujo desacoplado.
- `scripts/migracion_alinear_tipo_comprobante_codigo.sql`: corrección de código AFIP frente a ID interno.
- `scripts/migracion_modulo_compras_mvp.sql`: base del módulo de compras.

## 7. Lecciones de base de datos

- Usar migraciones incrementales e idempotentes.
- Consultar `INFORMATION_SCHEMA` antes de crear columnas, índices o FKs cuando se mantenga compatibilidad con instalaciones existentes.
- En MariaDB/MySQL históricos, preferir `LONGTEXT` para JSON si la versión o el hosting no garantizan soporte uniforme.
- Alinear tipos de columnas FK con sus PK: en este proyecto `ventas.id` es `INT`.
- Usar `DECIMAL` para importes y cantidades monetarias; no `FLOAT`.
- Separar claves internas de códigos regulatorios.
- No almacenar certificados, claves privadas ni tokens en tablas sin cifrado o control de acceso.
- Probar restauraciones de backup verificando que todas las migraciones posteriores sigan aplicadas.

## 8. Pruebas mínimas para el nuevo sistema

### Stock y fabricación

- recepción total y parcial;
- anulación de recepción;
- fabricación con receta vigente;
- falta de stock de un componente;
- reintento de confirmación sin duplicar movimientos;
- auditoría completa del saldo.

### Remitos

- remito con importe manual;
- remito sin política de precios;
- edición o anulación según regla definida;
- numeración por establecimiento o punto de emisión;
- impresión y reimpresión del documento correcto.

### ARCA futura

- homologación con certificado válido;
- certificado vencido o incorrecto;
- timeout con reintento;
- rechazo funcional por datos fiscales;
- CAE aprobado y vencimiento persistido;
- no duplicación ante doble ejecución;
- recuperación después de restaurar un backup.

## 9. Plan de trabajo optimizado

### Fase 1: núcleo de inventario

Definir productos, unidades, depósitos, movimientos, ajustes y usuarios. No incluir todavía AFIP.

### Fase 2: compras y recepción

Implementar órdenes, recepciones parciales, remitos de proveedor, anulación y auditoría.

### Fase 3: composiciones y fabricación

Implementar recetas versionadas, órdenes de fabricación, consumos, producción y trazabilidad.

### Fase 4: remitos de salida

Separar el remito del precio: el total puede ser manual y opcional según la necesidad del cliente.

### Fase 5: preparación fiscal

Crear catálogos fiscales, configuración de empresa, puntos de venta y numeración, sin activar emisión.

### Fase 6: factura electrónica futura

Incorporar WSAA/WSFE mediante jobs, estados, reintentos, auditoría, PDF/XML y pruebas de homologación.

## 10. Regla de documentación

La documentación de esta solución se mantiene como parte de cada avance o modificación funcional/técnica. Actualizar los documentos afectados y su bitácora en el mismo cambio:

1. [`docs/desarrollo/arquitectura.md`](arquitectura.md): procesos, componentes, datos, integraciones, seguridad y decisiones técnicas.
2. [`docs/manual_usuario/README.md`](../manual_usuario/README.md): procedimientos de uso observables por el usuario, permisos, validaciones y resolución de errores.
3. Esta memoria: aprendizajes reutilizables, decisiones heredadas y criterios técnicos que convenga conservar para desarrollos futuros.

Para cada modificación registrar fecha, resumen, documento afectado y estado. Documentar el modelo de datos/migración, backend/frontend y prueba mínima cuando correspondan. Si una modificación afecta más de un ámbito, actualizar todos los documentos pertinentes; no copiar el mismo contenido íntegro entre ellos.

La documentación debe distinguir siempre entre **Propuesta**, **Implementado** y **Verificado**. Solo describir como comportamiento disponible aquello que se haya comprobado en el código o mediante una prueba; reflejar en el manual únicamente flujos habilitados para usuarios.

## 11. Fuentes consolidadas

Esta memoria sintetiza los documentos Markdown funcionales y técnicos del proyecto actual, excluyendo `README` y documentación de dependencias de terceros. Los documentos originales se conservan como evidencia detallada y para consultar comandos o decisiones específicas.

## 12. Bitácora de cambios

| Fecha | Cambio | Estado |
|---|---|---|
| 2026-09-28 | Se establece la política de actualización coordinada con arquitectura y manual de usuario; se aclara el uso de estados documentales. | Política vigente |
