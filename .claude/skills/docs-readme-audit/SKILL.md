---
name: docs-readme-audit
description: Deja el README, docs/ y CHANGELOG de una librería Deno/JSR completos, coherentes y profesionales en una sola pasada exhaustiva — cross-check de símbolos contra exports reales, integridad de links/anchors, cobertura completa exports→docs, y validación opcional contra un consumidor real en producción. Úsalo cuando te pidan mejorar/completar/profesionalizar documentación de un paquete.
---

Úsalo como prompt inicial en cualquier paquete Deno/JSR (`deno.json(c)` con `"exports"`) cuando
quieras dejar su README, `docs/` y `CHANGELOG.md` completos, coherentes y profesionales de una sola
pasada — sin la ronda de "¿ya está? ¿seguro que no falta nada?" que dispara re-lecturas repetidas.
Esta skill existe justamente para evitar ese desperdicio de tokens: haz TODAS las verificaciones de
abajo en la primera pasada, no una por turno.

Puedes correr esto como tarea completa, o invocar solo la Fase que necesites (ej. "solo Fase 4" para
una verificación de coherencia sin tocar contenido).

## Regla de oro (ahorro de tokens)

- **Exhaustividad de una sola vez, no iterativa.** El usuario no debería tener que preguntar "¿y no
  falta nada más?" tres veces para que aparezcan cosas reales. Antes de decir "listo", corre TODAS
  las verificaciones de la Fase 1 y la Fase 4 en un solo barrido — son baratas (grep/scripts, no
  relectura manual) y cada una históricamente ha encontrado bugs reales, no cosméticos.
- **Verifica programáticamente, no leyendo con los ojos.** Un link roto, un nombre de símbolo mal
  escrito en un ejemplo, o un ejemplo que no compila contra la firma real, se encuentran con un
  script de 10 líneas (ver Fase 4), no releyendo el archivo cinco veces. Escribe el script una vez,
  córrelo, arregla lo que reporte. Esto incluye verificar que los **anchors** (`#seccion`) resuelvan
  con el algoritmo de slug real del renderer (ver Fase 4) — no asumas que colapsa espacios/símbolos
  igual que tu primera intuición; un heading con "&" u otro caracter especial puede generar un slug
  con guiones dobles que rompe el link si lo calculas mal.
- **No documentes lo que no puedes verificar contra el código fuente real.** Si vas a escribir un
  ejemplo de uso, ábrelo contra la implementación real (tipos, firma, comportamiento) o contra un
  test/fixture existente — nunca "a partir de lo que suena plausible". Esta sesión encontró bugs
  reales de código (no solo de doc) precisamente por verificar en vez de asumir.
- **Resume, no transcribas.** Reporta hallazgos como lista corta con `archivo:línea`, no vuelques el
  archivo completo en el chat en cada paso.
- **`deno fmt` hace el wrap, tú no.** Si `deno.json(c)` tiene `"fmt"` con `proseWrap` y no excluye
  `.md`, escribe el markdown en párrafos naturales y corre `deno fmt` al final — no envuelvas líneas
  a mano, es más lento y menos consistente.
- **Antes de "corregir" algo que parece un artefacto de una edición previa, confirma que de verdad
  lo es.** Si el usuario o una edición anterior agregó contenido que no reconoces del todo, no
  asumas que es un error tuyo o un accidente — pregunta o verifica antes de borrarlo. Un `git diff`
  que muestra dos cambios adyacentes en la misma región no prueba que sean el mismo cambio.

## Fase 0 — Alcance y punto de partida

1. Lee el `README.md` actual completo, el `CHANGELOG.md`, y lista lo que ya existe en `docs/`.
2. Lee el archivo de entrada (`mod.ts`/`index.ts`) y extrae la lista COMPLETA de símbolos
   re-exportados (clases, funciones, constantes, tipos). Esta es tu superficie de verdad — todo lo
   demás se mide contra esta lista.
