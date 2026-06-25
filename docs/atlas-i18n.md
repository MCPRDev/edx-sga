# Internacionalización de edx-sga con Atlas y Tutor

Este documento describe la adaptación de `edx-sga` al flujo moderno de
traducciones de Open edX: Atlas obtiene los catálogos desde
`openedx-translations`, `edx-platform` los compila durante la construcción de
Tutor, y el XBlock los consume en LMS y Studio.

La implementación está orientada al despliegue que instala:

- `MCPRDev/edx-sga`, rama `fix/edx-sga-openedx-translations`.
- `eduNEXT/openedx-translations`, rama
  `ednx-release/teak.atentamente`.

El diseño no queda acoplado a esos forks: los mismos pasos aplican a cualquier
fork equivalente de ambos repositorios.

## Objetivo

Antes de este cambio, SGA contenía catálogos locales bajo `edx_sga/locale`,
pero no seguía completamente el contrato de Atlas para XBlocks. En particular,
no tenía una fuente de catálogos en `conf/locale`, no exponía el catálogo
JavaScript de XBlock y renderizaba la plantilla de forma manual.

El resultado esperado es que, al seleccionar `es-419` en Open edX:

1. Atlas descargue los catálogos de `edx-sga` desde
   `openedx-translations`.
2. Tutor compile los archivos `.po` a `.mo` y genere el catálogo JavaScript.
3. El XBlock traduzca sus plantillas, sus textos de Python y sus textos de
   JavaScript.

En URLs y configuración de Open edX el locale se llama `es-419`. En nombres de
directorios y catálogos gettext se usa `es_419`. Ambas formas son correctas y
se refieren al mismo locale.

## Arquitectura resultante

```text
edx-sga (código fuente)                  openedx-translations
┌─────────────────────────┐             ┌─────────────────────────────────────────┐
│ edx_sga/conf/locale     │             │ translations/edx-sga/edx_sga/conf/locale │
│ ├── en/.../django.po    │  extracción │ └── es_419/LC_MESSAGES/                  │
│ └── en/.../djangojs.po  │ ───────────▶ │     ├── django.po                        │
└───────────┬─────────────┘             │     └── djangojs.po                      │
            │                           └──────────────────┬──────────────────────┘
            │ enlaces y empaquetado                         │ Atlas durante el build
            ▼                                               ▼
   locale/ y translations/                         edx-platform/conf/plugins-locale/
                                                           xblock.v1/edx_sga/es_419/
                                                                      │
                                                                      ▼
                                                        LMS y CMS renderizan SGA
```

`conf/locale` es la fuente de verdad de este repositorio. Los catálogos de
idiomas traducidos viven en `openedx-translations`, no deben mantenerse como
copias manuales dentro de `edx-sga`.

## Cambios en el repositorio

| Archivo o ruta | Cambio | Impacto |
| --- | --- | --- |
| `edx_sga/conf/locale/` | Nueva ubicación canónica de los catálogos. | Permite extracción y sincronización compatibles con Atlas. |
| `edx_sga/locale` | Enlace a `conf/locale`. | Conserva el descubrimiento convencional de catálogos por Django. |
| `edx_sga/translations` | Enlace a `conf/locale`. | Expone los catálogos al runtime y recursos de XBlock. |
| `Makefile` | Nuevos comandos de extracción, compilación y validación. | Hace repetible el mantenimiento del catálogo fuente. |
| `setup.py` y `MANIFEST.in` | Inclusión explícita de catálogos y enlaces en el paquete. | Evita que una instalación por wheel o pip pierda los archivos de traducción. |
| `edx_sga/sga.py` | Integración del servicio i18n, render de template y catálogo JavaScript. | Hace que el XBlock use los catálogos que Atlas descargó. |
| `edx_sga/test_settings.py` | Añade `edx_sga` a las aplicaciones de pruebas. | Permite a Django descubrir los catálogos al ejecutar comandos locales de i18n. |
| `edx_sga/tests/test_sga.py` | Pruebas del servicio i18n y de la URL del catálogo JavaScript. | Evita que vuelva el error por asumir `self.i18n_service`. |

