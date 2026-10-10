# Changelog

Cambios relevantes de AUDITPRO por versión, escritos desde el punto de vista del usuario.

## v1.1.0 — 2026-10-08

- Antes de ejecutar, la aplicación muestra las columnas que necesita cada análisis. Con un
  archivo cargado, marca cuáles encuentra.
- Mapeo de columnas manual en todas las opciones: cada dato muestra su estado y el aviso
  desaparece al asignarlo bien. Si falta una obligatoria, no se puede ejecutar.
- Conciliación bancaria: el desplegable de hoja aparece en los dos extractos.
- Conciliación bancaria: un desplegable explica qué hace la herramienta y qué contiene cada
  una de las 8 hojas del Excel de resultados.
- Pantalla de bienvenida al abrir la aplicación, con una explicación de las opciones. El
  menú lateral permanece oculto hasta pulsar "Entrar".

## v1.0.0 — 2026-10-04

Primera versión estable en uso real.

- Tipologías por asiento y tipologías por usuario.
- Importes duplicados por estructura y partes vinculadas.
- Pruebas de revisión (seis): Ley de Benford, asientos faltantes, fechas inusuales (fines de
  semana y festivos), importes acabados en 999, múltiplos de 1.000 e importes superiores a
  materialidad.
- Conciliación bancaria con mapeo manual de columnas.
- Corrección: los extractos con columnas de cabecera repetidas ya no provocan error.
