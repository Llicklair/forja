# /forja — Loop Engineering portable

Un **bucle autónomo de revisión de código** que recorre un proyecto entero en profundidad y mejora el
código sin que teclees: descubre por flujos de ejecución, escribe tests, arregla lo seguro en ramas
aisladas, lo verifica con un evaluador independiente, y abre PRs — **nunca auto-mergea; tú decides**.

Genérico para cualquier stack/repo. Vive en `~/.claude/` y se activa con `/loop forja` (continuo) o
`/forja` (una pasada).

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
**Qué obtienes**: tras la pasada, una rama `loop/forja/auto` + un PR único con fixes verificados
(cada uno con su test/repro), un `tasks/forja/coverage.md` con el % de avance, y lo dudoso drenado a
`tasks/forja/inbox/` para que tú decidas. **Nunca mergea solo.** Requiere el repo en git; GitNexus es
opcional (mejora el marco global). Sin plugin, ver [Instalación → B) Manual](#b-manual-sin-plugin).

---

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

## Los agentes (`agents/`)
- `loop-finder` — explora un flujo con UNA lente; reporta defectos con evidencia. No edita.
- `loop-fixer` — implementa UN arreglo en un worktree; verifica con las gates; commitea. No abre PR.
- `loop-tester` — escribe un test/repro (test-first). Aditivo, auto-verificable.
- `loop-evaluator` — adversarial, otro modelo; ejecuta las gates reales; PASS/REJECT/BLOCKER. El "decir no".

## Instalación

### A) Como plugin de Claude Code (recomendado)
Este repo ES un plugin + su marketplace (`.claude-plugin/plugin.json` + `marketplace.json`).
- **Local (desarrollo):** `/plugin marketplace add <ruta-local-al-repo>` y luego
  `/plugin install forja@forja-marketplace`.
- **Desde GitHub** (tras `git push` a un repo público): `/plugin marketplace add Llicklair/forja`
  (o la URL) y luego `/plugin install forja@forja-marketplace`.
- También por CLI: `claude plugin marketplace add <url-o-ruta>` + `claude plugin install forja@forja-marketplace`.

La skill queda como `/forja` y los 4 subagentes (`loop-finder/fixer/tester/evaluator`) se auto-descubren
y los invoca la skill internamente.

### B) Manual (sin plugin)
```
cp -r skills/forja  ~/.claude/skills/forja
cp agents/loop-*.md ~/.claude/agents/
```

Necesita el repo objetivo en git. GitNexus (`npx gitnexus analyze`) es opcional pero recomendado (da el
marco global por grafo; si no, usa grep).

> Nota: actualiza `homepage`/`repository` en `.claude-plugin/plugin.json` y el `source` del marketplace si
> publicas bajo otro usuario/URL de GitHub.

## Uso
- `/loop forja` — bucle continuo (se auto-reprograma). Para con "para".
- `/forja` — una sola pasada.
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
- **Nada se mergea sin tu OK.**

## Lecciones cosidas al motor (de uso real)
Gate de suite completa (una fixture filtró estado y rompió 11 tests que ningún `-k` veía) · el finder
busca la mitigación existente antes de gritar ALTA (alto recall, baja precisión) · trampa de herencia de
schema (validar solo en Create, no en la Base que hereda Response) · trampa de fixture sin teardown ·
escaneo de hermanas (el bug casi nunca está en un solo sitio) · veto por blast radius · changelog humano
por PR · verificación de lente diversa para high-risk · mapa de cobertura con % honesto.

---
*Inspirado en el "Loop Engineering" (Orange Book). Construido y endurecido con Claude Code.*
