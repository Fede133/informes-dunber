# CLAUDE.md

Guía para asistentes de IA que trabajen en este repositorio.

## Qué es este repositorio

Es un **host estático de informes publicados** para Dunber S.A. (distribuidora de
bebidas). No es una aplicación con código fuente, build o dependencias: es el
*destino de publicación* de informes HTML autocontenidos y **cifrados**, que
genera un sistema externo (backoffice/app de gestión que no vive acá).

Cada informe es un único archivo `.html` que:

- trae todo embebido (CSS, JS, datos) — cero requests externos, cero librerías;
- guarda los datos **cifrados** en una constante `CIFRADO` y pide un código de
  acceso de la empresa para desbloquearse en el navegador;
- se comparte por link directo (Chat Empresas / WhatsApp), no por navegación
  desde el índice.

El repositorio se sirve como sitio estático (GitHub Pages). No hay workflows de
CI, ni linters, ni tests, ni `package.json`.

## Estructura

```
.
├── index.html          Landing mínima: explica que el acceso es por link directo
├── v.html              Visor/redirector con espera-y-reintento (?i=<ruta-al-informe>)
├── README.md
└── 2026/
    ├── 07/             211 informes publicados en julio 2026
    └── 08/              16 informes publicados en agosto 2026
```

- **`YYYY/MM/` es el mes de *publicación*, no el período del informe.** Ej.:
  `2026/08/…informe-gestion-mes-2026-07-01-2026-07-31.html` cubre julio pero se
  publicó en agosto.
- ~227 archivos, ~28 MB. Cada informe pesa entre 22 KB y 810 KB (promedio
  ~130 KB) porque el payload cifrado va inline en base64.
- No hay assets compartidos: si dos informes se ven iguales es porque el
  generador repitió el mismo CSS/JS en cada archivo.

### Convención de nombres

```
{epochMs}_{tipo}[-{slug}]-{período}.html
```

El prefijo es el timestamp de generación en milisegundos y hace único cada
archivo. Tipos que existen hoy:

| Patrón | Ejemplo | Cant. |
|---|---|---|
| `{ts}_planificacion-preventa-ruta-{ruta}-{YYYY-MM-DD}.html` | `1785732147850_planificacion-preventa-ruta-901-2026-08-03.html` | 130 |
| `{ts}_visita-supervisor-ruta-{ruta}-{YYYY-MM-DD}.html` | `1785732258245_visita-supervisor-ruta-801-2026-08-03.html` | 64 |
| `{ts}_informe-gestion-semana-{desde}-{hasta}.html` | `1785424550503_informe-gestion-semana-2026-07-27-2026-07-30.html` | 5 |
| `{ts}_informe-gestion-mes-{desde}-{hasta}.html` | `1785732330631_informe-gestion-mes-2026-07-01-2026-07-31.html` | 1 |
| `{ts}_comparativo-{marca}-{per_a}_{per_a}-vs-{per_b}_{per_b}.html` | `1784086396354_comparativo-ades-2026-07_2026-07-vs-2025-07_2025-07.html` | 25 |
| `{ts}_informe-{marca}-{desde}_{hasta}.html` | `1784078663375_informe-monster-energy-2026-04_2026-06.html` | 2 |

- Rutas vistas: `10`, `15`, `99`, `101`–`103`, `201`–`204`, `301`–`303`,
  `401`–`403`, `501`, `601`–`603`, `801`–`803`, `901`–`904`.
- Marcas (slug en minúscula, sin tildes): `coca-cola`, `sprite`, `fanta`,
  `aquarius`, `powerade`, `monster-energy`, `ades`, `cepita`, `cepita-fresh`,
  `benedictino`, `las-tres-ninas`, `vinas-riojanas`.
