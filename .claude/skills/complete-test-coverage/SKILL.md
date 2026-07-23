---
name: complete-test-coverage
description: Audita y completa la cobertura de tests de un proyecto de forma exhaustiva pero eficiente en tokens — detecta archivos invisibles, ramas escondidas dentro de líneas "cubiertas", código muerto que en realidad es un bug, y clasifica cada gap antes de escribir nada. Úsalo cuando te pidan mejorar/completar cobertura de tests en cualquier stack (Deno, Node, Python, Go, etc.).
---

Úsalo como prompt inicial en cualquier proyecto con test suite existente. Sustituye
`<runtime>` por el gestor real (`deno test`, `npm test`/`jest`, `pytest`, `go test`, etc.)
y `<coverage-tool>` por su equivalente de cobertura.

---

## Objetivo

Completar la cobertura de tests del proyecto de forma **exhaustiva pero eficiente en
tokens**, apuntando al mejor puntaje de cobertura real posible: encontrar TODOS los gaps
reales (incluidos los invisibles y los que se esconden dentro de líneas ya "cubiertas"),
descartar los que no son gaps de verdad, y cerrar solo los que aportan señal real — sin
volcar tablas de cobertura completas en el chat en cada paso ni releer archivos
innecesariamente.

**"Mejor puntaje posible" no significa 100% a cualquier costo.** Significa: ningún gap
real se queda sin cerrar por pereza o por confiar en un reporte engañoso, pero el código
muerto, el defensivo-inalcanzable y el que requiere infraestructura viva sin punto de
inyección siguen quedando fuera de alcance (Fase 2) — perseguirlos igual sería el
desperdicio de tokens que esta auditoría existe para evitar. La diferencia frente a una
pasada superficial es que aquí no se declara "listo" hasta que el número que queda sin
cerrar está explícitamente justificado línea por línea, no simplemente redondeado.

## Regla de oro (ahorro de tokens)

- Analiza en bloque, no interactivo. Corre el suite completo con cobertura UNA vez,
  extrae el detalle a un archivo temporal, y haz todo el triage sobre ese archivo con
  `grep`/`sed`/scripts cortos — no repitas la corrida completa después de cada test
  individual. Corre archivos individuales solo para iterar rápido en algo puntual, y
  reserva la corrida completa para: (a) el diagnóstico inicial, (b) checkpoints cada
  ~5-8 archivos nuevos, (c) la verificación final.
- No pegues tablas de cobertura completas en el chat. Resume: "N archivos al 100%,
  quedan estos M con esta razón concreta". Detalle solo si el usuario lo pide.
- No repitas explicaciones ya dadas. Si ya se estableció que "branch % en la tabla
  resumen puede ser artefacto, la fuente de verdad es el dato de branch crudo (Fase
  0.4), no el `--detailed` por línea ni el resumen agregado", no lo reexpliques cada
  vez — aplícalo.
- No confíes en que un test "pasó" implica que cerró la cobertura que buscabas. Si el
  test usa un helper que reconstruye/clona la función bajo prueba (ver Fase 3), puede
  pasar perfectamente y no mover un solo bit del reporte del archivo original. Verifica
  el dato de cobertura después de escribir el test, no solo el resultado del test.
- Antes de escribir un test, clasifica el gap (ver Fase 2). Los "no reales" no se
  escriben — se reportan en una línea y se avanza. Perseguir 100% en código muerto es
  el mayor desperdicio de tokens posible en esta tarea.

## Fase 0 — Punto de partida y detección de puntos ciegos

1. Corre el suite completo con cobertura:
   `rm -rf coverage && <runtime> --coverage=coverage` (o equivalente del stack).
2. **Detecta archivos invisibles.** Un reporte de cobertura solo lista archivos que
   ALGÚN test cargó. Si nada importa un archivo, no aparece — ni siquiera como 0%.
   Compara el árbol completo de código fuente contra la unión de archivos que
   aparecen en los datos crudos de cobertura:
   ```
   find src -name "*.ext" -not -path "*/tests/*" | sort > /tmp/all_src.txt
   # extraer urls/paths únicos de los datos crudos de cobertura (json/lcov/etc.)
   comm -23 /tmp/all_src.txt /tmp/covered_src.txt
   ```
   Excluye archivos de solo-tipos/interfaces sin código ejecutable (no aplican).
   Cualquier archivo real que quede en la diferencia es un **gap total** — trátalo
   con prioridad máxima, son los más baratos de encontrar y los que más se esconden.
