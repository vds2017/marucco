# Manual de usuario

Fecha de inicio: 2026-09-28  
Estado: estructura inicial; todavía no hay funciones de negocio disponibles para documentar.

## 1. Alcance

Este manual describirá cómo operar las funciones habilitadas de la aplicación. Las instrucciones se incorporarán cuando cada flujo esté implementado y verificado; las funciones planificadas no se presentan como disponibles.

La [arquitectura de la solución](../desarrollo/arquitectura.md) registra el diseño y los procesos técnicos. La [memoria reutilizable](../desarrollo/MEMORIA_REUTILIZABLE_ARCA_STOCK.md) conserva aprendizajes y criterios técnicos.

## 2. Estado actual

La aplicación muestra la página de bienvenida de Laravel en `/`. No hay todavía procedimientos de stock, recepción, fabricación, remitos ni facturación que el usuario pueda ejecutar.

## 3. Secciones previstas

Se agregarán instrucciones verificadas para las funciones que se incorporen, por ejemplo:

- acceso y navegación;
- productos, insumos y depósitos;
- consulta y ajustes de stock;
- órdenes de compra y recepción;
- recetas y órdenes de fabricación;
- remitos de entrada y salida;
- usuarios, permisos y resolución de errores.

Cada procedimiento incluirá, cuando corresponda, permisos necesarios, pasos, datos requeridos, resultado esperado y manejo de errores. No se incluirán credenciales ni secretos.

## 4. Política de actualización

Actualizar este manual en cada cambio que modifique una pantalla, permiso, validación o procedimiento visible para el usuario. Registrar el cambio en la bitácora. Si el cambio también altera procesos o decisiones técnicas, actualizar además [`docs/desarrollo/arquitectura.md`](../desarrollo/arquitectura.md); agregar a la memoria reutilizable solo criterios que deban conservarse para futuros desarrollos.

Documentar únicamente comportamientos disponibles y verificados. Si una función se modifica o retira, corregir el procedimiento afectado en el mismo avance.

## 5. Bitácora de cambios

| Fecha | Cambio | Estado |
|---|---|---|
| 2026-09-28 | Se crea la estructura del manual, se registra el estado inicial y la política para documentar flujos verificados. | Línea base |