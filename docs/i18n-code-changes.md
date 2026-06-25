# Cambios de código para soporte i18n/Atlas

Este documento describe los cambios realizados en el código de `edx-sga` para
que el XBlock pueda consumir traducciones desde `openedx-translations` mediante
Atlas durante la construcción de una imagen Open edX/Tutor.

El objetivo de este documento es explicar qué cambió, por qué cambió y cómo
afecta al flujo de traducciones. Para la guía completa de operación con Tutor,
ver [Internacionalización de edx-sga con Atlas y Tutor](atlas-i18n.md).

## Problema original

`edx-sga` tenía traducciones locales, pero no estaba integrado completamente al
flujo moderno de Open edX para XBlocks.

Los problemas principales eran:

- Los catálogos vivían bajo `edx_sga/locale`, pero Atlas espera una estructura
  compatible con `conf/locale`.
- El paquete no exponía de forma clara los catálogos para los dos consumidores:
  Django y el runtime de XBlock.
- Las plantillas se renderizaban manualmente, sin pasar por el servicio i18n
  del runtime.
- El XBlock no obtenía correctamente el servicio `i18n`; se asumía la existencia
  de `self.i18n_service`, pero ese atributo no es creado automáticamente por
  `@XBlock.needs("i18n")`.
- Los textos JavaScript que usan `gettext` necesitaban un catálogo JavaScript
  cargado antes del script del XBlock.

El síntoma más visible fue este error en Studio/CMS:

```text
'StaffGradedAssignmentXBlockWithMixins' object has no attribute 'i18n_service'
```

Ese error indicaba que el XBlock declaraba o intentaba usar i18n, pero no estaba
obteniendo el servicio de la forma correcta.

## Objetivo del cambio

El objetivo fue que, al construir la imagen con Tutor:

1. Atlas descargue los catálogos de `edx-sga` desde `openedx-translations`.
2. Open edX compile esos catálogos en el path de plugins:

   ```text
   /openedx/edx-platform/conf/plugins-locale/xblock.v1/edx_sga/<locale>/LC_MESSAGES/
   ```

3. El XBlock renderice plantillas usando el servicio i18n del runtime.
4. Los textos de JavaScript puedan resolverse desde el catálogo JavaScript del
   XBlock.
5. La instalación por pip desde `OPENEDX_EXTRA_PIP_REQUIREMENTS` incluya los
   catálogos fuente y los enlaces necesarios.

## Resumen de archivos modificados

| Archivo o ruta | Cambio | Motivo |
| --- | --- | --- |
| `edx_sga/conf/locale/` | Se define como ubicación canónica de catálogos fuente. | Es el layout esperado para sincronización con Atlas. |
| `edx_sga/locale` | Enlace simbólico hacia `conf/locale`. | Mantiene compatibilidad con descubrimiento estándar de Django. |
| `edx_sga/translations` | Enlace simbólico hacia `conf/locale`. | Mantiene compatibilidad con el runtime y tooling de XBlock. |
| `MANIFEST.in` | Incluye catálogos y enlaces. | Evita que falten archivos al crear sdist/wheel. |
| `setup.py` | Incluye datos de traducción en el paquete. | Permite que pip instale los archivos necesarios. |
| `Makefile` | Agrega targets de extracción, compilación y validación. | Hace repetible el mantenimiento de catálogos. |
| `edx_sga/sga.py` | Usa `ResourceLoader`, servicio i18n y catálogo JS. | Permite consumir catálogos generados por Atlas. |
| `edx_sga/test_settings.py` | Agrega `edx_sga` a `INSTALLED_APPS`. | Permite que Django encuentre catálogos en pruebas/comandos locales. |
| `edx_sga/tests/test_sga.py` | Agrega pruebas de integración i18n. | Evita regresiones en el uso del servicio i18n. |

## Cambio 1: `conf/locale` como fuente canónica

Antes, el repositorio dependía de `edx_sga/locale`.

Ahora la fuente de verdad es:

```text
edx_sga/conf/locale/
├── config.yaml
├── en/LC_MESSAGES/
│   ├── django.po
│   └── djangojs.po
├── eo/LC_MESSAGES/
├── fake2/LC_MESSAGES/
└── rtl/LC_MESSAGES/
```

