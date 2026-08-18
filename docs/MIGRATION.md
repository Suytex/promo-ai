# Arquitectura y migración — promo-ai

Documentación técnica de `ai.suytex.com`. Landing estática, sin build step, servida
por GitHub Pages desde la raíz de `main`.

Para editar el **contenido** no hace falta leer esto: todo el texto vive en
[`assets/content.txt`](../assets/content.txt) y la página se reconstruye sola.
Este documento cubre el **cómo**: el pipeline de render, las secciones
disponibles, qué se consume del design system y en qué se desvía esta landing
del patrón canónico de `Suytex/promo-portafolios`.

---

## Pipeline de render

`index.html` es un shell: no contiene contenido, solo el layout y el motor que
pinta `content.txt`. El orden importa.

```
fetch('./assets/content.txt')
        │
        ▼
parseFrontmatter(text)      → { meta, body }     separa el bloque --- ... ---
        │
        ▼
domReady()                  espera DOMContentLoaded
        │                   (icons.js inyecta el sprite en ese evento; sin
        │                    esperarlo los <use href="#su-*"> no resuelven
        │                    cuando el fetch responde desde caché)
        ▼
parseSections(body)         → [ { type, content }, ... ]
        │                   split por <!-- section: x -->
        ▼
buildSection(section)       → <section class="section section--{type}">
        │                   marked.parse() del markdown de cada bloque
        ▼
#app.innerHTML = ...        una sola escritura, no por sección
        │
        ▼
enhanceTwoCols('comparison')    reestructuran el DOM ya pintado
enhanceTwoCols('for-who')
enhancePricing(meta)
enhanceCTA(meta)
enhanceHero(meta)
enhanceStickyCTA(meta)
        │
        ▼
enhanceIcons()              SIEMPRE el último
                            rellena los marcadores data-su-icon; corre al
                            final para que sobrevivan a los movimientos de
                            nodos de los enhance* anteriores
```

### Las funciones

| Función | Qué hace |
|---|---|
| `domReady()` | Promesa que resuelve en `DOMContentLoaded` (o ya resuelta si el DOM está listo). |
| `parseFrontmatter(text)` | Extrae el bloque `--- ... ---` inicial a un objeto `meta`. Devuelve `{ meta, body }`. |
| `parseSections(body)` | Parte el body por `<!-- section: x -->` y devuelve `[{ type, content }]`. Descarta lo que haya antes del primer marcador. |
| `buildSection(section)` | Envuelve el markdown parseado en `<section class="section section--{type}">`. Añade `id="reserva"` a la sección `cta`. |
| `asButton(a, meta)` | Helper: apunta un `<a>` a `meta.whatsapp`, le pone `target="_blank" rel="noopener noreferrer"` y las clases `btn btn-primary`. |
| `enhanceTwoCols(type)` | Convierte los dos últimos `h3` de la sección y sus listas en una rejilla de dos columnas. Sirve a `comparison` y `for-who`. |
| `enhancePricing(meta)` | Mete todo lo que sigue al `h2` en una `.pricing-card`; marca el primer `<p>` con `<strong>` como `.price-main`. |
| `enhanceCTA(meta)` | El primer enlace pasa a botón; los demás a `.link-accent`. Marca la primera línea en cursiva como `.cta-closing`. |
| `enhanceHero(meta)` | Reapunta el enlace `#reserva` del hero a WhatsApp y lo convierte en botón. |
| `enhanceStickyCTA(meta)` | Apunta el botón flotante a WhatsApp y lo muestra al salir del hero vía `IntersectionObserver`. |
| `enhanceIcons()` | Rellena cada `[data-su-icon]` con `window.suIcon(nombre)`. Salta los que ya tengan `<svg>`. |

Todas las `enhance*` van envueltas en `try/catch` con `console.warn`: si una
falla, el resto de la página sigue pintando. Un fallo del `fetch` cae al
`.error-state`, que explica que la página necesita un servidor HTTP.

### Frontmatter

```yaml
---
brand: AI Productivity Experience
whatsapp: https://wa.me/18094089999?text=Hola%2C%20quiero%20reservar%20mi%20cupo%20en%20AI%20Productivity%20Experience
---
```

| Campo | Qué hace |
|---|---|
| `brand` | Texto del footer (`© {año} {brand}`). **No** sobrescribe el `<title>` — ver Desviaciones. |
| `whatsapp` | URL de los 4 CTA: hero, pricing, cta y botón flotante. Cambiarlo aquí los cambia todos. |

---

## Secciones

12 secciones, en el orden en que aparecen en `content.txt`. El fondo alterna
solo: `#app .section:nth-of-type(even)` recibe `--su-bg-alt`, así que reordenar
las secciones mantiene el ritmo visual sin tocar CSS.