- **El repositorio es append-only.** Regenerar un informe crea un archivo nuevo
  con timestamp nuevo; el anterior queda. Por eso hay 5 archivos distintos para
  `informe-gestion-semana-2026-07-27-2026-07-30` y 3 para
  `planificacion-preventa-ruta-903-2026-07-16`. **Nunca deduplicar ni borrar
  informes viejos** sin pedido explícito: los links compartidos ya circulan y
  romperlos deja al destinatario sin el informe.

## Anatomía de un informe

Los 227 archivos comparten exactamente el mismo esqueleto (verificado: el bloque
criptográfico es byte-idéntico en todos). De arriba hacia abajo:

1. `<style>` con design tokens en `:root` + variante
   `@media (prefers-color-scheme: dark)` — paleta verde (`--accent:#4f8a10`),
   `--down` para caídas, `--warn` para alertas. Tipografía `"Segoe UI", system-ui`.
2. `#nojsAviso`: banner de fallback que se auto-elimina con un `<script>`
   inmediato. Si el visor no ejecuta JS (preview de iPhone, algunos clientes de
   correo), el banner queda visible explicando que hay que abrir el link en el
   navegador. **No tocar este patrón.**
3. `#lock` / `#lockForm`: pantalla de código de acceso. `<body class="locked">`
   oculta el contenido hasta desbloquear.
4. Estructura vacía del informe (`#kpis`, tablas, secciones) — se llena por JS.
5. `<script>` con: `const CIFRADO = {…}`, la función `iniciarInforme()` con todo
   el render, la implementación de SHA-256 a mano, y el handler del form.

### Esquema de cifrado

Implementado a mano, sin WebCrypto (para funcionar también sobre `file://` y en
WebViews viejos):

```js
CIFRADO = { s: salt_b64, n: nonce_b64, v: verificador_b64, c: ciphertext_b64 }

derivarClave(codigo, salt) = sha256^30000( sha256(salt || utf8(codigo)) )
verificación:  sha256(clave || "verificacion") === v   // comparación en tiempo constante
descifrado:    keystream por bloque = sha256(clave || nonce || contador_be32)
               plaintext = ct XOR keystream   → JSON.parse
```

- `salt`, `nonce` y `verificador` son **únicos por archivo** (no se repite
  ningún salt en el repositorio).
- 30.000 iteraciones de SHA-256 encadenado como KDF; el desbloqueo tarda un
  instante y por eso el `setTimeout(…, 30)` antes de descifrar (deja pintar el
  "Verificando…").
- El código se normaliza con `trim().toUpperCase()` y, si es correcto, se
  persiste en `localStorage['dunber_informes_codigo']`, con reintento silencioso
  al abrir el siguiente informe.
- Al desbloquear, `abrir(payload)` inyecta `payload.titulo`, `payload.eyebrow`,
  `payload.h1`, `payload.foot`, asigna `DATA = payload.data`, saca
  `body.locked`, elimina `#lock`, llama a `iniciarInforme()` y agrega
  `#shareBox` (botones "Copiar resumen" / "Compartir link", el segundo sólo
  bajo `http(s):`).

**No intentes descifrar, adivinar ni fuerza-brutear el código de acceso, ni
extraer los datos del payload.** Si necesitás entender un informe, leé el
código de render (`iniciarInforme`), que es plaintext y describe la forma exacta
de los datos.

### Forma de `DATA` por tipo de informe

Deducible del render sin descifrar nada:

| Tipo | Campos de `DATA` |
|---|---|
| `planificacion-preventa` | `id`, `fecha`, `ruta`, `objetivo`, `checklist`, `clientesRuta`, `clientesDetalle`, `coberturasResumen`, `config.{ventanaProducto, ventanaNoComprador}` |
| `visita-supervisor` | `id`, `fecha`, `ruta`, `vendedor`, `clientes`, `config` |
| `informe-gestion-semana` / `-mes` | `desdeLegible`, `hastaLegible`, `ruta`, `rutas[]`, `dias[]`, `secciones[]`, `kpis`, `tareas` |
| `comparativo-{marca}` | `a`, `b` (un período por lado) |
| `informe-{marca}` | `resumen`, `meses`, `dias`, `articulos`, `vendedores`, `clientes` |