Este cambio alinea el repositorio con el layout que Atlas usa para plugins y
XBlocks. La ruta equivalente en `openedx-translations` queda así:

```text
translations/edx-sga/edx_sga/conf/locale/es_419/LC_MESSAGES/
├── django.po
└── djangojs.po
```

### Por qué no se agrega `es_419` al repo fuente

`edx-sga` mantiene catálogos fuente y pseudo-locales. Las traducciones reales,
como `es_419`, deben vivir en `openedx-translations`.

Eso evita duplicar traducciones y permite que Tutor compile siempre desde el
repositorio central de traducciones.

## Cambio 2: enlaces `locale` y `translations`

Se agregaron dos enlaces simbólicos:

```text
edx_sga/locale       -> conf/locale
edx_sga/translations -> conf/locale
```

Ambos apuntan al mismo contenido, pero existen por compatibilidad.

| Enlace | Consumidor principal | Motivo |
| --- | --- | --- |
| `locale` | Django | Django históricamente busca catálogos bajo `locale`. |
| `translations` | XBlock/runtime/tooling | Algunos XBlocks y herramientas esperan esta ruta. |
| `conf/locale` | Atlas y mantenimiento | Es la ruta canónica que se sincroniza con `openedx-translations`. |

La regla de mantenimiento es sencilla: editar siempre `edx_sga/conf/locale`.
Los enlaces existen para consumo, no para edición manual.

## Cambio 3: empaquetado de catálogos

Se actualizó el empaquetado para que los catálogos no se pierdan al instalar el
paquete con pip.

Esto importa porque Tutor instala `edx-sga` usando:

```yaml
OPENEDX_EXTRA_PIP_REQUIREMENTS:
  - git+https://github.com/MCPRDev/edx-sga.git@<branch-or-commit>#egg=edx-sga
```

Si `setup.py` o `MANIFEST.in` no incluyen los catálogos, el código puede quedar
instalado sin sus archivos de traducción. En ese caso, Atlas podría descargar
catálogos externos, pero el paquete instalado no tendría el layout esperado para
validación, extracción o compatibilidad local.

## Cambio 4: Makefile de traducciones

Se agregó un `Makefile` con targets de i18n.

Los más importantes son:

| Target | Qué hace |
| --- | --- |
| `make extract_translations` | Extrae strings desde Python, Django templates y JavaScript. |
| `make compile_translations` | Compila `.po` a `.mo` y genera catálogo JS. |
| `make dummy_translations` | Genera traducciones dummy para pruebas. |
| `make validate_translations` | Valida extracción y catálogos dummy. |
| `make check_translations_up_to_date` | Verifica que los catálogos fuente estén actualizados. |

Tutor no depende de este `Makefile` para construir la imagen. Tutor usa Atlas
desde `edx-platform`. El Makefile sirve para desarrollo del repositorio y para
mantener actualizados los catálogos fuente antes de sincronizar con
`openedx-translations`.

## Cambio 5: obtener el servicio i18n correctamente

Se agregó la declaración:

```python
@XBlock.needs("i18n")
```

Pero esta declaración no crea automáticamente un atributo
`self.i18n_service`.

La forma correcta de obtener el servicio es:

```python
def _get_i18n_service(self):
    return self.runtime.service(self, "i18n")
```

Esto evita el error:

```text
'StaffGradedAssignmentXBlockWithMixins' object has no attribute 'i18n_service'
```

### Por qué esto era necesario

El runtime de XBlock maneja servicios de forma explícita. `needs("i18n")`
declara la dependencia, pero el bloque debe pedir el servicio al runtime cuando
lo necesita.

Este patrón es consistente con otros servicios usados por SGA, como:

```python
self.runtime.service(self, "user")
self.runtime.service(self, "replace_urls")
```

## Cambio 6: render de templates con `ResourceLoader`

Antes, la plantilla se renderizaba manualmente.

Ahora se renderiza usando:

```python
loader.render_django_template(
    "templates/staff_graded_assignment/show.html",
    context=context,
    i18n_service=i18n_service,
)
```