| # | Marcador | Qué renderiza |
|---|---|---|
| 1 | `hero` | Pantalla completa: `h1` titular, `h2` subtítulo, párrafo y CTA. |
| 2 | `pain` | Rejilla de cards (ícono arriba, etiqueta en negrita, descripción). |
| 3 | `reality` | Texto + callout. |
| 4 | `solution` | Filas con ícono a la izquierda y sangría francesa. |
| 5 | `method` | Rejilla de 5 cards — el temario de los 5 días. |
| 6 | `transformation` | Filas con ícono a la izquierda. |
| 7 | `comparison` | Rejilla de dos columnas + callout debajo. |
| 8 | `format` | Rejilla de cards. |
| 9 | `for-who` | Rejilla de dos columnas (sí / no). |
| 10 | `question` | Texto + callout. |
| 11 | `pricing` | Card de precio con lista, callout y botón. |
| 12 | `cta` | Lista, botón y cierre en cursiva. Lleva `id="reserva"`. |

### Convención tipográfica

Cada sección repite la misma forma: el `h2` es una etiqueta corta
("El Problema") y el `h3` que le sigue carga la declaración. Por eso el CSS
invierte la escala habitual — el `h2` va pequeño y atenuado, el `h3` se lleva
el tamaño de display. Es la "entradilla atenuada" de portafolios, generalizada
a todas las secciones porque aquí todo el copy tiene esa estructura.

El hero es la excepción: su `h1` **es** el titular (en portafolios el `h1` era
la marca y el titular iba en el `h2`). Queda un solo `h1` en la página.

### Dos columnas: por qué los *dos últimos* `h3`

`enhanceTwoCols` toma los dos últimos `h3` de la sección como cabeceras de
columna, no los dos primeros. En esta landing cada sección abre con un `h3` de
titular que debe quedar **por encima** de la rejilla; el `enhanceForWho` de
portafolios asume que el primer `h3` ya es una columna y aquí se comería el
titular. Lo que venga después de la segunda columna (un `blockquote`, otro
encabezado) vuelve a bajar al flujo normal.

---

## Consumo del design system

Tres recursos, todos fijados al tag **`@v1.2.0`**. Nunca `@main`: un tag es
inmutable, `@main` deja entrar cambios sin aviso en una página en producción.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/suytex.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/theme-light.css">
<script src="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/icons.js"></script>
```

| Recurso | Aporta |
|---|---|
| `suytex.css` | Importa las fuentes (Space Grotesk + Inter), los literales de marca y las clases `.su-*`. |
| `theme-light.css` | Capa semántica light: tokens de rol (`bg`/`fg`/`accent`/`line`), radius, `--su-maxw`, `--su-section-y`. **Exige `data-theme="light"` en el `<html>`** — sin ese atributo no aplica nada. |
| `icons.js` | Sprite SVG de 24 íconos Lucide. Define `window.suIcon(nombre, tamaño)`. |

`marked` también va fijado: **`marked@9`**. Antes se cargaba sin versión, lo que
resolvía a `@latest` y dejaba la página expuesta a un breaking change silencioso.

### Tokens usados

El `<style>` inline no declara ni un color. Todo sale de `var(--su-*)`:

```
--su-bg-alt        --su-fg            --su-accent          --su-radius-sm
--su-surface       --su-fg-muted      --su-accent-hover    --su-radius-md
--su-line          --su-fg-subtle     --su-on-accent       --su-radius-lg
--su-line-subtle   --su-font-body     --su-shadow          --su-radius-pill
--su-maxw          --su-font-display  --su-section-y
```

Nota sobre `--su-fg-subtle`: `theme-light.css` documenta en sus propios
comentarios que `#86868B` se queda en 3.6:1 sobre el fondo y no cumple AA como
texto. Aquí se usa solo como **stroke de ícono** (donde el mínimo es 3:1). El
footer y los estados de carga/error usan `--su-fg-muted`.

### Lo que el DS no cubre

El `<style>` inline existe para lo que no está en el sistema: layout,
`.container`, las rejillas de cards y filas, la rejilla de dos columnas, los
callouts, la card de precio, los botones, el botón flotante y los estados de
carga/error. `.su-card` no es alcanzable desde markdown (`marked` genera `<li>`
pelados), así que las cards replican sus valores del tema light con los mismos
tokens.

---

## Iconografía

Cero emoji en el proyecto. Los íconos son marcadores en `content.txt` que
`enhanceIcons()` resuelve contra el sprite del design system:

```markdown
- <span class="su-icon" data-su-icon="search"></span> **Etiqueta** Descripción.
```

La clase `su-icon` es **obligatoria**. Sin ella el `<use>` hereda
`fill:black / stroke:none` y el ícono sale como una mancha negra.