### Vocabulario del dominio

`UC` (unidades de caja / units) · `bultos` · `ticket promedio` · `efectividad de
visita` (clientes con compra / clientes a visitar) · `cobertura` · `Reglas de
Oro` · `Sell Out` · `SOVI` (Share of Visible Inventory) · `preventa` /
`preventista` · `ruta` · `cierre` (cierre de la planificación del día) ·
`atípico` (cierre con volumen > 5× el objetivo, se excluye del %).

## Flujo de trabajo y convenciones

- **Los informes son artefactos generados. No editarlos a mano**, no
  reformatearlos, no "prettificarlos", no refactorizar el CSS/JS duplicado.
  Un cambio de estilo o de lógica de render pertenece al generador (fuera de
  este repositorio), no a los 227 archivos ya publicados.
- Un commit por archivo publicado, con mensaje exactamente
  `Publicar {ruta/del/archivo.html}`. Los 50 commits del historial siguen esa
  forma sin excepción. Respetala si publicás algo.
- Formato de los informes: HTML sin indentación uniforme, comillas dobles en
  atributos, `<!doctype html>` en minúscula, `lang="es"`. Todo el texto de cara
  al usuario, los comentarios y los identificadores del código están **en
  español** (`descifrar`, `abrir`, `intentar`, `barra`, `renderCierres`). Mantené
  ese idioma en cualquier cosa que agregues.
- El clon de esta sesión es **shallow (depth 50)**: el "primer" commit visible
  trae 181 archivos de golpe, no es el inicio real del proyecto. Si necesitás
  historia más profunda, `git fetch --unshallow`.
- Sin build ni tests: para verificar un cambio en `index.html` o `v.html`,
  abrilos en un navegador (o `python3 -m http.server` desde la raíz) — un
  informe cifrado no se puede verificar sin el código de acceso.
- No hay `.nojekyll`; los nombres de archivo empiezan con dígitos y ningún
  directorio empieza con `_`, así que Jekyll los sirve tal cual.

## `v.html` — visor con espera

`v.html?i=2026/08/1785732330631_informe-….html` muestra un spinner y hace
`HEAD` sobre el destino cada 5 s (hasta 40 intentos ≈ 3,3 min), redirigiendo con
`location.replace` en cuanto el archivo existe. Sirve para links que se envían
*antes* de que el commit termine de propagarse a Pages.

El parámetro `i` se valida contra
`/^[0-9]{4}\/[0-9]{2}\/[A-Za-z0-9_.-]+\.html$/` antes de armar el destino: sólo
acepta `YYYY/MM/archivo.html`, con lo cual no se puede usar `v.html` para
redirigir a un dominio externo ni para salir del directorio.

**Bug ya corregido (no reintroducir):** hasta agosto 2026 ese literal de regex
tenía las barras internas sin escapar (`/^[0-9]{4}/[0-9]{2}/…/`), lo que es un
`SyntaxError` de parseo: el IIFE entero nunca corría y la página quedaba con el
spinner girando para siempre, sin redirigir ni mostrar el error. Si tocás esta
línea, acordate de que las `/` y el `.` de `.html` van escapados.

## Seguridad y privacidad

- Los informes contienen datos comerciales reales (clientes, ventas, montos,
  evaluación de vendedores) protegidos únicamente por el código de acceso de la
  empresa. El repositorio es la única barrera de distribución.
- **Nunca commitear datos en claro**: ni JSON de origen, ni un informe
  descifrado, ni el código de acceso, ni fixtures con nombres de clientes
  reales.
- No publiques informes en artifacts, gists, ni ningún servicio externo.
- Los payloads cifrados ya versionados no se pueden "desalojar" del historial
  sin reescribirlo; tratá cada publicación como definitiva.