### 1. Nueva ubicación canónica de catálogos

Los catálogos previos se movieron de `edx_sga/locale` a:

```text
edx_sga/conf/locale/
├── config.yaml
├── en/LC_MESSAGES/
│   ├── django.po
│   └── djangojs.po
├── eo/
├── fake2/
└── rtl/
```

Este es el layout que Atlas espera de un XBlock o plugin Python. El archivo
`config.yaml` mantiene únicamente `en` como idioma fuente. No se agrega
`es_419` allí, porque Atlas lo obtiene desde el repositorio de traducciones.

### 2. Enlaces simbólicos de compatibilidad

Se añadieron los enlaces siguientes dentro del paquete:

```text
edx_sga/locale       -> conf/locale
edx_sga/translations -> conf/locale
```

Ambos son intencionales y cubren consumidores distintos:

| Ruta | Motivo |
| --- | --- |
| `locale` | Conserva el mecanismo convencional de descubrimiento de catálogos de Django. SGA también se comporta como aplicación Django. |
| `translations` | Expone los catálogos con el nombre que emplean los runtimes y recursos de XBlock. |
| `conf/locale` | Es la ruta de mantenimiento, extracción y sincronización con Atlas. |

No debe editarse contenido a través de los enlaces: se trabaja siempre sobre
`edx_sga/conf/locale`.

### 3. Empaquetado de los catálogos

`setup.py` incluye ahora `locale` y `translations` dentro de `package_data`.
`MANIFEST.in` incluye `conf/locale`, `locale` y `translations` en el sdist.

Esto evita el caso en que el código Python se instala correctamente pero los
catálogos desaparecen al construir un wheel o una imagen. Es especialmente
importante para instalaciones de Tutor con `OPENEDX_EXTRA_PIP_REQUIREMENTS`.

### 4. Makefile de traducciones

El `Makefile` incorpora comandos de mantenimiento local:

| Comando | Efecto |
| --- | --- |
| `make extract_translations` | Extrae cadenas de Python, templates y JavaScript hacia `edx_sga/conf/locale/en/LC_MESSAGES`. Fusiona los parciales de `i18n_tool` en `django.po`. |
| `make compile_translations` | Compila los `.po` a `.mo` y genera el catálogo JavaScript con el namespace de SGA. |
| `make dummy_translations` | Genera catálogos de idiomas ficticios de prueba. |
| `make validate_translations` | Comprueba extracción y catálogos dummy. |
| `make check_translations_up_to_date` | Regenera, compila y verifica que no haya cadenas fuente sin extraer. |

Estos comandos requieren `i18n-tools` y las dependencias de desarrollo del
repositorio. **Tutor no ejecuta este Makefile dentro del paquete de SGA**; Tutor
realiza el pull y la compilación de XBlocks desde `edx-platform`. El Makefile
sirve para mantener el catálogo fuente y para validar cambios antes de subirlos
a `openedx-translations`.

## Cambios de runtime en el XBlock

### Declaración del servicio i18n

La clase `StaffGradedAssignmentXBlock` declara:

```python
@XBlock.needs("i18n")
```

Esto documenta que el bloque requiere el servicio de internacionalización
proporcionado por el runtime. Se conserva junto a los servicios existentes
`user` y `replace_urls`.

Importante: `XBlock.needs("i18n")` **no crea** un atributo
`self.i18n_service`. Solo declara el requisito al runtime. El servicio debe
obtenerse explícitamente:

```python
def _get_i18n_service(self):
    return self.runtime.service(self, "i18n")
```

Esta diferencia fue la causa del error de Studio:

```text
'StaffGradedAssignmentXBlockWithMixins' object has no attribute 'i18n_service'
```

El diagnóstico dentro del contenedor CMS confirmó que la clase tenía
`i18n_js_namespace`, pero no tenía el atributo `i18n_service`. La corrección
evita depender de un atributo inexistente y sigue el mismo patrón que SGA ya
usaba para `user` y `replace_urls`.