Esto es importante porque conecta el render del template con el servicio i18n
del XBlock. Sin ese servicio, tags como estos no siempre resuelven el catálogo
correcto del plugin:

```django
{% trans "Grade Submissions" %}
{% trans "Upload your assignment" %}
{% blocktrans %}...{% endblocktrans %}
```

### Impacto

Después de este cambio, el template puede traducirse usando los catálogos que
Atlas instala en:

```text
conf/plugins-locale/xblock.v1/edx_sga/<locale>/LC_MESSAGES/django.mo
```

## Cambio 7: catálogo JavaScript del XBlock

SGA también tiene textos JavaScript con `gettext`, por ejemplo en:

```text
edx_sga/static/js/src/edx_sga.js
```

Para esos textos no basta con `django.po`; también se necesita `djangojs.po`.

Se declaró el namespace:

```python
i18n_js_namespace = "StaffGradedAssignmentI18N"
```

Y se solicita la URL del catálogo JS al servicio i18n:

```python
static_i18n_js_url = self._get_statici18n_js_url(i18n_service)
```

Si el runtime expone la URL, el fragment agrega el catálogo antes del JS del
XBlock.

### Por qué el orden importa

El catálogo JS debe cargarse antes de que `edx_sga.js` ejecute llamadas como:

```javascript
gettext("Uploading...")
```

Si el catálogo se carga después, el texto puede quedar en inglés o depender del
catálogo global de Open edX en vez del catálogo específico del XBlock.

## Cambio 8: pruebas agregadas

Se agregaron pruebas para cubrir los puntos críticos:

- El helper `render_template` debe delegar en `ResourceLoader`.
- El servicio i18n debe obtenerse con `runtime.service(self, "i18n")`.
- La URL del catálogo JavaScript debe pedirse al servicio i18n.

Estas pruebas protegen contra regresiones como volver a usar
`self.i18n_service` o volver al render manual de templates.

## Impacto en Tutor

Para que estos cambios se reflejen en Tutor se necesita:

1. Instalar una rama o commit del fork de `edx-sga` que contenga estos cambios.
2. Apuntar `ATLAS_REPOSITORY` y `ATLAS_REVISION` al repositorio/rama de
   traducciones correcta.
3. Reconstruir la imagen `openedx`.

En caso de usar branches móviles, es recomendable reconstruir sin cache o fijar
commits para validar:

```yaml
OPENEDX_EXTRA_PIP_REQUIREMENTS:
  - git+https://github.com/MCPRDev/edx-sga.git@<commit>#egg=edx-sga
```

```yaml
ATLAS_REVISION: <commit-de-openedx-translations>
```

## Cómo validar

Dentro del contenedor LMS/CMS, se puede revisar que el paquete instalado tenga
el template actualizado:

```bash
tutor local run cms bash -lc '
python - <<PY
from pathlib import Path
import edx_sga

base = Path(edx_sga.__file__).parent
print(base)
print((base / "templates/staff_graded_assignment/show.html").read_text()[:500])
PY
'
```

También se puede revisar el catálogo compilado de Atlas:

```bash
tutor local run cms bash -lc '
python - <<PY
import gettext

mo = "/openedx/edx-platform/conf/plugins-locale/xblock.v1/edx_sga/es_419/LC_MESSAGES/django.mo"
t = gettext.GNUTranslations(open(mo, "rb"))

for msgid in ["Submit", "Your score is", "Grade for"]:
    print(msgid, "=>", t.gettext(msgid))
PY
'
```

## Riesgos evitados

Estos cambios evitan:

- Dependencia de un atributo inexistente `self.i18n_service`.
- Templates renderizados sin contexto i18n de XBlock.
- Paquetes instalados sin catálogos.
- Textos JavaScript sin catálogo propio.
- Divergencia entre layout local y layout esperado por Atlas.

## Lo que este cambio no hace

Este cambio no modifica:

- La lógica de calificación.
- La lógica de roles staff/instructor.
- El flujo de aprobación de notas.
- El contenido de cursos existentes.
- Los valores guardados en `display_name`.

Si un título como `Staff Graded Assignment` aparece en inglés como encabezado
del componente, puede ser porque `display_name` es contenido editable del curso,
no necesariamente un label de interfaz.