`enhanceIcons()` reemplaza al auto-render de `icons.js`: ese auto-render corre
una sola vez en `DOMContentLoaded` y esta landing pinta su contenido después.

### Íconos en uso

39 marcadores en total: 38 en `content.txt` + 1 en `index.html` (el botón
flotante). 15 de los 24 íconos del set:

| Ícono | Usos | Ícono | Usos |
|---|---|---|---|
| `message-circle` | 7 | `users` | 2 |
| `check-circle` | 6 | `search` | 2 |
| `video` | 4 | `alert-circle` | 2 |
| `clipboard-list` | 4 | `brain` · `award` · `calendar` | 1 c/u |
| `x-circle` | 3 | `clock` · `layers` · `zap` | 1 c/u |
| `target` | 3 | | |

Set completo disponible en `v1.2.0` (24): `alert-circle` `award`
`bar-chart-3` `brain` `calendar` `check-circle` `clipboard-list` `clock`
`dice-5` `help-circle` `layers` `map` `message-circle` `play-circle` `search`
`shield` `smartphone` `target` `trending-up` `users` `video` `wallet`
`x-circle` `zap`.

### La columna negativa

En `comparison` y `for-who` la columna negativa se atenúa con
`--su-fg-subtle`, sin introducir ningún color fuera del sistema. La regla se
ancla al **nombre del ícono**, no a la posición:

```css
[data-theme="light"] #app .two-col .su-icon[data-su-icon="x-circle"] {
  stroke: var(--su-fg-subtle);
}
```

Es a propósito: en `comparison` la columna negativa es la primera y en
`for-who` la segunda. Una regla posicional se rompería en una de las dos.

---

## Historial de migración

### Estado anterior (hasta `55d4a67`)

Estética dark propia, sin relación con el design system:

- Tokens ad-hoc: `--bg: #0a0a0f`, `--accent: #7c3aed`, `--text: #e2e2f0`,
  más `--green` y `--yellow` declarados y nunca usados.
- Glow radial púrpura tras el hero, gradiente de texto en los `h1`, tres
  `box-shadow` tipo glow, `#fff` hardcodeado en 5 sitios.
- Arquitectura primitiva: un único `#content-wrapper` que recibía todo
  `content.txt` parseado de golpe. Sin secciones.
- Hero hardcodeado en `index.html`, duplicando el `#` de la línea 1 de
  `content.txt` (dos `h1` seguidos).
- 2 tablas markdown, 41 emoji, `marked` sin versión, un solo breakpoint (600px).
- Cero consumo de `Suytex/suytex-design`.

### Migración a light-first (`dfa9f60`)

| | Antes | Ahora |
|---|---|---|
| Tema | dark propio | `theme-light.css` @v1.2.0 |
| Secciones | 1 wrapper | 12 declarativas |
| Emoji | 41 (27 glifos) | 0 → 39 íconos Lucide |
| Tablas markdown | 2 | 0 (cards + rejilla) |
| `h1` | 2 | 1 |
| CSS inline | 267 líneas | 421 líneas |
| Colores hardcodeados | 18 | 0 |
| Breakpoints | 1 media query (600px) | 3 media queries (769–1024 · ≤768 · ≤640) |
| `marked` | sin versión | `@9` |

Eliminado: glow radial, gradiente de texto, los 3 `box-shadow` tipo glow, todos
los tokens dark propios, `--green` y `--yellow`, los `#fff` hardcodeados y el
nav sticky.

Añadido: componente de callout para los `blockquote`, rejilla de dos columnas,
tramo intermedio 769–1024px y botón flotante restilizado con tokens.

El CSS creció porque los 267 originales eran mayormente tokens dark y overrides
sobre un único wrapper; los 421 cubren cinco componentes reales (cards, filas,
dos rejillas de columnas, callouts, card de precio) y tres tramos responsive.

Auditado con un barrido de selectores contra el DOM pintado. Los únicos que no
matchean son los dependientes de estado (`.loading`, `.error-state*`,
`#sticky-cta.visible`), el reset de `img` —defensivo, por si el copy gana una
imagen— y `#app .two-col li .su-icon`, que se conserva a propósito: los bullets
de las columnas hoy no llevan ícono, pero los de portafolios sí, y quitarlo
haría que el componente se rompiera en silencio al primer cambio de copy.

### QA verificado

Chromium, 4 viewports (375 · 768 · 1024 · 1440), con Space Grotesk e Inter
realmente cargadas:

- 12/12 secciones renderizan · 39/39 íconos inyectados · cero `<use>` roto
- Cero emoji en el DOM · un solo `h1` · cero gradientes o glows
- Sin desborde horizontal (`scrollWidth == clientWidth`)
- Consola limpia: 0 errores, 0 warnings, 0 fallos de red
- 4/4 CTA a `wa.me/18094089999` con `rel="noopener noreferrer"`
- Contraste AA: 10/10 medidos pasan (mínimo 4.66:1)