3. Antes de confiar en el % de "branch" de la tabla resumen: si el mismo módulo se
   carga en muchos archivos de test aislados (contenedores/singletons compartidos),
   esa columna puede no reflejar la unión real de ramas cubiertas. Verifica con el
   reporte detallado por línea (`<coverage-tool> --detailed` o equivalente) antes de
   escribir un test para "cerrar" un % de branch — puede que la línea ya esté al 100%
   y el número sea ruido de agregación entre instancias de módulo aisladas.
4. **El reporte "por línea" no es suficiente para branch — es necesario pero no
   suficiente.** Un archivo puede marcar 100% de línea y seguir teniendo ramas al 0%,
   porque una línea "cubierta" puede contener una sub-rama que nunca se ejecutó:
   ternarios (`a ? b : c`), `||`/`??` de fallback, encadenamiento opcional (`?.`),
   parámetros con valor por defecto, y la selección de rama dentro de un factory
   (`if (options.each) {...} else {...}`) cuando solo se probó una de las dos opciones.
   Antes de declarar un archivo "cerrado", extrae el dato de branch crudo (no el
   resumen agregado) y confírmalo contra el código fuente:
   ```
   # lcov: cada línea BRDA:<line>,<block>,<branch>,<hits> — hits=0 es la rama sin cubrir
   awk '/^SF:.*ruta\/al\/archivo\.ts/{f=1} f&&/^BRDA:/{print} f&&/^end_of_record/{f=0}' coverage/lcov.info
   ```
   (para otros formatos de cobertura, busca el equivalente — istanbul/json expone
   `branchMap`+`b` por statement, go tiene `-covermode=count` por bloque, etc.)
   Si una rama sigue en 0 a pesar de que la línea está "verde", trátala como el resto
   de los gaps: clasifícala en Fase 2 antes de decidir si se cierra o se descarta.

## Fase 1 — Priorización