### Render de templates

Antes, SGA leía la plantilla como bytes y la procesaba con
`django.template.Template`. Esa vía no conecta la plantilla con el servicio
i18n del XBlock.

Ahora `student_view` obtiene el servicio una vez y lo pasa a:

```python
loader.render_django_template(
    "templates/staff_graded_assignment/show.html",
    context=context,
    i18n_service=i18n_service,
)
```

El helper local `render_template` conserva una interfaz pequeña y delega al
`ResourceLoader`. Se usa `xblock.utils.resources.ResourceLoader` y existe un
fallback para releases antiguas que lo exponían desde `xblockutils`.

Consecuencia: etiquetas como `{% trans "Grade Submissions" %}` y
`{% blocktrans %}` en `show.html` pueden resolver el catálogo específico del
XBlock descargado por Atlas.

### Textos de JavaScript

`edx_sga/static/js/src/edx_sga.js` contiene llamadas a `gettext`, por ejemplo
para errores de carga de archivo y mensajes de calificación. Una traducción
completa necesita un catálogo JavaScript, no solo `django.po`.

Por ello la clase declara:

```python
i18n_js_namespace = "StaffGradedAssignmentI18N"
```

Durante `student_view`, SGA solicita al servicio i18n la URL del catálogo:

```python
static_i18n_js_url = self._get_statici18n_js_url(i18n_service)
if static_i18n_js_url:
    fragment.add_javascript_url(static_i18n_js_url)
```

El catálogo se añade antes del JavaScript propio del XBlock. Si el runtime no
expone la función de URL —por ejemplo, en un entorno local antiguo— el método
devuelve `None` y el XBlock sigue renderizando sin introducir un error.

## Catálogos en openedx-translations

Para `es_419`, los archivos se agregan al fork de traducciones en esta ruta
exacta:

```text
translations/edx-sga/edx_sga/conf/locale/es_419/LC_MESSAGES/
├── django.po
└── djangojs.po
```

| Archivo | Cubre |
| --- | --- |
| `django.po` | Cadenas de Python y de templates Django, incluyendo `{% trans %}` y `{% blocktrans %}`. |
| `djangojs.po` | Cadenas llamadas desde `gettext(...)` en `edx_sga.js`. |
| `.mo` | Artefacto compilado. No se edita: lo genera el proceso de compilación de `edx-platform` durante la construcción. |

Los `msgid` deben coincidir exactamente con la cadena extraída. Una traducción
puede estar en el path correcto y aun así no verse si el `msgid`, los
placeholders (`%(name)s`) o el dominio no coinciden.

## Configuración y construcción con Tutor

### 1. Instalar el fork de SGA

Tutor debe instalar el fork que contiene estos cambios, no la versión publicada
de MIT. En `OPENEDX_EXTRA_PIP_REQUIREMENTS` debe aparecer una entrada como:

```text
git+https://github.com/MCPRDev/edx-sga.git@fix/edx-sga-openedx-translations#egg=edx-sga
```

Si se instala otro commit o la dependencia original, Atlas puede descargar los
catálogos pero el XBlock seguirá sin tener el código que los consume.

### 2. Apuntar Atlas al fork de traducciones

```bash
tutor config save \
  --set ATLAS_REPOSITORY=eduNEXT/openedx-translations \
  --set ATLAS_REVISION=ednx-release/teak.atentamente
```

El nombre `ATLAS_REPOSITORY` debe referirse al fork que realmente contiene la
ruta indicada arriba. La revisión debe existir y contener los commits de los
catálogos.

### 3. Construir la imagen

```bash
tutor images build openedx
tutor local stop
tutor local start -d
```

Si se reutiliza una capa de Docker que contiene una descarga anterior de Atlas,
hay que ejecutar el equivalente a una construcción sin caché que soporte la
versión instalada de Tutor.

Al construir, `edx-platform` descubre los XBlocks instalados y ejecuta su
flujo de pull/compilación. En el log deben aparecer acciones relacionadas con:

```text
pull_xblock_translations
compile_xblock_translations
```

