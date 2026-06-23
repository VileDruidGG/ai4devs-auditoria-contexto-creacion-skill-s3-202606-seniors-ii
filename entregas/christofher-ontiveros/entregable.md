# Entregable · Sesión 3 — Copilotos IA

- **Nombre / usuario:** Chris / VileDruidGG
- **Fecha de entrega:** 22 Junio 2026
- **Repo auditado en la Parte A** (solo tipo/contexto, NO el código): app móvil iOS en Swift (EquiPay), arquitectura modular por features con capas data/domain/presentation y multi-entorno por schemes; con una app Android en Kotlin + Jetpack Compose planeada para paridad.

---

## 1. Hallazgos de la auditoría (Parte A)

> 3-5 cosas que el agente **no pudo inferir** del código y que tendrías que decirle explícitamente.
> Redáctalas para que otra persona las entienda sin contexto adicional. **Sin código propietario ni secretos.**

1. **Hay que elegir un _scheme_ para correr la app; no basta con "Run".** El proyecto define tres schemes en Xcode (Dev, Stage, Prod) y cada uno carga un `.xcconfig` distinto que decide contra qué API apunta la app (`API_BASE_URL`) y en qué entorno corre (`APP_ENV`). Hay que seleccionar el scheme del entorno deseado y un simulador con iOS 17+. Arrancar con el scheme equivocado conecta la app al backend equivocado.

2. **Dónde va cada archivo nuevo no se puede adivinar mirando el código existente.** Varios módulos "compartidos" están vacíos a propósito (estructura preparada, no código muerto). Reglas: dentro de cada feature, el código va en tres capas (`data/`, `domain/`, `presentation/`); los modelos de dominio compartidos (User, Group, Expense, Settlement) van en `SharedDomain`; la UI reutilizable sin lógica va en `DesignSystem`; las utilidades en `Core` y los feature flags en `FeatureFlags`; la navegación entre pestañas usa `selectedTabIndex` del módulo `MainTab`. El error típico es dejar todo dentro del feature.

3. **Hay un "error" de diseño que es intencional y NO se debe corregir.** El módulo `Home` importa directamente al módulo `Groups` para reutilizar `GroupDetailView`. Esto rompe la regla de "cada feature es independiente", pero es deuda técnica conocida y aceptada temporalmente (se resolverá moviendo la vista a `SharedDomain`). No hay que "arreglar" ese import ni replicar el patrón en otros features.

4. **Las convenciones existen para mantener paridad con una futura app de Android.** Los nombres públicos y la arquitectura están pensados para parecerse a una app Android (Kotlin + Jetpack Compose) que aún no existe. Por eso hay que evitar meter patrones que solo tengan sentido en iOS/Swift en las interfaces públicas de los módulos.

---

## 2. SKILL.md de la skill creada (Parte B)

````markdown
---
name: commit-conventional
description: Genera un mensaje de commit siguiendo Conventional Commits a partir de los cambios que están en staging. Úsala siempre que el usuario pida "un mensaje de commit", "escribe el commit", "commit message", "¿cómo titulo este commit?", o cuando esté a punto de commitear cambios ya preparados con git add. Detecta el tipo (feat, fix, refactor, etc.) a partir del diff y añade la clave del ticket de Jira si aparece en el nombre de la rama. NO commitea por su cuenta: solo propone el mensaje.
---

# Mensaje de commit con Conventional Commits

Tu trabajo es leer los cambios que ya están en _staging_ y proponer **un** mensaje
de commit bien formado. Solo propones el texto; el usuario decide si commitea.

## 1. Reúne el contexto antes de escribir

Ejecuta estos comandos (solo lectura) y básate en su salida real, no en suposiciones:

```bash
git diff --staged          # qué cambió exactamente
git branch --show-current  # para sacar la clave del ticket
git log -n 5 --oneline     # para imitar el idioma y estilo del repo
```

Si `git diff --staged` viene vacío, avisa que no hay nada en staging y detente
(sugiere `git add` primero). No inventes cambios que no estén en el diff.

## 2. Formato

```
<tipo>(<scope>): <asunto>

<cuerpo opcional, una idea por viñeta>

<footer opcional: Refs / BREAKING CHANGE>
```

### Tipos permitidos

- `feat` — nueva funcionalidad para el usuario
- `fix` — corrección de bug
- `refactor` — cambio interno sin alterar comportamiento

Elige el tipo a partir de lo que **predomina** en el diff.

### Reglas del asunto (primera línea)

