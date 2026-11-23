# Examples

Invented examples. The "before" column shows the voice to avoid.

## A rule without its reason

| Before | After |
|---|---|
| Instalar los hooks una vez por clon. No omitirlos. | Los hooks se instalan una vez por clon. Hasta que se instalen no corre ninguna validación local, y el clon no lo advierte. |
| No reconstruir la imagen en cada ambiente. | La imagen se construye una sola vez, porque reconstruirla en cada ambiente produciría imágenes distintas a partir del mismo commit. |

## Internal labels and maxims

| Before | After |
|---|---|
| Donde una convención la hace cumplir un gate, el gate es la fuente de verdad. | Los hooks y CI deciden qué pasa y qué no. Esta guía los describe, de modo que si dice algo distinto de su configuración, vale lo que está configurado. |
| El guard rechaza bases shallow. | La comparación falla cuando el clon no tiene el historial completo, porque no puede calcular qué cambió. |

## Stacked fragments

| Before | After |
|---|---|
| Los hooks rechazan el commit. No corrigen. No vuelven a hacer staging. | Los hooks rechazan el commit pero no corrigen nada, así que el formato se arregla a mano antes de volver a intentarlo. |

## Emphasis

| Before | After |
|---|---|
| El orden de los tests es **aleatorio**, y eso es **clave**. | **El orden de los tests es aleatorio.** Un orden fijo oculta que un test dependa de otro, y con orden aleatorio esa dependencia aparece como una falla. |

## Costs and limits

| Before | After |
|---|---|
| La solución es robusta y cubre todos los casos. | La verificación cubre el código, no los datos. Un error que solo aparece con volumen real se detecta después del merge. |
