# Nombre del estudiante en el modal de calificación

Este documento explica el cambio realizado para que el modal de calificación
muestre correctamente el estudiante en la línea:

```text
Grade for {student}
```

o, en español:

```text
Calificación de {student}
```

El cambio afecta principalmente a:

- `edx_sga/templates/staff_graded_assignment/show.html`
- `edx_sga/static/js/src/edx_sga.js`
- `edx_sga/tests/test_sga.py`

## Problema observado

Al abrir el modal de calificación, el encabezado del formulario podía verse así:

```text
Grade for
```

o:

```text
Calificación de
```

sin mostrar el nombre del estudiante.

Esto hacía difícil confirmar a quién se estaba calificando, especialmente cuando
el modal se abre desde una tabla con varias entregas.

## Diseño anterior

El template usaba:

```django
{% blocktrans with student_name="<span id='student-name'/>" %}Grade for {{student_name}}{% endblocktrans %}
```

La intención era que Django produjera algo como:

```html
Grade for <span id="student-name"></span>
```

y que luego JavaScript llenara ese `<span>`.

## Problemas del diseño anterior

### 1. El placeholder no era el nombre del estudiante

El placeholder `student_name` no contenía un nombre. Contenía un string con
HTML:

```html
<span id='student-name'/>
```

Eso mezclaba dos responsabilidades distintas:

- traducción del label `Grade for`;
- estructura HTML usada por JavaScript.

### 2. El nombre real se inserta en el navegador

El nombre no existe como variable Django cuando se renderiza el modal. El modal
se renderiza una vez, y luego se rellena cuando el usuario hace click en una fila
de la tabla.

El flujo real es:

1. Django renderiza el template.
2. JavaScript obtiene datos de entregas.
3. Underscore.js renderiza la tabla de entregas.
4. El código asocia cada objeto `assignment` con su fila.
5. Al hacer click en `Enter grade`, JavaScript lee los datos de la fila.
6. JavaScript escribe el nombre en el modal.

Por esa razón, no era correcto tratar el nombre del estudiante como una variable
de `blocktrans`.

### 3. `<span />` no es una forma robusta de escribir un `span`

`span` no es un elemento void en HTML. Es preferible escribirlo con apertura y
cierre explícitos:

```html
<span id="student-name"></span>
```

Esto evita diferencias de parseo entre navegadores y mantiene el DOM más claro.

### 4. `fullname` puede venir vacío

El JavaScript original usaba solo:

```javascript
row.data('fullname')
```

Pero `fullname` viene de:

```python
student_module.student.profile.name
```

En Open edX ese campo puede estar vacío. Si `profile.name` está vacío, el modal
queda sin estudiante visible aunque la fila tenga `username`.

## Cambio realizado en el template

### Antes

```django
{% blocktrans with student_name="<span id='student-name'/>" %}Grade for {{student_name}}{% endblocktrans %}
```

### Después

```django
{% trans "Grade for" %} <span id="student-name"></span>
```

## Por qué se usa `Grade for` como msgid simple

Se decidió traducir solo el label estático:

```po
msgid "Grade for"
msgstr "Calificación de"
```

El nombre del estudiante queda fuera del `msgid` porque no se conoce durante el
render Django; se conoce cuando el usuario hace click en una fila.

Esto hace que el catálogo sea más estable y evita usar HTML como placeholder de
traducción.

## Cambio realizado en JavaScript

### Antes

```javascript
$(element).find('#student-name').text(row.data('fullname'));
```

Si `fullname` estaba vacío, el modal quedaba sin nombre.

### Después

```javascript
var studentName = row.data('fullname') || row.data('username') || '';
$(element).find('#student-name').text(studentName);
```

Ahora el modal intenta mostrar:

1. `fullname`, si existe.
2. `username`, si `fullname` está vacío.
3. string vacío como último fallback.

## De dónde salen `fullname` y `username`

El backend construye cada fila de calificación en `staff_grading_data`.

Los campos relevantes son:

```python
"username": student_module.student.username,
"fullname": student_module.student.profile.name,
```

Luego `edx_sga.js` asocia cada objeto `assignment` a la fila:

```javascript
data.assignments.map(function (assignment) {
  $(element).find('#grade-info #row-' + assignment.module_id).data(assignment);
});
```

Cuando el usuario hace click en el botón de calificación, el código recupera la
fila y lee sus datos:

```javascript
var row = $(this).parents("tr");
```

Por eso el fallback puede usar:

```javascript
row.data('fullname')
row.data('username')
```

## Resultado esperado

Si el perfil tiene nombre completo:

```text
Calificación de Mario Pérez
```

Si el perfil no tiene nombre completo:

```text
Calificación de mario.perez
```

## Por qué no usar `Grade for %(student_name)s`

`Grade for %(student_name)s` es una buena práctica cuando el valor se conoce en
el momento en que se traduce la cadena.

Ejemplo ideal:

```po
msgid "Grade for %(student_name)s"
msgstr "Calificación de %(student_name)s"
```

Pero en este modal el nombre no se conoce durante el render de Django. Se
inyecta después en el navegador. Para usar ese formato habría que mover la
construcción de la frase a JavaScript o generar un template JS adicional con
interpolación. Para este caso, separar label y valor es más simple y menos
riesgoso.

## Relación con traducciones

Este cambio necesita que el catálogo tenga:

```po
msgid "Grade for"
msgstr "Calificación de"
```

Ya no necesita:

```po
msgid "Grade for %(student_name)s"
msgstr "Calificación de %(student_name)s"
```

Esa entrada antigua puede permanecer temporalmente sin causar error, pero no se
usa con el template actual.

## Validación manual

Para validar en LMS:

1. Subir una entrega como estudiante.
2. Abrir el curso como instructor o course staff con permisos de calificación.
3. Abrir `Calificar entregas`.
4. Hacer click en `Introducir calificación`.
5. Confirmar que el modal muestre:

   ```text
   Calificación de <nombre-o-username>
   ```

Si no aparece el nombre completo pero sí aparece el username, el fallback está
funcionando correctamente.

## Validación en Tutor

Para confirmar que el template instalado contiene el cambio:

```bash
tutor local run cms bash -lc '
python - <<PY
from pathlib import Path
import edx_sga

template = Path(edx_sga.__file__).parent / "templates/staff_graded_assignment/show.html"
for i, line in enumerate(template.read_text().splitlines(), 1):
    if "student-name" in line or "Grade for" in line:
        print(i, line)
PY
'
```

Para confirmar el cambio JavaScript:

```bash
tutor local run cms bash -lc '
python - <<PY
from pathlib import Path
import edx_sga

script = Path(edx_sga.__file__).parent / "static/js/src/edx_sga.js"
for i, line in enumerate(script.read_text().splitlines(), 1):
    if "studentName" in line or "student-name" in line:
        print(i, line)
PY
'
```

## Pruebas agregadas

Se agregaron pruebas para verificar que:

- el template tenga un `<span id="student-name"></span>` real;
- el label `Grade for` permanezca marcado con `{% trans %}`;
- JavaScript use `fullname` y haga fallback a `username`.

Esto protege contra regresiones donde el modal vuelva a mostrar solo
`Grade for` sin estudiante.

## Lo que este cambio no corrige

Este cambio no modifica:

- permisos de calificación;
- aprobación instructor/staff;
- creación de `Score` en Submissions;
- estado `staff_score`;
- finalización de entregas.

Si una nota aparece como pendiente de aprobación, la causa está en la lógica de
roles/calificación, no en este cambio del nombre del estudiante.