3. Pregunta el alcance solo si es genuinamente ambiguo (ej. "¿el README ya tiene un formato/tono que
   deba respetar, o puedo reestructurar libremente?"). Si el usuario dice "no cambies el formato
   innecesariamente", trátalo como: sí puedes reordenar/recortar/segmentar contenido y arreglar
   bugs, pero no reescribas el tono o la estructura de secciones que ya funciona.

## Fase 1 — Auditoría exhaustiva del README (todo en una pasada)

No esperes a que el usuario pregunte "¿está completo?" para encontrar estas cosas — búscalas todas
ahora:

1. **Cross-check de símbolos**: extrae con un script (regex sobre bloques de código con
   `import { A, B, C } from '...'`) cada identificador mencionado en ejemplos del README, y
   compáralo contra la lista real de exports de la Fase 0. Cualquier nombre que no matchee es un
   typo real (ej. `ZanixAsyncmqProvider` vs `ZanixAsyncMQProvider` — un solo carácter, invisible a
   simple vista, encontrado solo por el cross-check automatizado).
2. **Cada ejemplo de código debe reflejar la API real**: si un ejemplo llama `objeto.metodo(x)`,
   verifica contra la firma real qué es `x` — un error común es pasar el tipo/nombre de algo en vez
   del identificador que el método realmente espera (ej. `manager.start('rest')` cuando `start()`
   espera el ID que devolvió `create()`, no el string del tipo).
3. **Links y anchors**: todo `[texto](ruta)` interno debe resolver a un archivo real, y todo
   `[texto](ruta#seccion)` debe resolver a un heading real en ese archivo — verifica ambos con
   script (ver Fase 4), no visualmente. Revisa también badges (shields.io, etc.): la URL del link
   del badge a veces no coincide con la URL de la imagen (typos tipo `@org/proyecto` vs
   `@org/proyecto2`).
4. **Tabla de contenidos vs. headings reales**: cuenta los `##` del documento y compáralos 1:1 contra
   la TOC — un heading nuevo (ej. `## Changelog`) que nunca se agregó a la TOC es un bug silencioso
   común.
5. **Ejemplo principal = patrón idiomático real, no el "escape hatch"**: si el proyecto tiene un
   patrón recomendado (decoradores, builder, etc.) y también una API de bajo nivel para casos
   avanzados, el ejemplo de "Basic Usage"/"Quickstart" debe mostrar el patrón recomendado — no el de
   bajo nivel. Verifica esto explícitamente comparando el ejemplo principal contra lo que el resto
   de la documentación (y el código de tests reales) trata como "la forma normal de hacerlo".
6. **Redundancia real vs. duplicación intencional**: un catálogo largo de imports/features, o una
   tabla completa (ej. de variables de entorno), que repite contenido ya cubierto palabra por
   palabra en otra sección o en `docs/`, debe recortarse a una versión compacta con link — no lo
   dejes duplicado en dos lugares. Un ejemplo corto repetido como teaser en dos lugares (README +
   guía) SÍ está bien; una tabla o catálogo completo duplicado NO — revisa esto explícitamente
   comparando cada tabla del README contra las de `docs/`, no solo al escribir contenido nuevo.
7. **Orden lógico**: pasos de instalación/setup que quedaron "sueltos" entre dos secciones no
   relacionadas deben moverse junto a la sección de Installation. Una sección sin numerar que
   interrumpe una secuencia numerada (1, 2, [sin número], 3) rompe la lectura — muévela al final.
8. **Consistencia de estilo**: emojis en headings — o se usan en todos los headings del mismo nivel,
   o en ninguno; no mezclado. Headings con dos puntos finales (`## Título:`) inconsistentes con el
   resto del documento. Blockquotes de advertencia/nota sin marcador (ℹ️/⚠️) cuando el resto del
   documento sí los usa para el mismo tipo de aviso. Frases de relleno ("consulta la documentación
   completa para más ejemplos") justo antes de una sección que ya lo hace explícitamente con links
   reales — bórralas.
9. **Nota de compatibilidad/requisitos**: si el CHANGELOG u otra fuente interna menciona un requisito
   de versión (ej. "compatible con Deno 2.9"), y ese requisito no aparece en ningún lugar visible
   para alguien instalando por primera vez, agrégalo bajo Installation.
10. **No inventes badges**: si vas a sugerir un badge de CI/coverage, verifica primero que exista un
    workflow real que lo respalde (`.github/workflows/`) — si no existe, no lo agregues ni lo
    sugieras como si existiera.
11. **Diagramas ASCII vs. prosa**: si hay un diagrama de arquitectura, confirma que lo que dibuja
    (flechas, cajas conectadas) no contradiga lo que el texto de al lado afirma explícitamente — un
    diagrama que solo dibuja 2 de 4 conexiones mencionadas en el texto es una fuente de confusión
    real, no solo un detalle estético.

## Fase 2 — Cobertura completa en `docs/` (mapeo exports → documentación)

1. Con la lista de exports de la Fase 0, agrúpalos por categoría temática natural del dominio (para
   una librería de servidor: Handlers, Middlewares, Dependency Injection, Configuration, Errors,
   Utilities — ajusta las categorías al dominio real de la librería que estés documentando).
2. Para cada símbolo exportado, verifica que tenga un hogar documentado en alguna guía. Si no lo
   tiene, decide: ¿es lo bastante importante/usado como para merecer una sección propia, o es
   plumbing interno de bajo nivel usado sobre todo por el propio framework (ej. una función que solo
   se llama a sí misma en el ciclo de vida interno)? Documenta lo primero con ejemplo completo; lo
   segundo con una tabla de referencia compacta (una fila por símbolo, sin ejemplo extenso) — no le
   des el mismo peso a ambos.
3. **Segmenta, no dupliques**: cuando recortes contenido del README (Fase 1.6), el detalle recortado
   va a `docs/`, no se pierde. Verifica con el mismo script de la Fase 1.1 que el contenido movido
   sigue siendo preciso en su nuevo lugar.
4. **Enlaza en ambas direcciones**: cada guía nueva de `docs/` debe tener una sección "See also" /
   "Ver también" enlazando a las guías relacionadas, y el README debe enlazar a todas desde su
   sección de Documentation.
5. **Sé explícito sobre el origen de cada pieza en los ejemplos**: si un ejemplo usa una clase
   concreta que no es parte de esta librería (viene de una librería hermana o la escribe el usuario),
   dilo explícitamente — no dejes que el lector asuma que viene de la librería que estás
   documentando. Verifica el origen real (import real en un consumidor, o el propio código fuente)
   antes de afirmarlo.
6. **Actualiza o extiende una guía existente antes de crear una nueva** cuando el contenido nuevo
   pertenece temáticamente a una guía ya escrita — evita fragmentar en demasiados archivos pequeños.
7. **Verifica también los defaults y las restricciones cruzadas entre símbolos relacionados**: si
   documentas tres variantes de un mismo concepto (ej. tres decoradores que aceptan las mismas
   opciones), confirma en el código que las tres de verdad aceptan los mismos valores — un valor de
   enum válido para dos pero excluido por tipo para la tercera es un gap fácil de pasar por alto si
   solo miras un ejemplo a la vez.

## Fase 3 — CHANGELOG y versión

1. Agrega una entrada nueva siguiendo el formato existente (normalmente Keep a Changelog:
   Added/Changed/Fixed/Removed/Deprecated/Security). Agrupa por esas categorías, no por
   archivo-tocado.
2. Si hubo bugs de código reales corregidos durante la auditoría (no solo de texto), decláralos en
   `Fixed` con una frase que explique el síntoma real, no solo "se corrigió X".
3. Decide el bump de versión según el impacto real: solo docs/tipos nuevos exportados sin romper
   nada = minor; bugs de comportamiento corregidos = patch o minor según el proyecto lo venga
   haciendo (revisa el historial del CHANGELOG para inferir la convención que ya siguen). Si el
   trabajo de la sesión aún no se ha publicado/commiteado, no crees múltiples bumps de versión para
   distintas rondas de la misma sesión — todo cabe en una sola entrada de versión.
4. Actualiza el campo `version` real en `deno.json(c)` para que coincida con la entrada nueva del
   CHANGELOG.

## Fase 4 — Verificación automatizada (correr todo junto, no por partes)

```bash
# 1. Formato
deno fmt --check README.md CHANGELOG.md docs/*.md

# 2. Integridad de links internos (una vez, para TODOS los .md a la vez)
python3 - <<'EOF'
import re, os
files = ['README.md'] + [f'docs/{f}' for f in os.listdir('docs')]
ok = True
for f in files:
    content = open(f).read()
    base = os.path.dirname(f)
    for m in re.finditer(r'\]\(([^)]+)\)', content):
        link = m.group(1)
        if link.startswith('http') or link.startswith('#'):
            continue
        target = os.path.normpath(os.path.join(base, link.split('#')[0]))
        if not os.path.exists(target):
            print(f"BROKEN in {f}: {link} -> {target}")
            ok = False
print("All internal links resolve." if ok else "Some broken.")
EOF

# 3. Integridad de anchors (#seccion) contra headings reales — usa el algoritmo de slug real
#    (github-slugger: minúsculas, quita puntuación excepto espacios/guiones, CADA espacio -> UN
#    guion SIN colapsar corridas de espacios; un "&" puede dejar dos espacios seguidos -> "--")
python3 - <<'EOF'
import re, os

def slugify(h):
    h = h.lower().strip()
    h = re.sub(r'[`*]', '', h)
    h = re.sub(r'[^\w\s-]', '', h)
    h = re.sub(r'\s', '-', h)  # sin '+': no colapsa espacios consecutivos
    return h

files = ['README.md'] + [f'docs/{f}' for f in os.listdir('docs')]
headings = {f: {slugify(h) for h in re.findall(r'^#{1,6}\s+(.+)$', open(f).read(), re.MULTILINE)} for f in files}

ok = True
for f in files:
    content = open(f).read()
    base = os.path.dirname(f)
    for m in re.finditer(r'\]\(([^)]+)\)', content):
        link = m.group(1)
        if '#' not in link or link.startswith('http'):
            continue
        path, frag = link.split('#', 1)
        target_file = f if not path else os.path.normpath(os.path.join(base, path))
        if target_file in headings and frag not in headings[target_file]:
            print(f"ANCHOR MISMATCH in {f}: #{frag} -> {target_file}")
            ok = False
print("All anchors resolve." if ok else "Some anchors mismatched.")
EOF

# 4. Cross-check de símbolos en ejemplos vs exports reales (adaptar el regex al lenguaje/paquete)
python3 - <<'EOF'
import re
readme = open('README.md').read()
mod = open('mod.ts').read()  # o el entrypoint real
blocks = re.findall(r"import\s*\{([^}]+)\}\s*from\s*'jsr:@scope/pkg", readme)
names = {n.strip() for b in blocks for n in b.split(',') if n.strip()}
exported = set(re.findall(r'export\s*\{\s*([^}]+)\}', mod))
exported_names = set()
for block in exported:
    for item in block.split(','):
        item = item.strip()
        exported_names.add(item.split(' as ')[1].strip() if ' as ' in item else item)
missing = [n for n in names if n not in exported_names]
print("Missing from real exports:", missing)
EOF

# 5. Tipos, tests, y (si es JSR) doc-lint + slow-types
deno check <entrypoint>
deno test --allow-all
deno doc --lint <entrypoint>          # debe ser 0, o solo excepciones documentadas de terceros
deno publish --dry-run --allow-dirty  # confirma "Checking for slow types..." sin warnings
```

Para confirmar que un conteo (ej. errores de `doc --lint`) no es una regresión y no solo "no subió":
`git stash && deno doc --lint <entrypoint> 2>&1 | tail -3 && git stash pop`.

## Fase 5 (OPCIONAL) — Validar contra un proyecto real en producción

**No lo hagas por defecto.** Solo si el usuario lo pide explícitamente, o pregúntale una vez: "¿hay
algún proyecto real en producción que use esta librería y que pueda revisar para validar que la
documentación es coherente con el uso real?" Si no hay o no lo pide, no ofrezcas este paso de más.

Si sí hay proyecto(s) reales:

1. Verifica qué versión de la librería tiene pineada ese consumidor — si es más vieja que la que
   estás documentando, confirma que los cambios entre versiones sean aditivos (no rompan lo que vas
   a validar) antes de asumir que el comportamiento es el mismo.
2. Busca específicamente:
   - Combinaciones de opciones de API usadas en producción que tu doc nunca muestra.
   - Convenciones de nombre de archivo/módulo reales vs. las documentadas — y si difieren, averigua
     SI es la librería la que las exige o una herramienta satélite (ej. un CLI/bootstrapper
     separado) — no asumas que la librería que documentas es la que impone la convención.
   - Patrones de acceso a instancias/dependencias realmente usados (¿usan solo el getter singular
     documentado, o hay un getter dinámico/plural para el caso de múltiples dependencias que tu doc
     nunca mencionó?).
   - Middlewares: ¿se usa el primitivo base directo, o casi siempre un decorador de más alto nivel
     construido sobre él por un paquete interno de la organización? Si es lo segundo, tu doc debe
     decir explícitamente que ese es el patrón esperado en la práctica.
   - Manejo de errores, variables de entorno/constantes realmente seteadas.
   - El origen real de cualquier clase concreta usada en un ejemplo (¿viene de esta librería, de una
     hermana, o la escribió la propia app?) — no lo asumas, grep el import real.
3. **Alcance estricto**: el objetivo es validar/completar la doc de ESTA librería, no documentar las
   librerías satélite que el proyecto real también usa. Si un patrón real pertenece a otra librería,
   anótalo como "fuera de alcance, vive en `<paquete>`" y sigue — no le escribas una guía aquí.
4. **Si un hallazgo resulta ser un bug de comportamiento real de la librería** (no solo un gap de
   doc) — por ejemplo, un decorador que no aplica correctamente algo que otro decorador hermano sí
   aplica — **detente y pregunta al usuario** antes de parchear el código. Corregir documentación
   procede de forma autónoma; cambiar comportamiento en tiempo de ejecución es una decisión del
   usuario, incluso si el fix es obvio y de una línea. Verifica el hallazgo con rigor antes de
   reportarlo (lee la implementación completa del camino de código involucrado, no solo el síntoma)
   — un "parece un bug" mal verificado cuesta más tokens que confirmarlo bien una vez. Y no asumas
   que la conclusión inicial es correcta solo porque suena razonable: si al implementar el fix
   descubres que no resuelve lo que creías (ej. el registro funciona pero el consumo en runtime
   nunca lee ese registro), dilo explícitamente y reconsidera — no dejes un "fix" a medias que da
   falsa confianza.

## Formato de reporte esperado (para no gastar tokens narrando)

```
README: X→Y líneas. Bugs reales corregidos: <lista corta con línea>.
docs/: N guías (nuevas: ..., extendidas: ...). Cobertura: todo símbolo exportado tiene hogar
  documentado, excepto <lista con razón, si aplica>.
CHANGELOG: entrada [X.Y.Z] agregada con <categorías>.
Verificación: fmt/links/anchors/símbolos/check/test/doc-lint — todo limpio (o excepciones
  documentadas: ...).

Validación contra proyecto real (si se hizo):
- Confirmado: <lo que coincidió con la doc>.
- Gaps encontrados y corregidos: <archivo:línea del proyecto real → qué se agregó/corrigió en docs>.
- Posible bug de comportamiento (requiere tu decisión): <descripción + dónde está el código>.
```