## Validación dentro de Tutor

Después del build, estos son los artefactos esperados:

```text
/openedx/edx-platform/conf/plugins-locale/xblock.v1/edx_sga/es_419/LC_MESSAGES/
├── django.po
└── django.mo

/openedx/edx-platform/cms/static/js/xblock.v1-i18n/edx_sga/es_419.js
/openedx/edx-platform/lms/static/js/xblock.v1-i18n/edx_sga/es_419.js
```

Una comprobación útil es:

```bash
tutor local run cms bash -lc \
  'find /openedx/edx-platform/conf/plugins-locale/xblock.v1/edx_sga/es_419 -type f -print'
```

Y para confirmar qué código está instalado:

```bash
tutor local run cms bash -lc \
  'python -c "import edx_sga.sga; print(edx_sga.sga.__file__)"'
```

Finalmente, prueba una cadena conocida de la plantilla, como `Grade
Submissions`, y una cadena JavaScript, como `Uploading...`, con un usuario
cuya preferencia de idioma sea `es-419`.

## Guía de diagnóstico

| Síntoma | Causa probable | Acción |
| --- | --- | --- |
| No existe `conf/plugins-locale/.../edx_sga/es_419` | Atlas no encuentra el path, la rama no es la configurada o SGA no está instalado durante el build. | Confirmar `ATLAS_REPOSITORY`, `ATLAS_REVISION`, la ruta externa y `OPENEDX_EXTRA_PIP_REQUIREMENTS`. Reconstruir `openedx`. |
| Existe `.po`, pero falta `.mo` | No se ejecutó o falló la compilación. | Buscar `compile_xblock_translations` en el log y revisar errores de gettext. |
| Studio muestra `object has no attribute i18n_service` | Se trató `@XBlock.needs` como si inyectara un atributo. | Usar `self.runtime.service(self, "i18n")`, como hace la implementación actual. |
| Templates siguen en inglés | El usuario no usa `es-419`, el `msgid` no coincide o la plantilla no recibe el servicio i18n. | Confirmar el idioma activo, el `django.po` y el uso de `render_django_template`. |
| Solo textos dinámicos en inglés | Falta `djangojs.po` o no se cargó el archivo `es_419.js`. | Añadir/verificar `djangojs.po` y revisar el catálogo en `xblock.v1-i18n`. |
| La traducción funciona localmente, pero no en la imagen | Docker reutilizó una capa o se instaló otro paquete. | Reconstruir sin caché y comprobar `edx_sga.sga.__file__` y el origen del paquete. |

## Pruebas incluidas en el repositorio

Se añadieron pruebas unitarias para verificar que:

1. Las plantillas delegan en `ResourceLoader.render_django_template` y reciben
   el servicio i18n.
2. El servicio se solicita explícitamente al runtime con
   `runtime.service(block, "i18n")`.
3. La URL del catálogo JavaScript se obtiene desde ese servicio para el bloque
   actual.

Las pruebas se ejecutan junto con las existentes del repositorio mediante:

```bash
tox
```

Para validar exclusivamente la sintaxis de los catálogos instalados se puede
usar `msgfmt --check` sobre cada `.po`.

## Resumen operativo

Para agregar o cambiar una traducción:

1. Cambiar cadenas fuente en `edx-sga` si es necesario.
2. Ejecutar `make extract_translations` y revisar los `msgid` resultantes.
3. Actualizar `django.po` y, cuando haya JavaScript, `djangojs.po` en el fork
   de `openedx-translations`.
4. Confirmar que Tutor instala la rama correcta de `edx-sga` y apunta Atlas a
   la rama correcta de traducciones.
5. Reconstruir `openedx`, reiniciar LMS/CMS y verificar los artefactos
   compilados.
6. Probar en Studio y LMS con `es-419` seleccionado.

El hecho de que Atlas descargue archivos no basta por sí solo: el paquete debe
estar preparado para empaquetarlos, `edx-platform` debe compilarlos, y el
XBlock debe solicitar correctamente el servicio i18n al runtime. Esta
implementación cubre las tres capas.
