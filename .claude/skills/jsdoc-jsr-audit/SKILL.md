---
name: jsdoc-jsr-audit
description: Audita la precisión del JSDoc existente (no solo cobertura) y baja a cero los errores de "deno doc --lint" en paquetes Deno/JSR — encuentra doc copiado entre símbolos hermanos, defaults/throws desactualizados, y arregla el ciclo de exportar tipos privados hasta que el entrypoint quede limpio. Úsalo en cualquier paquete Deno con deno.json(c) y "exports".
---

Úsalo como prompt inicial en cualquier paquete Deno/JSR (`deno.json(c)` con `"exports"`) que ya
tenga JSDoc pero quieras (a) verificar que sea **correcto y no esté desactualizado**, no solo
"presente", y/o (b) bajar a cero los errores de `deno doc --lint` (cobertura de tipos públicos
que exige JSR para el score 100 y para que la doc generada en jsr.io no tenga huecos).

Esta tarea tiene DOS fases independientes que puedes correr por separado o juntas:
- **Fase A — Auditoría de precisión**: encontrar JSDoc que miente o quedó viejo.
- **Fase B — Cero doc-lint**: exportar/documentar lo que JSR exige para que `deno doc --lint`
  quede en 0 errores.

## Regla de oro (ahorro de tokens)

- Corre `deno doc --lint <entrypoint>` UNA vez, redirige a un archivo temporal, y filtra ANSI con
  `sed -E 's/\x1b\[[0-9;]*m//g'`. Haz todo el triage sobre ese archivo con `grep -oE`/`awk`, no
  releyendo el comando repetidamente.
- Agrupa por regla ANTES de tocar nada: `grep -oE "error\[[a-z-]+\]" | sort | uniq -c`. Esto te da
  el tamaño real del problema (típicamente: `private-type-ref` >> `missing-jsdoc` >
  `missing-return-type`) y evita que ataques archivos al azar.
- Arregla primero `missing-jsdoc`/`missing-return-type` (mecánico, sin efectos en cascada) y deja
  `private-type-ref` para el final (SÍ tiene cascada — ver Fase B).
- Después de cada tanda de cambios, re-corre el doc-lint completo UNA vez y compara el conteo total
  contra la corrida anterior. No expliques cada error individual en el chat; resume
  "N→M errores, quedan estos por categoría".
- Antes de reportar "sin regresiones", compara contra el baseline con `git stash` (corre el lint
  con los cambios guardados, anota el número, haz `git stash pop`). Un LSP/diagnóstico en vivo
  puede mentir; el comando real (`deno check`, `deno doc --lint`, `deno test`) es la fuente de verdad.

## Fase A — Auditoría de precisión del JSDoc existente (no solo cobertura)

El objetivo no es "que tenga comentario", sino "que el comentario sea cierto hoy". Un doc con
`@throws`, defaults o comportamiento inventado/desactualizado es peor que no tener doc.

1. **Identifica la superficie pública real**: lee el archivo de entrada (`mod.ts`/`index.ts`) y
   lista TODOS los símbolos re-exportados. Esa lista es tu alcance — no audites código interno
   que un consumidor del paquete nunca ve.
2. **Divide en categorías temáticas** (clases base/abstractas, decoradores, utils/constantes, tipos
   puros) y lanza un agente `Explore` por categoría EN PARALELO (un solo mensaje con varias
   invocaciones), cada uno con instrucciones explícitas de:
   - Comparar cada claim del doc (parámetros, `@returns`, `@throws`, defaults, orden de ejecución,
     "esto lanza X", "esto hereda Y") contra la implementación real, línea por línea.
   - Reportar SOLO con confianza alta y con `archivo:línea` — nada de especulación.
   - Devolver un reporte corto (400–500 palabras), no transcribir el código.
   - Decirle explícitamente qué NO debe re-reportar si ya sabes de antemano de 1-2 bugs
     encontrados por ti mismo (evita que 4 agentes reporten lo mismo).
3. **Patrones de bug reales que aparecen todo el tiempo** (aprendidos en esta sesión, busca esto
   específicamente):
   - Doc copy-pasteado entre símbolos hermanos (p. ej. el doc de `Guard` con la prosa de `Pipe`,
     o viceversa) — el síntoma es que dos símbolos con comportamiento distinto tienen la MISMA
     frase.
   - `@returns`/`@throws` que describe un valor viejo porque la firma cambió (usa `grep` de la
     firma real, no confíes en el doc para saber qué retorna).
   - Reclamos de orden de ejecución ("se aplica después de X") que en realidad es al revés —
     verifica leyendo el código que orquesta la llamada (el "pipeline" real), no el doc del propio
     decorador.
   - Un decorador de clase (`@Controller`, `@Resolver`, etc.) que valida/lanza si la clase no
     extiende la base correcta, pero eso NO está documentado con `@throws`.
   - Un overload de función/decorador con un campo marcado como opcional en el doc pero requerido
     en el tipo real (o viceversa) — compara `@param [x]` contra la firma TS real.
   - Un tipo público cuyo doc lista campos que ya no existen, o que omite campos nuevos — compara
     la lista de `@property`/prosa contra las keys reales del `type`/`interface`.
   - Bugs de código de verdad que se descubren auditando docs (no asumas que solo hay bugs de
     texto): un `.map()` sin `return` dentro de un `Promise.all` (pierde awaits/rechazos), un
     overload de tipos que exige un campo que el resto de decoradores hermanos no exige.
4. **Al aplicar un fix de código real encontrado en la auditoría** (no solo de doc): corre el test
   suite del área afectada antes y después, y confirma que el fix no cambia comportamiento
   observable salvo el bug corregido.

