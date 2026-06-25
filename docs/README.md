# Documentación de cambios de internacionalización

Esta carpeta contiene la documentación de los cambios hechos para que
`edx-sga` funcione correctamente con traducciones en Open edX/Tutor.

Los documentos están separados por propósito para que cada pull request o
cambio operativo pueda revisarse sin tener que leer toda la historia del
proyecto.

## Documentos principales

| Documento | Cuándo usarlo |
| --- | --- |
| [Internacionalización con Atlas y Tutor](atlas-i18n.md) | Guía general de arquitectura, flujo Atlas, Tutor, paths esperados y diagnóstico. |
| [Cambios de código para soporte i18n/Atlas](i18n-code-changes.md) | Explica los cambios de Python, empaquetado, servicios XBlock, catálogos y Makefile. |
| [Cambios del template para labels traducibles](template-translatable-labels.md) | Explica por qué se cambiaron textos como `Submit`, `Your score is` y `Grade for`. |
| [Nombre del estudiante en el modal de calificación](grade-modal-student-name.md) | Explica el cambio de `Grade for {student}` y el fallback de nombre completo a username. |

## Orden recomendado de lectura

1. Leer [Internacionalización con Atlas y Tutor](atlas-i18n.md) para entender
   cómo Tutor obtiene y compila traducciones desde `openedx-translations`.
2. Leer [Cambios de código para soporte i18n/Atlas](i18n-code-changes.md) para
   revisar qué se modificó en el paquete.
3. Leer [Cambios del template para labels traducibles](template-translatable-labels.md)
   para entender los ajustes específicos en `show.html`.
4. Leer [Nombre del estudiante en el modal de calificación](grade-modal-student-name.md)
   para revisar el cambio puntual del texto `Grade for`.

## Relación entre documentos

Los cambios de Atlas/i18n hacen que el XBlock pueda consumir catálogos de
traducción. Los cambios del template son una corrección posterior: una vez que
las traducciones cargaban correctamente, algunos textos seguían en inglés porque
no estaban marcados de forma adecuada o mezclaban sintaxis de Django con
Underscore.js.

El cambio del nombre del estudiante es independiente de Atlas: corrige cómo se
inyecta el nombre en el modal de calificación y evita que `Grade for` quede sin
un estudiante visible.