- En modo imperativo: "agrega", "corrige", "elimina" (no "agregué"/"agregando").
- Minúscula inicial, sin punto final.
- Máximo ~72 caracteres.

### Cuerpo (opcional)

- Inclúyelo solo si el cambio no se explica solo con el asunto.
- Explica **qué** y **por qué**.

### Footer: ticket de Jira

- Si el nombre de la rama contiene una clave tipo `AXLG-447`, `ABC-123`
  (mayúsculas + guion + número), añádela como `Refs: AXLG-447`.
- Si no hay clave en la rama, no inventes una; omite el footer.
- Para cambios que rompen compatibilidad, añade `BREAKING CHANGE: <descripción>`.

## 4. Qué NO hacer

- No ejecutes `git commit` ni `git add`; solo entregas el texto del mensaje.
- No pegues la lista de archivos como cuerpo del commit.
- No inventes cambios, tickets ni motivos que no se vean en el diff/rama.

## 5. Salida

Devuelve el mensaje dentro de un bloque de código listo para copiar. Nada más.
````

---

## 3. Diario de decisiones

_Skill creada:_ `commit-conventional` — genera mensajes de commit en formato Conventional Commits a partir de los cambios en staging, añadiendo la clave de ticket de Jira cuando está en el nombre de la rama.

_Decisiones de diseño tomadas:_

- **Decisión 1 — "Solo propone, no commitea".** La skill nunca ejecuta `git commit` ni `git add`, solo devuelve el texto. Quería conservar el control de qué entra al historial y evitar que un mensaje mal inferido se commitee sin revisión.
- **Decisión 2 — Que lea el diff real antes de escribir.** Corre `git diff --staged` en lugar de pedirme que describa el cambio. Reduce mi trabajo y evita mensajes inventados; el diff es la fuente de verdad. Añadí una guarda para staging vacío.
- **Decisión 3 — Integrar la clave de Jira desde el nombre de la rama.** Trabajo a diario con tickets `AXLG-xxx` (workspace `axendev`), así que la skill saca la clave de la rama y la pone como `Refs:`. Es lo que más se olvida a mano y lo que más valor da en Jira.

_Qué me resultó fácil:_

- El frontmatter (`name` + `description`): tenía claro qué debía realizar la skill y en que momento debe dispararse.
- Listar los tipos de Conventional Commits y las reglas del asunto: ya los conozco.

_Qué me resultó ambiguo o difícil de decidir:_

- **Cuánto detalle meter en el `description`.** Dudé entre poner solo el "qué" o también todos los disparadores; terminé cargándolo de frases gatillo para intentar prevenir que falle el disparo de la skill
- **Si debía dejar que ejecutara el commit.** Entre "todo en uno" (cómodo) y "solo el texto" (seguro) elegí seguro, pero no estoy 100% convencido de que sea lo más óptimo en mi día a día.
- **Cuánto recortar sin quedarme corto.** Al querer simplificar dudé si usar los ejemplos o dejar la skill demasiado pelona y más ambigua en sus respuestas. Aposté por lo simple, pero no sé si fue la decisión correcta hasta probarla.

_Tiempo real invertido:_ ~45 min total (lectura previa ~20 min / diseño y escritura ~25 min).

_Qué probarías si tuvieras más tiempo:_

- Un archivo `references/` con las convenciones de mi equipo (scopes permitidos, mapeo feature→scope) para que el `scope` no sea adivinado.
- Añadir manejo de idioma (imitar `git log`) solo si empiezo a trabajar en repos con historial en inglés, para ver si compensa el detalle extra.
- Multiples casos distintos para refinar más el skill

_¿Usaste IA para crear la skill?_

- Sí, usé Claude. _Generé con IA:_ Usé a la IA para generar el borrador inicial del `SKILL.md` en base a un prompt con lo que necesitaba que la skill hiciera. _Decidí y edité yo:_ el propósito (commits + Jira) y que solo proponga sin commitear; además, en una segunda pasada **recorté la skill a mano** con la intención de no generar mucho "ruido"

### Resultado de la prueba (Paso 8)

- **¿Se activó cuando lo esperabas?** Sí el skill se disparo de forma correcta cuando debía hacerlo
- **¿El resultado fue el que querías?** Sí, de forma parcial. Ya que hice la prueba con una rama sin cambios y la skill avisó correctamente que no podia generar un mensaje de commit si no había cambios pendientes.
- **Si no, ¿qué crees que falló?** No falló, pero no tuve oportunidad de probar con una rama con cambios en el diff