## Fase B — Cero errores de `deno doc --lint`

### Las 3 categorías típicas y cómo se arreglan

1. **`missing-jsdoc`** — símbolo exportado (incluyendo métodos/constructores/getters de una clase
   exportada, y también miembros `private` de TS — JSR los sigue viendo porque `private` es solo
   compile-time) sin comentario. Fix: una línea de JSDoc explicando qué hace, no qué es obvio del
   nombre.
2. **`missing-return-type`** — función exportada (incluye `const fn: SomeType = (x) => {...}`)
   sin anotación de retorno explícita en la propia expresión de función, aunque el tipo de la
   variable ya lo implique. Fix: anota el retorno directamente en la función (`(x): void => {}`,
   `(x): Promise<void> => {}`).
3. **`private-type-ref`** — un símbolo público referencia un tipo que JSR no puede resolver porque
   NO es alcanzable desde el entrypoint (`mod.ts`). Importante: esto pasa AUNQUE el tipo ya tenga
   `export` en su propio archivo — JSR exige que también esté re-exportado desde el entrypoint. Y
   pasa el doble si el tipo ni siquiera tiene `export` en su archivo de origen (hay que agregarlo).

### Proceso iterativo para `private-type-ref` (el que realmente tiene volumen)

1. Extrae la lista única de tipos "privados" referenciados:
   `grep "error\[private-type-ref\]" | grep -oE "references private type '[^']+'" | sed ... | sort -u`
2. Para cada tipo único, ubica su archivo de origen (`grep -rn "^export type X\|^type X"`).
   Si no tiene `export`, agrégaselo ahí mismo (no lo dupliques ni lo redefinas en otro lado).
3. Agrupa por archivo de origen y añade un bloque `export type { A, B, C } from '...'` en el
   entrypoint, ordenado alfabéticamente dentro del bloque para que sea fácil mantenerlo.
4. **Casos que no son un simple tipo suelto — decide con criterio, no automatices ciegamente**:
   - Clases base abstractas internas que son la superclase real de clases ya públicas (ej. una
     clase pública `extends InternalBase` sin que `InternalBase` esté exportada): expórtalas como
     `export type { InternalBase } from '...'` (solo el tipo, no hace falta el valor si nadie va a
     instanciarla directamente — son abstractas). Es lo correcto, no un parche: si un consumidor
     extiende la clase pública, hereda miembros de esa base y necesita poder nombrarla.
   - Una clase interna sin `export` en absoluto (ej. `class Foo {}` que solo se usa como tipo de
     un singleton exportado, `const publicThing: Readonly<Foo> = ...`): agrégale `export class Foo`
     en el archivo de origen; es un cambio seguro (no cambia runtime, solo hace el tipo nombrable).
   - Tipos que vienen de una librería de terceros (`npm:paquete`) referenciados por un tipo público
     tuyo: NO los persigas exportando el tipo del paquete externo. Suele destapar un grafo de tipos
     internos de esa librería (genéricos con sus propios defaults no exportados) que no controlas
     y que no vale la pena arreglar. Deja ese ÚNICO error como excepción documentada y sigue.
     Cómo confirmarlo: si al re-exportar el tipo externo el conteo de errores SUBE en vez de bajar,
     es la señal de que entraste al grafo interno de un paquete de terceros — revierte esa
     exportación puntual.
5. **Cascada esperada, no es un error tuyo**: cada tipo que pasa de privado a público puede:
   - Necesitar su propio JSDoc (antes no se exigía porque era privado) → nueva tanda de
     `missing-jsdoc`, arréglalos igual que en la Fase B.1.
   - Referenciar a su vez OTRO tipo no alcanzable → nueva tanda de `private-type-ref`.
   Itera: fix en lote → `deno check <entrypoint>` (que no haya typos/exports rotos) →
   `deno doc --lint` → repite. 3-5 rondas es normal para un paquete con ~70 errores iniciales.
6. **Cuidado con el editor de texto en lote**: si usas `sed`/`python` para insertar `export`
   antes de un `type X = ...` en varios archivos a la vez, verifica con `deno check` inmediatamente
   después — un `old_string` de reemplazo demasiado corto puede matchear como substring dentro de
   una línea que ya tenía `export` y duplicarlo (`export /** doc */\nexport type X = ...`), lo cual
   es un `SyntaxError` silencioso hasta que corres el check.

## Verificación final (no te la saltes)

```bash
deno fmt --check <paths>
deno lint <paths>
deno check <entrypoint>
deno test --allow-all   # o el runner del proyecto
deno doc --lint <entrypoint>   # debe bajar a 0, o a las excepciones documentadas de terceros
deno publish --dry-run --allow-dirty   # confirma "Checking for slow types..." sin warnings
```

Para confirmar que no rompiste nada preexistente (y no solo que "no subió el número"):
```bash
git stash && deno doc --lint <entrypoint> 2>&1 | tail -3 && git stash pop
```

## Formato de reporte esperado (para no gastar tokens narrando)

```
Doc-lint: X errores → Y errores (categorías: private-type-ref A→B, missing-jsdoc C→D, missing-return-type E→F)
Excepciones documentadas (no arregladas a propósito):
- <tipo> — referencia un tipo interno de <paquete npm> no exportable sin arrastrar su grafo interno.

Bugs de código reales encontrados en la auditoría (no solo de doc):
- archivo.ts:línea — <qué hacía mal y qué se corrigió, una frase>

Docs corregidos por estar desactualizados (muestra solo los más importantes, no los 30):
- archivo.ts:línea — <qué decía vs qué es cierto ahora, una frase>

Verificación: fmt/lint/check/test OK, doc-lint sin regresión (confirmado con git stash).
```