Ordena los gaps encontrados por impacto, no por archivo:
1. Archivos completamente invisibles (Fase 0.2) — máxima prioridad.
2. Cachés/guards que nunca se cumplen (Fase 2, "código muerto que en realidad es un
   bug") — el hallazgo más valioso posible en esta auditoría, revísalo antes que nada
   apenas lo detectes.
3. Lógica de negocio real sin cubrir (cálculos, validaciones, ramas de error con
   comportamiento distinto).
4. Registro/DI/decoradores: rama de "clase inválida", opciones por defecto,
   short-circuit de guards/middlewares, selección de rama por opciones dentro de un
   factory (aunque esté "escondida" dentro de una línea con line% 100, ver Fase 0.4).
5. Fallbacks de una línea (`||`, `??`, ternarios anidados, callbacks por defecto) en
   código real.
6. Todo lo demás (branches triviales, catch-ignore de infraestructura).

## Fase 2 — Triage por gap (clasifica ANTES de escribir nada)

Para cada línea/rama sin cubrir, decide en una frase:

- **Gap real y aislado** → escribe un test unitario, sin tocar fixtures compartidos.
- **Código muerto por diseño** (el chequeo nunca puede ser cierto dado cómo se llama
  hoy, p.ej. un check sobre un ID recién generado al azar, o una constante hardcodeada
  donde antes había una config) → NO escribas un test para forzarlo. Repórtalo como
  hallazgo y pregunta si se elimina o se refactoriza para que sea alcanzable (a veces
  la corrección correcta es exponer un parámetro opcional que le dé sentido real al
  chequeo, no solo borrarlo — evalúa ambas).
- **Código "muerto" que en realidad es un BUG** — antes de archivar una rama como
  código muerto, pregúntate: *¿esta rama debería dispararse en el uso normal y no lo
  hace por un error de implementación?* Señal típica: una caché/memoización o un guard
  (`if (cache?.key === x) return cache`) que nunca se cumple ni siquiera llamando dos
  veces con los mismos argumentos — revisa si el valor se está comparando contra la
  variable equivocada, o si falta la asignación que debería poblar la caché antes del
  `return`. Si encuentras esto, es un hallazgo de mayor severidad que un gap de test:
  repórtalo aparte y pregunta antes de tocar código de producción (el fix suele ser
  una línea, pero es un cambio de comportamiento real, no solo de tests).
- **Dependiente de entorno real** (lee un archivo de config real, usa un valor cacheado
  a nivel de módulo) → antes de descartarlo, revisa si:
  - la función memoiza en el primer llamado → puedes mockear la primitiva de bajo
    nivel (lectura de archivo, fetch, etc.) ANTES de esa primera llamada, en un
    archivo de test aislado dedicado solo a esa rama.
  - acepta un parámetro opcional de override que el código que la envuelve no expone
    — en ese caso sí es un gap real, solo hace falta pasar por la capa correcta.
  - el fallback que quieres forzar (p.ej. "usa la ruta/cwd por defecto") escribiría o
    leería sobre una ubicación real del proyecto si el entorno real coincide por
    accidente con esa ruta por defecto → nunca lo asumas seguro implícitamente. Redirige
    explícitamente CUALQUIER parámetro de escritura/ubicación a una carpeta temporal y
    usa el stub de entorno (`cwd`, primitiva de red, etc.) solo para decidir la rama, no
    para el efecto colateral. Verifica al terminar con `git status`/diff que no se tocó
    nada fuera de la carpeta temporal.
- **Requiere infraestructura viva** (socket real, servidor real, conexión de red) →
  no lo fuerces con mocks frágiles. Es aceptable dejarlo sin cobertura unitaria si ya
  existe una prueba de integración/e2e real que lo ejercita, aunque sea parcialmente.
  Prioriza reutilizar esa infraestructura real (llamar dos veces al wrapper público con
  una variación mínima de opciones) antes de construir un mock nuevo — es más barato y
  más confiable. Nunca fuerces el fallo de un comando/servicio externo manipulando
  variables de entorno globales del proceso (`PATH`, etc.): no está aislado por test y
  el riesgo de romper otros tests o el propio proceso supera el valor de la rama.
- **Manejo de errores defensivo que nunca debe dispararse** (try/catch envolviendo un
  bootstrap completo, ignorar-y-loguear) → no vale la pena forzar el catch; forzarlo
  suele requerir romper deliberadamente el flujo feliz de todo lo que envuelve.
- **Unidad de cobertura escondida dentro de una línea "cubierta"** (ver Fase 0.4):
  parámetro con callback por defecto (`function f(cb = (x) => x)`), rama de un factory
  seleccionada por opciones (`each`, `optional`, etc.), fallback `||`/`??`/`?.` — estas
  SÍ son gaps reales casi siempre (no son "triviales" solo por ser cortas) y suelen ser
  las más baratas de cerrar una vez identificadas. Clasifícalas igual que cualquier otra
  antes de decidir, pero por defecto tratarlas como cerrables.

## Fase 3 — Patrones de test (aprendidos, aplican a la mayoría de stacks)

- **Aislamiento entre archivos de test**: si el runtime aísla cada archivo de test
  (proceso/worker/módulo fresco por archivo — verifícalo una vez para el stack en
  cuestión), puedes mockear globals (`fetch`, reloj, FS) libremente en un archivo
  dedicado sin miedo a filtrar el mock a otros archivos. Documenta el hallazgo si no
  es obvio, porque cambia todo el approach de mocking. Verificación rápida y barata
  (dos archivos desechables, córrelos juntos y borra):
  ```
  # a.test.ts: setea un global. b.test.ts: lee ese mismo global.
  # Si b lo ve undefined, el runtime aísla por archivo — mockea con confianza.
  ```
- **Los helpers de test que reconstruyen la función bajo prueba NO cuentan para la
  cobertura del archivo original.** Cualquier utilidad que tome una función y genere
  una nueva a partir de su código fuente (`new Function(...)`, `eval`, stringify +
  reemplazo de identificadores — el patrón típico de un helper "mockWrap"/"rewire"
  hecho a mano) crea un objeto de función distinto al que el instrumentador de
  cobertura registró. El test puede pasar perfectamente contra ese clon y la línea
  original del archivo se queda en rojo. Detéctalo así: si escribiste un test con ese
  tipo de helper y la línea/rama que buscabas cerrar sigue en 0 después de correrlo,
  cambia de técnica — stub real de la primitiva de bajo nivel (`stub(Deno, 'cwd', ...)`,
  `stub(globalThis, 'fetch', ...)`, etc.) o llamada directa a la función real con los
  argumentos que fuercen la rama, nunca el helper que clona código.
- **Los factories/decoradores ejecutan la selección de rama al invocarse, no al
  usarse después.** Un patrón como `function Decorator(options) { if (options.each)
  {...} else {...}; return definirComportamiento(...) }` corre su `if/else` en el
  momento en que se llama `Decorator(...)` — típicamente al declarar la clase/objeto
  que lo usa — no cuando el valor decorado se valida o ejecuta más adelante. Aprovecha
  esto: para cerrar cada combinación de opciones basta con invocar el factory con esa
  combinación (p.ej. una clase de fixture con una propiedad por combinación); no hace
  falta disparar el flujo de validación/ejecución completo solo para la cobertura,
  aunque sigue valiendo la pena un caso end-to-end real (uno positivo y uno negativo)
  que confirme el comportamiento, no solo la cobertura.
- **Los callbacks/parámetros con valor por defecto son una unidad de cobertura aparte.**
  `function f(cb = (x) => x) {...}` — si todos los tests pasan su propio `cb`, esa
  función identidad por defecto se queda en 0% de function-coverage aunque `f` esté al
  100%. Para cerrarla, añade un caso que invoque `f` sin ese argumento opcional.
- **Decoradores/registro DI**: para cubrir "clase inválida" o "opciones por defecto",
  no necesitas levantar todo el framework — invoca el decorador como función plana
  contra una clase mínima y verifica el efecto (excepción esperada, o el objeto de
  configuración que termina registrándose vía spy sobre el método real del
  contenedor/registro).
- **Guards/middlewares que cortan el flujo**: regístralos con la API pública real del
  framework (no mockees el contenedor entero) y luego invoca el punto de entrada
  exportado directamente con un contexto mínimo simulado. Evita tocar fixtures
  compartidas por varios tests — usa clases y registros nuevos, locales al test.
- **No puedes reasignar exports nombrados de función** (bindings ESM de solo lectura).
  Sí puedes reasignar métodos de objetos/clases (útil para spy/stub manual con
  guardar-original + restaurar en `finally`, o con el helper `spy`/`stub` del
  framework de test si existe).
- **Seguridad al forzar una rama de fallback "usa el valor/ubicación por defecto":**
  nunca dejes que el efecto colateral de ese fallback caiga en una ruta real del
  proyecto por accidente. Patrón seguro: stubea solo la primitiva que decide la rama
  (`cwd`, resolución de config, fetch) y pasa explícitamente cualquier parámetro de
  escritura/lectura de archivos a una carpeta temporal — así el resultado de la rama
  nunca depende de que el entorno real coincida "por suerte" con algo seguro. Cierra
  cada test de este tipo con `try/finally` restaurando el stub y borrando la carpeta
  temporal, y al terminar el lote corre `git status`/diff para confirmar que ningún
  archivo real del repo cambió.
- **Confía en la corrida real del compilador/test runner por encima de cualquier
  diagnóstico de IDE/LSP en vivo** si contradicen al resultado real — en esta sesión
  los diagnósticos en vivo fueron ocasionalmente inconsistentes con el archivo real.

## Fase 4 — Verificación

- Por archivo nuevo: corre solo ese archivo (rápido) y confirma con el dato de branch
  crudo (Fase 0.4) que la rama que buscabas quedó en verde — no asumas por el "ok" del
  test. Luego formatter + linter si el proyecto los tiene configurados como gate.
- Cada 5-8 archivos, o al terminar un lote temático: corrida completa + cobertura, y
  repite el diff de Fase 0.2 para confirmar que no quedó ningún archivo invisible.
- Si tocaste código de producción (fix de un bug encontrado en Fase 2), corre el
  suite completo antes y después del fix para confirmar que nada que dependía del
  comportamiento roto se quiebra.
- Antes de declarar terminado: vuelve a extraer el listado de archivos por debajo del
  umbral "verde" del proyecto (no solo mirar el resumen coloreado — usa el mismo dato
  crudo de Fase 0.4) y confirma que cada uno que sigue ahí tiene una razón de Fase 2
  explícita, no que simplemente se dejó de revisar. Si algo bajó de puntaje respecto a
  una corrida anterior, no asumas regresión de inmediato — primero verifica si ese
  archivo simplemente no había sido triado a nivel de branch todavía (ver Fase 0.4).
- Si el entorno lo permite, corre `git status`/diff de los archivos que NO son de test
  ni de docs, para confirmar que ningún fallback forzado escribió sobre algo real.
- Al final: una corrida completa, un resumen de números antes/después, y una lista
  corta de lo que quedó fuera con la razón (una línea por ítem, no un ensayo).

## Formato de reporte esperado (para no gastar tokens narrando)

```
Cobertura: X% branch / Y% función / Z% línea (antes: X0/Y0/Z0)
N tests nuevos, M total, todos pasando.

Bugs encontrados vía cobertura (si los hay — reportar aparte, no como gap de test):
- archivo.ts:línea — <qué rama debería dispararse y por qué no lo hace>. ¿Corrijo?

Cerrado:
- archivo.ts: <qué rama/gap se cerró en una frase>

Fuera de alcance (con razón):
- archivo.ts:línea — <código muerto | requiere infra viva | env-dependiente sin punto de inyección>
```
