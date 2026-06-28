# /forja — Loop Engineering portable

Un **bucle autónomo de revisión de código** que recorre un proyecto entero en profundidad y mejora el
código sin que teclees: descubre por flujos de ejecución, escribe tests, arregla lo seguro, lo verifica con
un evaluador independiente, y te lo entrega para revisar — en **PR** (rama aislada) o **en local sin PR**,
tú eliges. **Nunca auto-mergea ni commitea sin tu OK; tú decides.**

Genérico para cualquier stack/repo. Vive en `~/.claude/` y se activa con `/loop forja` (continuo) o
`/forja` (una pasada).

> 🧱 **¿Construir, no solo revisar?** Este kit incluye la modalidad hermana **`/construye`** (Spec-Driven
> Build): edifica features nuevas desde una spec con el mismo motor verificado. Ver la sección **/construye** abajo.

## Quickstart (≈60s)
Apúntalo a un repo en git y déjalo trabajar:
```
# 1. instálalo como plugin (una vez)
/plugin marketplace add Llicklair/forja
/plugin install forja@forja-marketplace

# 2. dentro del repo que quieras revisar
/forja            # una pasada
# …o…
/loop forja       # bucle continuo (para con "para")
```
**Qué obtienes**: fixes verificados (cada uno con su test/repro) listos para revisar, un mapa de cobertura
con el % de avance, y lo dudoso drenado a un inbox para que tú decidas. **Nunca mergea ni commitea solo.**
Requiere el repo en git; GitNexus es opcional (mejora el marco global). Sin plugin, ver
[Instalación → B) Manual](#b-manual-sin-plugin).

## Modos: con-PR o sin-PR — **el PR es opcional**
El usuario elige cómo entrega el loop; sin decir nada, **recuerda el último modo**:
- **`pr`** (aislado): los fixers trabajan en un worktree (`loop/forja/auto`) y se acumula **un PR** que
  crece (nunca se mergea solo). Para revisar en GitHub. Dispáralo con **"forja con PR"**.
- **`no-pr`** (local): los fixers editan tu **working tree directamente, SIN commitear** — revisas los
  cambios en el panel Source Control de tu IDE y commiteas tú por lotes. Sin PR, sin push. El estado del
  loop vive **fuera del repo** (`~/.claude/forja-state/<repo>/`) para no ensuciar `tasks/`. Dispáralo con
  **"forja sin PR"**.
- En ambos modos: evaluador independiente + **nunca mergea ni commitea sin tu OK**.

> **Seguridad — ejecutar las gates corre código del repo objetivo** (`conftest.py`, scripts de
> `package.json`…) con tus privilegios; la allowlist filtra el *nombre* del comando, no el *payload*. En
> **tu repo / el de tu equipo**: auto-ejecuta. En un **repo ajeno sin auditar**: usa sandbox (red cortada +
> FS limitado al worktree) o trátalo como "verificación no disponible" → el fix va a inbox sin ejecutar.

---

## Comandos (índice)
| Escribe… | Qué hace |
|---|---|
| `/forja` | Una pasada del loop de **revisión** (lente rotativa: bugs · seguridad · concurrencia · errores/borde · test-gaps · perf). |
| `/loop forja` | Revisión **continua** (se auto-reprograma; para con `para`). El completo — más profundo, más tokens. |
| `forja con PR` · `forja sin PR` | Fija el **modo de entrega**: PR aislado, o local sin commitear (revisas en el Source Control del IDE). |
| `forja detector` | Modo **DETECTOR barato**: un detector AST escanea por ~0 tokens y el LLM solo arregla los **detalles atómicos reales** (lo arriesgado → inbox). |
| `loop forja detector` | El modo detector, **continuo**. Lo más barato para pulir detalles atómicos sin disparar tokens. |
| `/clean forja map` | Resetea el mapa de cobertura y **rota a la siguiente lente**. |
| `/construye` | Una feature **Spec-Driven Build**: construir desde una spec (sobre GitHub Spec Kit) con verificación cruzada. |
| `/loop construye` | Construcción **continua**, feature a feature (vía `converge`). |
| `construye con PR` · `construye sin PR` | Modo de entrega del builder (igual que la forja). |
| `para` | Detiene cualquier loop y muestra el resumen consolidado. |

> ⚠️ Para arreglar **detalles atómicos sin gastar tokens**, usa `forja detector` — **no** `/loop forja` (ese
> es el loop completo de 6 lentes). Ver [Modos](#modos-con-pr-o-sin-pr--el-pr-es-opcional) y la sección 1C del SKILL.

## Ejemplo end-to-end (un run real)
Para que sepas **qué esperar** antes de soltarlo, este es un turno real sobre un backend FastAPI +
SQLAlchemy en modo `no-pr` (el loop edita el working tree **sin commitear**; tú revisas y commiteas).

### 0 · Lo que tecleas
```
forja sin PR        # fija el modo de entrega (solo la 1ª vez; luego lo recuerda)
/forja              # una pasada    (o  /loop forja  para el bucle continuo)
```
No hay más config. Requisito: el repo en git. GitNexus opcional (si está, el "marco global" es por grafo;
si no, por grep).

### 1 · Qué hace en UN turno — el sándwich híbrido
Tomemos el flujo real `client_portal` con la lente **L2 (seguridad)**:

1. **MARCO GLOBAL** (orquestador) — mapea el flujo, lista las funciones hermanas del mismo patrón
   (`/auth`, `revoke`, `/me`) y pregunta *"¿quién ya frena esto?"* antes de gritar.
2. **ÁTOMOS · finders** — N `loop-finder` en paralelo, cada uno con la lente. No editan; reportan con
   evidencia. Aquí uno sospecha: `get_current_client_portal` valida el token de portal filtrando solo
   `is_active=True`, **sin comprobar `expires_at`**.
3. **test-first** — `loop-tester` escribe el repro *antes* del fix: token caducado→401, futuro→OK, NULL→OK.
   Un test en rojo = bug real con repro.
4. **fix atómico** — `loop-fixer` aplica el cambio mínimo (`or_(expires_at.is_(None), expires_at > now())`
   en el WHERE) en un worktree aislado, y hace **escaneo de hermanas**: el único otro lookup del patrón
   (`/auth`) ya estaba bien → fix de un solo sitio (no de una instancia suelta).
5. **PUERTA · evaluador** — `loop-evaluator`, en **otro modelo**, corre las gates reales: suite COMPLETA
   (no `-k`), dedup, y veta si el comentario sobre-afirma o si toca una zona NO-EDIT. Dictamen
   **PASS / REJECT / BLOCKER**. Aquí: **PASS** (correcto, mínimo, aditivo, sin romper callers).

### 2 · Lo que ves — un fix verificado
Cada fix llega documentado así (extracto real, condensado):
```
✅ ARREGLADO (working tree, sin commitear) — expires_at no comprobado por-request
  Severidad: media · Evaluador independiente (otro modelo): PASS
  Bug (confirmado 3/3 escépticos): get_current_client_portal valida is_active=True pero
    no expires_at → un token caducado con JWT aún vivo pasa el guard (ventana ≤24h).
  Fix: añade or_(expires_at IS NULL, expires_at > now()) al WHERE. Hermanas: /auth ya OK → 1 sitio.
  Test: tests/test_portal_a_repro.py (caducado→401, futuro→OK, NULL→OK).
```
En `no-pr` el cambio queda **sin commitear** en tu working tree; lo ves en el panel Source Control del IDE.

### 3 · Lo que ves — el inbox (lo que NO se auto-arregla)
Lo arriesgado/fiscal **no se parchea**: se drena a `inbox.md` **con un repro** para que **tú** decidas.
Ejemplos reales de esa misma sesión:
- **ALTA** — el gate de autonomía se bypasea para importes ≤5000 € (asiento/factura/nómina): 4 tools
  hermanas no llaman a `evaluate_autonomy`. *No auto-fix: cambia comportamiento fiscal de todos los tenants.*
- **ALTA (fiscal)** — doble presentación a la AEAT posible: `aeat_presentations` sin `UniqueConstraint`
  → retry/doble-click crea dos registros enviables. *Requiere migración + decisión.*
- **ALTA** — sobreventa de stock por carrera read-modify-write en `transfer()` sin `with_for_update()`.

### 4 · Lo que ves — el mapa de cobertura (`coverage.md`)
Recorre los flujos sin repetir y reporta un **% honesto**, separando amplitud de profundidad:
```
Pasada 1 (amplitud, lente mixta L1/L2) COMPLETA: 65/65 áreas tocadas ≥1 vez. ~16 fixes verificados.
Pasada 2 · lente L3 (concurrencia/idempotencia): baja a "flujo fino" en las áreas críticas.
PROFUNDIDAD real (la métrica honesta): ~10-15% (≈55-60 funciones de ~11.400 símbolos, 1 lente c/u).
```
La ✅ de un área = "barrida 1 flujo con 1 lente", **no** "revisada entera". `/clean forja map` archiva la
pasada, resetea a ⬜ y rota a la siguiente lente.

### 5 · Cómo lo cierras
- **`no-pr`**: revisas los cambios en Source Control y **commiteas tú por lotes** (en el run real: 14 fixes
  commiteados en 2 lotes, checkpoint suite completa 2437 passed).
- **`pr`**: revisas la rama `loop/forja/auto` / el PR en GitHub. **Nunca se auto-mergea.**
- **`para`**: detiene el loop y muestra el resumen consolidado (fixes verificados + items de inbox).

### 6 · Coste real (de este turno)
≈ **203K tokens** el turno completo de `client_portal` (verificación + fix + evaluador, **9 agentes**).
Profundizar cuesta tokens: 1 lente es rápido y superficial; 6 lentes = barrido completo, más caro.

## La idea en una frase
**Átomos para el trabajo; grafo + orquestador para el marco y la puerta.** Sub-agentes atómicos
(baratos, paralelos, reversibles) hacen el trabajo enfocado; el orquestador sostiene la visión global y
la puerta de calidad que a los átomos les falta.

## Los 4 pilares
1. **Lentes** (6, rotativas o en paralelo): correctitud · seguridad · concurrencia/idempotencia ·
   errores/borde · test-gaps · perf/recursos. La misma rebanada de código mirada de 6 maneras —cada lente
   *enfoca* el reconocimiento de un tipo de bug.
2. **Verificadores** (generador ≠ evaluador, test-first, puerta de suite COMPLETA): convierten "el finder
   cree" en "está probado". Sin esto sería humo.
3. **Marco global** (GitNexus o grep: blast radius + escaneo de hermanas + mitigación existente +
   jerarquía de tipos): la visión sistémica que ve patrones e interacciones que el átomo no.
4. **Mapa de cobertura** (`coverage.md`): recorre los flujos de forma sistemática, sin repetir, con un %
   honesto de progreso.

## El sándwich híbrido (cada turno)
**MARCO GLOBAL → ÁTOMOS → PUERTA GLOBAL**
- *Marco* (orquestador): mapa del flujo, hermanas del patrón, ¿quién ya lo frena?, re-graduar severidad.
- *Átomos*: N finders (1-6 lentes en paralelo) + fixers/testers, informados por el marco.
- *Puerta por lote*: suite COMPLETA + dedup + crítico del lote como conjunto. Aquí caen las fugas que un
  `-k` no ve.

## Gates duras (bloquean el lote — reglas, no consejos)
- **Zona NO-EDIT (fiscal/contable/nómina · cripto/pagos)**: el fixer tiene **PROHIBIDO editar** cálculo
  fiscal (modelos AEAT, IVA, retenciones, asientos, numeración correlativa, IRPF/IS) y pagos/cripto.
  Reporta a inbox **con un test que demuestra el problema** — un LLM no es asesor fiscal. El evaluador hace
  **REJECT automático** si un fix toca un fichero NO-EDIT. Cambiar una cifra fiscal "porque parece más
  correcta" es justo lo que el loop NO debe hacer.
- **Barrido por-clase**: antes de cerrar el lote, por CADA patrón arreglado el orquestador grepea TODO el
  repo y prueba `nº sitios del patrón == saneados + inbox`. "Lo arreglé" exige la **lista exhaustiva del
  grep**, no el sitio tocado; un fix de UNA instancia sin esa prueba → REJECT. (El bug casi nunca está en
  un solo sitio: `delete→409` aparecía en 5 entidades.)
- **No sobre-afirmar**: un `SELECT…then-check` sin `with_for_update()`/UNIQUE protege el caso SECUENCIAL,
  no el concurrente. El evaluador REJECT si un comentario promete garantía concurrente que el mecanismo no
  da; se reescribe a la verdad ("idempotente en secuencial; la carrera real → inbox") o se pone el lock.
- **Suite COMPLETA por lote** (no `-k`): las fugas entre tests (estado compartido / fixtures) solo aparecen
  corriendo la suite entera.

## Los agentes (`agents/`)
- `loop-finder` — explora un flujo con UNA lente; reporta defectos con evidencia. No edita.
- `loop-fixer` — implementa UN arreglo en un worktree; verifica con las gates; commitea. No abre PR.
  **Prohibido tocar zonas NO-EDIT** (fiscal/pagos): esas van a inbox con repro, no se parchean.
- `loop-tester` — escribe un test/repro (test-first). Aditivo, auto-verificable.
- `loop-evaluator` — adversarial, otro modelo; ejecuta las gates reales; PASS/REJECT/BLOCKER. El "decir no".
  **REJECT automático** si un fix toca NO-EDIT o si un comentario sobre-afirma garantía concurrente.

## Modalidad hermana: `/construye` — Spec-Driven Build (la forja que edifica)
La forja **revisa** lo que existe; **`/construye`** **edifica** lo que falta desde una **especificación**.
Misma regla de oro: evaluador independiente, gate de suite, **nunca auto-merge**.

No reinventa el flujo: adopta **[GitHub Spec Kit](https://github.com/github/spec-kit)** para la mitad
delantera (`constitution → spec → clarify → plan → tasks`) y le injerta el motor verificado de la forja en
`/speckit-implement` — el hueco que Spec Kit deja a propósito ("external verifier handles verification"):
- **test-first de ACEPTACIÓN**: los criterios de la spec se vuelven un test en rojo → construir hasta verde.
- **generador≠evaluador** en **otro modelo**: verifica que cumple la spec, sin auto-aprobarse.
- **gate de suite COMPLETA** (no romper lo existente), **barrido por-clase**, zonas **NO-EDIT**.
- **`/speckit-converge`** para brownfield: mapea intención→código **atado al scope del plan** (no al repo
  entero) y añade solo lo que falta → **viable en proyectos grandes, feature a feature**.

Requisito: Spec Kit instalado
(`uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude`).
Úsalo con **`/construye`** (una feature/lote) o **`/loop construye`** (continuo); modos `pr`/`no-pr` igual
que la forja. Detalle en [`skills/construye/SKILL.md`](skills/construye/SKILL.md); hook opcional en
[`integrations/speckit-after-implement.yml`](integrations/speckit-after-implement.yml).

## Instalación

### A) Como plugin de Claude Code (recomendado)
Este repo ES un plugin + su marketplace (`.claude-plugin/plugin.json` + `marketplace.json`).
- **Local (desarrollo):** `/plugin marketplace add <ruta-local-al-repo>` y luego
  `/plugin install forja@forja-marketplace`.
- **Desde GitHub** (tras `git push` a un repo público): `/plugin marketplace add Llicklair/forja`
  (o la URL) y luego `/plugin install forja@forja-marketplace`.
- También por CLI: `claude plugin marketplace add <url-o-ruta>` + `claude plugin install forja@forja-marketplace`.

Las skills quedan como `/forja` (revisar) y `/construye` (edificar, Spec-Driven Build); los 4 subagentes
(`loop-finder/fixer/tester/evaluator`) se auto-descubren y los invocan internamente.

### B) Manual (sin plugin)
```
cp -r skills/forja      ~/.claude/skills/forja
cp -r skills/construye  ~/.claude/skills/construye   # modalidad Spec-Driven Build (opcional)
cp agents/loop-*.md     ~/.claude/agents/
```

Necesita el repo objetivo en git. GitNexus (`npx gitnexus analyze`) es opcional pero recomendado (da el
marco global por grafo; si no, usa grep).

> Nota: actualiza `homepage`/`repository` en `.claude-plugin/plugin.json` y el `source` del marketplace si
> publicas bajo otro usuario/URL de GitHub.

## Uso
- `/loop forja` — bucle continuo (se auto-reprograma). Para con "para".
- `/forja` — una sola pasada.
- **"forja con PR"** / **"forja sin PR"** — fija el modo de entrega (ver **Modos**). Sin decirlo, usa el último.
- `/clean forja map` (o "resetea el mapa") — archiva la pasada, vuelve la cobertura a ⬜, pasa a la
  siguiente lente.
- **Lentes en paralelo** — elige cuántas (1-6) se barren por flujo. 6 = barrido completo por visita (la
  ✅ del mapa significa "revisado por las 6 lentes"); 1 = rápido y superficial. Más coste, más profundidad.

## Honestidad (léelo)
- El loop **converge / reduce**, no **elimina**. Sube el suelo: caza una clase grande de defectos
  detectables, arregla los seguros, y **escala los delicados** (fiscal/dinero/legal) para que tú decidas.
- El % de cobertura por ÁREAS es **amplitud de primer contacto**, no profundidad. Una ✅ con 1 lente ≠
  área revisada. Reporta siempre el matiz.
- Lo no testeable con la infra (concurrencia que pide BD real, RLS, locks) se etiqueta **"correcto por
  construcción, sin prueba empírica"**, no "verificado".
- **Nada se mergea ni se commitea sin tu OK.** En modo `no-pr` los cambios quedan sin commitear para que
  los revises y commitees tú; en `pr`, en un PR que nunca se auto-mergea.

## Lecciones cosidas al motor (de uso real)
Gate de suite completa (una fixture filtró estado y rompió 11 tests que ningún `-k` veía) · el finder
busca la mitigación existente antes de gritar ALTA (alto recall, baja precisión) · trampa de herencia de
schema (validar solo en Create, no en la Base que hereda Response) · trampa de fixture sin teardown ·
escaneo de hermanas (el bug casi nunca está en un solo sitio) · veto por blast radius · changelog humano
por PR · verificación de lente diversa para high-risk · mapa de cobertura con % honesto.

---
*Inspirado en el "Loop Engineering" (Orange Book). Construido y endurecido con Claude Code.*
