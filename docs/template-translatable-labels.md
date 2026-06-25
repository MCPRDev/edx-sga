# Cambios del template para labels traducibles

Este documento explica los cambios realizados en
`edx_sga/templates/staff_graded_assignment/show.html` para que algunos textos
del Student View y del modal de calificación sean traducibles de forma estable.

Este cambio es posterior a la integración con Atlas. Atlas ya cargaba
traducciones, pero algunos textos seguían apareciendo en inglés porque el
template no los marcaba correctamente o porque mezclaba variables de
Underscore.js con tags de traducción de Django.

## Problema observado

Después de habilitar Atlas/i18n, varios textos sí traducían:

- `Upload a different file`
- `File uploaded`
- `This assignment has not yet been graded.`
- `Grade Submissions`
- `Grade:`
- `Comment:`

Pero otros seguían en inglés:

- `Submit`
- `Your score is`
- `Grade for`

Esto indicaba que el catálogo estaba cargando correctamente, pero esos textos
puntuales no estaban expresados de forma adecuada en el template.

## Causa raíz

El template de SGA combina dos motores:

1. Django templates, evaluados en el servidor.
2. Underscore.js templates, evaluados en el navegador.

El archivo `show.html` contiene bloques como:

```html
<script type="text/template" id="sga-tmpl">
  ...
  <%= graded.score %>
  ...
</script>
```

Django procesa primero los tags como:

```django
{% trans "..." %}
{% blocktrans %}...{% endblocktrans %}
```

Luego, en el navegador, Underscore.js procesa expresiones como:

```javascript
<%= graded.score %>
```

Cuando se mezclan ambas sintaxis dentro de un mismo `msgid`, se vuelve frágil
mantener traducciones como:

```po
msgid "Your score is <%= graded.score %> / <%= max_score %>"
msgstr "Tu puntuación es <%= graded.score %> / <%= max_score %>"
```

Ese msgid depende de una expresión JavaScript exacta. Si se cambia el nombre de
una variable, espacios o sintaxis del template, la traducción deja de coincidir.

## Decisión tomada

Para estos labels cortos, se decidió traducir solo el texto estático y dejar los
valores dinámicos fuera del `msgid`.

Esto aplica a:

- `Submit`
- `Your score is`
- `Grade for`

La decisión reduce fragilidad y hace que el catálogo de traducciones sea más
simple.

## Cambio 1: botón final `Submit`

### Antes

```html
<a class="button finalize-upload">Submit</a>
```

Ese texto era literal. Aunque el catálogo tuviera:

```po
msgid "Submit"
msgstr "Entregar tarea"
```

Django no podía traducirlo porque no estaba marcado con `{% trans %}`.

### Después

```django
<a class="button finalize-upload">{% trans "Submit" %}</a>
```

### Impacto

El botón final de entrega ahora usa el mismo msgid que otros botones `Submit`
del template.

En `openedx-translations` basta con:

```po
msgid "Submit"
msgstr "Entregar tarea"
```

## Cambio 2: texto `Your score is`

### Antes

El template usaba una frase completa con expresiones Underscore dentro:

```django
{% blocktrans %}Your score is <%= graded.score %> / <%= max_score %>{% endblocktrans %}
```

Esto obligaba a tener una traducción dependiente de la sintaxis JS:

```po
msgid "Your score is <%= graded.score %> / <%= max_score %>"
msgstr "Tu puntuación es <%= graded.score %> / <%= max_score %>"
```

### Después

```django
{% trans "Your score is" %} <%= graded.score %> / <%= max_score %>
```

### Impacto

El texto estático se traduce con:

```po
msgid "Your score is"
msgstr "Tu puntuación es"
```

Y el resultado visual queda:

```text
Tu puntuación es 10 / 100
```

### Por qué no usar placeholders aquí

Una opción técnicamente correcta habría sido:

```po
msgid "Your score is %(graded_score)s / %(max_score)s"
msgstr "Tu puntuación es %(graded_score)s / %(max_score)s"
```

Pero en este caso los valores `graded.score` y `max_score` son expresiones de
Underscore.js que se resuelven en el navegador, no variables Django reales en el
servidor.

Para evitar que el catálogo dependa de sintaxis JS, se prefirió traducir el
label estático y mantener la expresión dinámica fuera del `msgid`.

## Cambio 3: texto `Grade for`

### Antes

El template intentaba usar un placeholder Django para inyectar un elemento HTML:

```django
{% blocktrans with student_name="<span id='student-name'/>" %}Grade for {{student_name}}{% endblocktrans %}
```

Este patrón era frágil por varias razones:

- El valor `student_name` no era realmente un nombre de estudiante.
- Era un string con HTML.
- El `<span />` self-closing no es una forma robusta para un elemento `span` en
  HTML.
- El nombre real se inyectaba después con JavaScript, no durante el render
  Django.

### Después

```django
{% trans "Grade for" %} <span id="student-name"></span>
```

### Impacto

El catálogo solo necesita:

```po
msgid "Grade for"
msgstr "Calificación de"
```

El nombre del estudiante se coloca después en el `<span>` usando JavaScript.

El resultado visual esperado es:

```text
Calificación de mario.perez
```

## Cambios en catálogos fuente

Los catálogos fuente locales se actualizaron para reflejar los nuevos msgid.

Antes existían entradas como:

```po
msgid "Your score is %(graded_score)s / %(max_score)s"
msgid "Grade for %(student_name)s"
```

Después se usan:

```po
msgid "Your score is"
msgid "Grade for"
```

## Cambios esperados en openedx-translations

Para `es_419`, el catálogo externo debe contener:

```po
msgid "Submit"
msgstr "Entregar tarea"

msgid "Your score is"
msgstr "Tu puntuación es"

msgid "Grade for"
msgstr "Calificación de"
```

Las entradas antiguas pueden dejarse temporalmente, pero ya no son necesarias
para este template:

```po
msgid "Your score is <%= graded.score %> / <%= max_score %>"
msgid "Grade for %(student_name)s"
```

Gettext ignora entradas no usadas, pero es mejor retirarlas para reducir ruido.

## Por qué este cambio no es parte de Atlas

Atlas se encarga de descargar y compilar catálogos. No corrige automáticamente
templates que:

- tienen texto literal sin `{% trans %}`;
- usan msgid que no coinciden con el catálogo;
- mezclan sintaxis de Django y Underscore de forma frágil.

Por eso fue necesario hacer un cambio específico en `show.html`.

## Cómo validar en Tutor

Dentro del contenedor CMS o LMS:

```bash
tutor local run cms bash -lc '
python - <<PY
from pathlib import Path
import edx_sga

template = Path(edx_sga.__file__).parent / "templates/staff_graded_assignment/show.html"
for i, line in enumerate(template.read_text().splitlines(), 1):
    if "Your score is" in line or "finalize-upload" in line or "Grade for" in line:
        print(i, line)
PY
'
```

Se espera ver:

```django
{% trans "Your score is" %}
{% trans "Submit" %}
{% trans "Grade for" %}
```

También se puede comprobar el catálogo compilado:

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

## Pruebas agregadas

Se agregó una prueba para confirmar que el template conserve estos textos como
traducibles:

- `Submit`
- `Your score is`
- `Grade for`

La intención es evitar que en el futuro alguien vuelva a dejar el botón como
texto literal o reintroduzca expresiones Underscore dentro del `msgid`.

## Lo que este cambio no modifica

Este cambio no altera:

- La lógica de subida de archivos.
- La lógica de calificación.
- La lógica de aprobación instructor/staff.
- El estado de entregas existentes.
- El `display_name` del componente.

Si una nota queda pendiente de aprobación, eso pertenece al flujo de calificación
de SGA y no a este cambio de template.