---

## Desviaciones del patrón canónico

`Suytex/promo-portafolios` es la referencia estructural. Estas cinco decisiones
se apartan de ella a propósito.

### 1. Botón flotante de WhatsApp conservado

Portafolios no lo tiene. Aquí se mantiene porque el diseño anterior también
tenía un **nav sticky con CTA**, y ese nav sí se eliminó al adoptar la
arquitectura de secciones. Quitar ambos habría dejado la página sin ninguna vía
de conversión permanentemente visible: entre el CTA del hero y el de pricing hay
nueve secciones de scroll. El botón flotante sustituye al nav como vía siempre
disponible, restilizado con tokens del DS y sin el glow del tema dark (usa
`var(--su-shadow)`, la elevación de 1px del sistema).

A ancho completo (≤640px) tapaba el texto del footer 16px al final del scroll;
se resolvió con `padding-bottom` en el footer.

### 2. `document.title` no se sobrescribe con `frontmatter.brand`

Portafolios hace `document.title = meta.brand`. Aquí no: el `<title>` estático
es *"AI Productivity Experience — Tu competencia ya está usando AI"*, que
combina marca y propuesta de valor. Sustituirlo por solo `brand` sería una
regresión de SEO en una página que ya está indexada. `meta.brand` se sigue
usando para el footer.

### 3. Las 2 tablas markdown convertidas a cards y rejilla

Portafolios no tiene ninguna tabla, así que el patrón no cubría este caso y el
design system no tiene componente de tabla.

- **Temario de 5 días** (3 columnas) → 5 cards, reutilizando el componente de
  rejilla de `pain` y `format`. Cero CSS nueva.
- **Comparativa** (2 columnas) → rejilla de dos columnas, reutilizando el mismo
  componente que `for-who`. El contraste "malo vs bueno" es exactamente lo que
  ese layout ya comunica.

Se conserva el texto de todas las celdas. La alternativa —mantenerlas como
tablas estilizadas— exigía construir un componente de tabla desde cero en el
DS, y con dos casos en todo el proyecto no se justificaba frente a reutilizar
componentes ya existentes. Añadido: una tabla de 3 columnas no respira, y la
versión dark ya necesitaba bajar la tipografía a `0.8rem` en móvil para que no
desbordara.

### 4. Callouts: componente nuevo

Portafolios no tiene ni un `blockquote` en su `content.txt` ni CSS para ellos.
Esta landing tiene 4. Tratamiento construido **solo con tokens del sistema**:

```css
#app .section blockquote {
  background: var(--su-surface);
  border: 1px solid var(--su-line-subtle);
  border-left: 3px solid var(--su-accent);
  border-radius: var(--su-radius-md);
}
```

Mismo espíritu que el callout del tema dark anterior (fondo elevado + barra de
acento a la izquierda), sin ningún color fuera del DS. El quinto blockquote del
copy original —el cierre— pasó a líneas en cursiva, que es lo que la sección
`cta` necesitaba.

### 5. Tres gaps de íconos resueltos con `target`, no con `zap`

El set de 24 no cubre seis conceptos del copy: email, pluma/escritura, fuego,
bombilla, herramienta y candado. Cada uno se resolvió con el ícono
conceptualmente más cercano ya disponible, sin inventar símbolos.

Tres de ellos —el fuego de "es un laboratorio, no una conferencia" y las dos
bombillas de los callouts de ROI— habrían caído en `zap`, que ya cubre el rayo
de "respuesta inmediata". Cargar fuego + bombilla + rayo en un mismo glifo lo
vacía de significado, así que se reasignaron a `target`, que estaba libre y
encaja mejor con "laboratorio" y "retorno". `zap` queda exclusivo del rayo.

Los otros tres gaps: email y pluma → `clipboard-list`; candado → `alert-circle`
(urgencia, que es lo que el copy comunica); herramienta → `layers`.

---

## Desarrollo local

`fetch()` no funciona sobre `file://`. Hace falta un servidor HTTP:

```bash
python3 -m http.server 3000
# o
npx serve .
```

Luego `http://localhost:3000`.

---

## Estructura del repo

```
/
├── index.html          # Shell: layout + CSS + motor de render
├── assets/
│   └── content.txt     # TODO el contenido editable
├── docs/
│   └── MIGRATION.md    # Este archivo
├── CNAME               # ai.suytex.com
├── .nojekyll           # Evita el procesado Jekyll de GitHub Pages
└── README.md
```

`CNAME` y `.nojekyll` son necesarios para el dominio y el servido correcto en
GitHub Pages. No borrar.
