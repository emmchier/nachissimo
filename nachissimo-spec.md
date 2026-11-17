# Nachissimo — Portfolio de Nacho · Especificaciones del proyecto

> **Archivo de contexto maestro.** Claude debe leer este archivo al inicio de cada sesión de
> trabajo sobre este proyecto, antes de escribir código o investigar documentación.
> Última actualización: 2026-11-17 · Versión: **0.1** (arranque)

---

# ▶ RETOMAR ACÁ — Estado al 17/11/2026

**Cómo se retoma:** sesión nueva, **adjuntar este archivo** y decir "sigamos con el portfolio de
Nacho". Una sesión nueva arranca con el workspace vacío: sin este archivo no hay contexto.

### Dónde estamos
Proyecto recién abierto. **Todavía no existe el sitio en Wix.** Arquitectura y stack definidos
(§2–§3), modelo de datos propuesto (§4), pipeline de imágenes definido (§13).
No se escribió código.

### Tareas pendientes
- [ ] **T1.** Definir en qué cuenta de Wix vive el sitio y quién paga el plan (§12.6).
- [ ] **T2.** Crear el sitio en Studio Two.
- [ ] **T3.** Crear la colección `proyectos` con el schema de §4.1 y la `configuracion` de §4.2.
- [ ] **T4.** Crear la item page dinámica de `proyectos`.
- [ ] **T5.** Decidir dónde se hostean los videos (§13.5). **Bloqueante para un animador.**
- [ ] **T6.** Pedirle a Nacho: 6–10 proyectos de muestra con sus imágenes, bio, links y CV.
- [ ] **T7.** Conectar `nachissimo.com` (requiere plan pago).

### Plan de la primera sesión de trabajo (en orden)
1. Crear el sitio en Studio Two y registrar su `siteId` en §0.
2. Crear colecciones y la item page dinámica.
3. Entregar **`ProjectHero`** + **`ProjectGallery`** como archivos del proyecto, con ruta destino
   y paso a paso de scaffold, pegado, build y binding.
4. Validar el binding de punta a punta con un proyecto de prueba. Ese paso prueba la arquitectura.
5. Seguir con `WorkGrid` + `FilterTabs` (listado filtrable).
6. **Fase 1.5:** el uploader con compresión (§13) antes de que Nacho cargue obra real.
7. Actualizar §11 y este bloque al cerrar la sesión.

### Criterio de "listo" de la fase 1
La ficha de un proyecto renderiza con ERCs propios leyendo el ítem real del CMS, en desktop y
mobile, con imágenes y video embebido.

---

## 0. Estado del sitio

| | |
|---|---|
| Sitio en Wix | **no creado todavía** |
| `siteId` | — (anotar acá en cuanto exista) |
| Editor | Studio Two |
| **Dominio** | **`nachissimo.com` — ya comprado y pago por Nacho**, registrado fuera de Wix |
| Plan | a definir (§12) |
| Idioma | a definir: es-AR, o es + en para clientes del exterior (§10) |

---

## 1. Contexto

- **Cliente:** Nacho — **ilustrador y animador**.
- **Producto:** portfolio público sencillo + dashboard administrativo propio.
- **Objetivo:** mostrar su obra con impacto visual. Es un portfolio: **la imagen y el video son el
  producto**, no un adorno. La fidelidad visual pesa más que en cualquier otro tipo de sitio.
- **Plataforma:** **Wix Studio Two** (no Wix Studio, no Headless). Fundamento en §3.
- **Premisa de diseño:** usar **Editor React Components (ERC)** como unidad de construcción para
  tener dominio total del diseño sin las limitaciones de los componentes nativos.
- **Premisa de trabajo:** economía de tokens. El usuario ejecuta a mano todo lo que sea
  "clickear en el editor"; Claude escribe código, arquitectura y revisiones. Ver §9.

### Alcance en dos fases
1. **Portfolio público (read-only):** Home, Proyectos (listado filtrable), detalle dinámico de
   cada proyecto, Sobre mí, Contacto.
2. **Dashboard administrativo** para Nacho: crear, editar, eliminar, publicar/despublicar,
   destacar y **reordenar** proyectos; subir imágenes con el pipeline de compresión; y editar sus
   datos de contacto, bio y links, que se reflejan en el sitio público.

---

## 2. Stack y restricciones de Studio Two (verificado en docs, nov-2026)

### Lo que usamos
| Capa | Herramienta |
|---|---|
| UI del sitio | **ERC** (React + TypeScript, manifest-driven) |
| Estado compartido en página | **Data Sources** (context providers propios) |
| Utilidades compartidas | **Function libraries** |
| Datos | **Wix CMS** + **Data API** (`@wix/data`, preinstalado) |
| Binding sin código | **CMS Data Sources** (uno por colección + uno `(item page)`) |
| Backend | **HTTP endpoints** (solo los llama nuestro frontend), **backend events**, **service plugins** |
| Dashboard | **Dashboard pages / modals / plugins** (React) + **Dashboard SDK** + **WDS** |
| Entorno | Tab **Code** = VS Code en el navegador, npm, terminal, Aria, Wix Extension Builder |
| Contexto en runtime | Site SDKs: `@wix/site-service-router`, `-url`, `-pages`, `-lightbox`, `-locale`, `-site-theme`, `-device-info`, `-rendering-context`, `-head-appender` |

### Lo que NO existe en este release (restricciones duras)
- **Custom Elements**, `$w`, page code, Velo, datasets, data hooks, forms con código.
- **Custom CSS global** (todavía no soportado).
- **Web modules**, **HTTP functions públicas**, **routers propios**, **scheduled jobs**,
  **triggered emails**, custom actions de automations.
- **Sin integración Git**, sin IDE local, sin Wix IDE, sin Velo packages, sin testing funcional.
- No se pueden reemplazar las páginas por defecto de apps de negocio.
- Backend con límites de cómputo fijos, iguales para todos los sitios, sin autoescalado.
- **Export to code (beta): un sitio con páginas dinámicas NO se puede exportar.**
  → El repo de GitHub se mantiene **a mano** (fuente de verdad local, copy/paste al tab Code).

### Límites del modelo de datos que condicionan el diseño
- Un CMS Data Source **solo muestra** datos: no agrega, no actualiza, no borra, no hace queries
  custom ni agregaciones.
- Un **ERC no puede leer ni controlar un CMS Data Source desde código**: solo recibe lo que se le
  bindea como props en el editor. Para datos en código → Data API o Data Source propio.
- La Data API respeta los permisos de cada colección. `auth.elevate()` **solo en backend**.
- Data events: solo colecciones creadas por el usuario.
- Consistencia eventual: `{ consistentRead: true }` cuando se lee justo después de escribir.
- Para enviar un mail al llegar una consulta: **automation con trigger custom llamada desde un
  HTTP endpoint** (los triggered emails no existen acá).

---

## 3. Decisión de arquitectura: Studio Two puro

**Todo en Studio Two. No Headless, no híbrido.**
- Headless daría control total del frontend, pero habría que construir y hostear el frontend,
  perder el editor y el binding visual, y crear una app privada para tener dashboard. Costo alto,
  beneficio nulo para un portfolio.
- Export to code no es vía de escape: no soporta páginas dinámicas, y este sitio las necesita.
- El límite real de Studio Two (sin routing propio ni CSS global) no afecta a un portfolio: las
  URLs las resuelve la página dinámica del CMS.

### Granularidad de los ERC
**ERCs componibles**, no un monolito por página: el binding al CMS es *por prop*, los componentes
se reusan entre Home, listado y detalle, y Nacho puede reordenar secciones sin tocar código.
Excepción: un ERC "de página" compuesto internamente por subcomponentes React propios, que recibe
`_id`/`_slug` por props y resuelve el resto con la Data API.

### Patrón para el listado filtrable
Un repeater con CMS Data Source lista y pagina, pero no filtra por categoría de forma combinada
ni ordena por un campo manual. Por eso:
```
[Data Source propio: WorkFeedSource]  ← query con Data API + estado del filtro
   ├── [ERC FilterTabs]   (categorías; bindeado a las funciones del Data Source)
   ├── [ERC WorkGrid]     (bindeado a items / isLoading / loadMore)
   └── [ERC ProjectCard]  (dentro del template container del grid)
```

### Patrón para la ficha de proyecto (item page dinámica)
- La item page la crea Wix (`Add Page > Collection page > Item page`); adjunta el Data Source
  `Proyectos (item page)`, que siempre contiene el ítem actual.
- Props simples (título, cliente, año, descripción, portada) → **binding directo**, cero código.
- Props con lógica (galería con lightbox, video embebido, "siguiente proyecto") → ERC que recibe
  `_id`/`_slug` por binding y consulta con la Data API.

---

## 4. Modelo de datos (CMS)

### 4.1 Colección `proyectos`
| Campo (key) | Tipo | Notas |
|---|---|---|
| `titulo` | TEXT | |
| `_slug` | SLUG | lo genera Wix al crear la item page |
| `cliente` | TEXT | vacío si es personal |
| `anio` | NUMBER | |
| `categoria` | TEXT | valores controlados: `ilustracion` · `animacion` · `editorial` · `branding` · `personal` |
| `formato` | TEXT | `imagen` · `video` · `mixto` — gobierna cómo rendea la ficha |
| `portada` | MEDIA_IMAGE | la que se ve en el grid; **ratio fijo** para que el grid no baile |
| `galeria` | MEDIA_GALLERY | piezas del proyecto |
| `videoUrl` | TEXT (URL) | Vimeo/YouTube — **nunca** video subido a Wix (§13.5) |
| `videoPosterUrl` | TEXT (URL) | frame de portada del video |
| `descripcion` | RICH_TEXT | el relato del proyecto |
| `rol` | TEXT | "dirección de arte, animación 2D" |
| `herramientas` | ARRAY_STRING | tags filtrables: After Effects, Procreate, Blender… |
| `creditos` | RICH_TEXT | equipo, estudio, agencia |
| `tags` | ARRAY_STRING | |
| `destacado` | BOOLEAN | aparece en Home |
| `publicado` | BOOLEAN | **gobierna la visibilidad**: todo query público filtra `publicado = true` |
| `orden` | NUMBER | **orden manual del portfolio** — clave: un ilustrador cura su secuencia, no la ordena por fecha |
| `fecha` | DATE | para ordenar por reciente cuando haga falta |
| `seoTitle`, `seoDescription` | TEXT | |

Índices: regular en `publicado`, `categoria`, `orden`.
Permisos: `read: ANYONE`, `insert/update/remove: ADMIN`.

### 4.2 Colección `configuracion` (singleton, un solo ítem)
Para que Nacho edite sus datos desde el dashboard sin tocar el editor.
Campos: `nombre`, `claim` (una línea), `bioCorta` (TEXT), `bioLarga` (RICH_TEXT),
`retrato` (MEDIA_IMAGE), `email`, `instagram`, `behance`, `vimeo`, `linkedin`, `cv` (DOCUMENT),
`disponibilidad` (TEXT: "disponible para proyectos" / "agenda cerrada"), `textoLegal` (RICH_TEXT).
Permisos: `read: ANYONE`, escritura `ADMIN`. Se consume con un **Data Source propio global**
(`SiteConfigSource`) attachado al header/footer, o por Data API dentro de los ERC.

### 4.3 Consultas de contacto
Dos caminos:
- **A (MVP, cero código):** app **Wix Forms**. Guarda las submissions y manda notificación sin
  escribir nada. Ventaja extra: **sus ítems no consumen el cupo de CMS** (§12.3).
- **B (diseño propio):** ERC `ContactForm` → **HTTP endpoint** que valida, inserta con
  `auth.elevate()` en una colección `consultas` cerrada al público, y dispara una **automation con
  trigger custom** para el mail.
**Arrancar con A.** Para un portfolio, un mailto o un form simple suele alcanzar.

---

## 5. Mapa de páginas

| Página | Ruta | Contenido |
|---|---|---|
| Home | `/` | hero con claim + grid de destacados + bio corta + CTA contacto |
| Proyectos | `/proyectos` | grid completo con filtros por categoría |
| Detalle de proyecto | `/proyectos/{slug}` | **item page dinámica** sobre `proyectos` |
| Sobre mí | `/sobre-mi` | bio larga, retrato, herramientas, clientes, CV |
| Contacto | `/contacto` | form + links |
| 404 | — | con links al portfolio |
| Privacidad | `/privacidad` | solo si se cargan analytics o formularios |

Rutas cortas y limpias desde el principio: renombrar antes de publicar es gratis, después
requiere redirects 301.

---

## 6. Inventario de ERC a construir

| ERC | Props clave | Dónde |
|---|---|---|
| `ProjectCard` | titulo, cliente, anio, categoria, portada, itemPageLink | Home, listado |
| `WorkGrid` | items[], isLoading, loadMore, renderCard (template container) | Home, listado |
| `FilterTabs` | categorías + funciones del Data Source | listado |
| `ProjectHero` | titulo, cliente, anio, rol, portada | detalle |
| `ProjectGallery` | galeria[] — con lightbox y navegación por teclado | detalle |
| `VideoEmbed` | videoUrl, videoPosterUrl — lazy, carga el player al click | detalle, Home |
| `ProjectCredits` | creditos, herramientas[] | detalle |
| `NextProject` | `_id`, `orden` (query por Data API) | detalle |
| `AboutHero` | de `configuracion` | Sobre mí |
| `ContactBlock` | email, redes (de `configuracion`) | global |
| `SiteConfigSource` | Data Source global de `configuracion` | header/footer |

Convenciones: el `type` del manifest arranca con `companionApp.`; los selectores deben ser clases
string planas (CSS Modules las reescribe en build); props de contenedor con `DATA` /
`CONTAINER_TYPE` de `@wix/react-component-schema`.

Notas de craft para este proyecto:
- El grid es la pieza principal: definir si es **masonry** o de ratio fijo **antes** de escribir
  `WorkGrid`. Masonry se ve mejor con obra de formatos mixtos, pero complica el layout responsive.
- `VideoEmbed` debe cargar el iframe **solo al click** (poster primero). Si no, una Home con
  varios videos se vuelve lentísima.

---

## 7. Dashboard administrativo

### 7.1 Login y seguridad — **ya resuelto por Wix, no se programa**
- Acceso con **cuenta de Wix** + **Roles & Permissions** del sitio. Sin auth custom, sin Members
  Area, sin sesiones.
- Nacho se invita como **colaborador** desde *Roles & Permissions* (invitación por email,
  válida 30 días).
- Rol recomendado: **Content Collection Manager** — agrega y modifica contenido de todas las
  colecciones y **no** puede editar otras áreas del sitio. Si hace falta más granularidad,
  **rol custom**.
- Las dashboard pages **nunca aparecen en el sitio live** y solo las ven usuarios de Wix con los
  permisos requeridos. El código corre con la **identidad del usuario logueado**, acotado por su rol.
- Activar **verificación en dos pasos** en su cuenta de Wix.
- ⚠️ **Asientos de colaborador:** 1 gratis por defecto; 5 en sitios de una cuenta de Wix Studio.
- Con ese rol también puede administrar las colecciones desde la **app móvil de Wix**.

### 7.2 Qué trae resuelto sin escribir código
- El **CMS dashboard nativo** ya hace alta, edición, borrado y vista de ítems.
- Publicar / destacar / reordenar = editar los campos `publicado`, `destacado` y `orden` ahí mismo.
- → El dashboard custom **no bloquea el lanzamiento**, con una excepción: el uploader de imágenes
  (§13.4), que sí es obligatorio.

### 7.3 Dashboard custom
- **Dashboard page** React agregada con Wix Extension Builder.
- UI con **Wix Design System (WDS)** + Patterns.
- **Dashboard SDK**: `navigate()`, modales y **`openMediaManager()`** (developer preview).
- CRUD con la **Data API**; operaciones sensibles o masivas vía HTTP endpoint con `elevate`.
- Pantallas:
  1. **Grilla de proyectos** con drag-to-reorder (escribe `orden`) y toggles de
     `publicado` / `destacado`. Para un portfolio, reordenar es la acción más frecuente.
  2. **Alta/edición de proyecto** con el uploader del pipeline (§13).
  3. **Mis datos** (`configuracion`).
  4. **Auditoría de archivos pesados** (§13.4).
- Sin scheduled jobs → nada automático por fecha.

---

## 8. Riesgos y cosas a vigilar
1. Release inicial de Studio Two: faltan piezas (CSS global, Git, HTTP functions públicas).
   Los paneles custom de ERC están marcados **alpha** en docs.
2. Sin Git: el repo es manual; la fuente de verdad del código vive local/GitHub.
3. Export to code no sirve acá (páginas dinámicas). No planificar round-trip.
4. **Performance**: es el riesgo número uno de un portfolio. Imágenes grandes y videos embebidos
   matan el LCP. Mitigación: pipeline de §13, servir medidas por breakpoint con la SDK Media API,
   y `VideoEmbed` con carga diferida.
5. **Fidelidad visual**: comprimir de más arruina la obra de un ilustrador. Los parámetros de §13
   son más conservadores que para un sitio común, a propósito.
6. SEO: los ERC rendean también en servidor, pero la semántica (h1/h2, `alt` descriptivo en cada
   pieza, datos estructurados) es responsabilidad nuestra. En un portfolio el `alt` también es
   accesibilidad real.
7. Accesibilidad: al no usar componentes nativos, teclado/ARIA quedan a cargo nuestro. El lightbox
   necesita foco atrapado, Escape y navegación por flechas. Usar `refElement` sobre elementos
   nativos de Wix donde convenga.
8. **Multi-idioma a verificar**: si Nacho apunta a clientes del exterior, hace falta confirmar el
   soporte de Wix Multilingual en Studio Two antes de prometerlo. No está verificado.

---

## 9. Protocolo de trabajo (economía de tokens)

**División de tareas**
- **El usuario, a mano en Studio Two:** crear/modificar colecciones y campos, crear páginas,
  agregar la dynamic page, colocar los ERC en el canvas, hacer los bindings, publicar, invitar
  colaborador.
- **Claude, en sesión:** arquitectura, schema, código de ERCs / Data Sources / HTTP endpoints /
  dashboard pages (como archivos de este proyecto), revisiones y debugging.

**Reglas**
1. Claude lee `specs.md` al inicio y **no re-investiga** lo que ya está documentado acá.
2. El código se entrega como archivos del proyecto; el usuario los copia al tab Code. Siempre con
   la ruta destino explícita.
3. **Inspección del sitio sin canvas:** una vez que exista el sitio, Claude puede leer colecciones,
   campos, páginas estáticas, permisos y apps por el MCP de Wix con el `siteId` de §0. Usar esa vía
   antes de pedir capturas.
   - Colecciones + campos: `GET https://www.wixapis.com/wix-data/v2/collections`
   - Páginas estáticas: `POST https://www.wixapis.com/ricos/v1/site-context/static-pages/list`
   - Contexto del sitio: `GetSiteContext`
4. Revisión visual: preferir capturas del canvas antes de abrir el navegador.
5. Aria se usa para tareas mecánicas dentro del editor (scaffolding, regenerar manifests), no para
   decisiones de arquitectura.
6. Toda decisión nueva se registra en §11 en la misma sesión en que se toma.
7. **Al cerrar cada sesión**, Claude actualiza el bloque **RETOMAR ACÁ** y reenvía el archivo.
8. El código entregado se guarda en `src/` con la misma estructura de carpetas que el tab Code.

---

## 10. Decisiones pendientes
- [ ] ¿El sitio vive en la cuenta de Wix del usuario o en una de Nacho?
- [ ] ¿Quién paga el plan? (§12)
- [ ] Plan: Basic anual o mensual (§12.6).
- [ ] **Hosting de video**: Vimeo (sin publicidad, mejor para portfolio, tiene costo) vs YouTube
      (gratis, pero marca y sugeridos). **Decisión necesaria antes de `VideoEmbed`.**
- [ ] Grid: masonry o ratio fijo.
- [ ] Idioma: solo es-AR, o es + en.
- [ ] ¿Cuántos proyectos tiene para cargar al inicio? Define si hace falta carga masiva.
- [ ] ¿Quiere mostrar precios/servicios o solo obra?

---

## 11. Changelog de decisiones
| Fecha | Decisión |
|---|---|
| 2026-11-17 | Plataforma: Studio Two puro. Descartados Headless e híbrido. |
| 2026-11-17 | ERCs componibles en lugar de un monolito por página. |
| 2026-11-17 | Listado con Data Source propio + Data API (los CMS Data Sources no filtran ni ordenan lo suficiente). |
| 2026-11-17 | Login del dashboard: Roles & Permissions de Wix + rol Content Collection Manager. Sin auth custom. |
| 2026-11-17 | Orden manual de proyectos con campo `orden` + drag-to-reorder en el dashboard. |
| 2026-11-17 | Consultas: arrancar con Wix Forms (camino A). |
| 2026-11-17 | Colección `configuracion` singleton para los datos editables por Nacho. |
| 2026-11-17 | **Video siempre externo (Vimeo/YouTube), nunca subido a Wix.** |
| 2026-11-17 | Dominio `nachissimo.com` ya es de Nacho: se conecta, no se compra. |

---

## 12. Costos — qué cuesta tener esto online

### 12.1 Qué se paga y qué no
| Concepto | Costo |
|---|---|
| Studio Two (el editor) | **$0** |
| Los ERC, el código, el backend, el dashboard custom | **$0** — la plataforma de desarrollo viene incluida desde el plan más barato |
| Acceso de Nacho al dashboard | **$0** — asiento de colaborador incluido |
| Plan del sitio (para publicarlo con dominio propio y sin marca Wix) | **único costo recurrente** |
| Dominio | **ya pago por Nacho**, se renueva donde está registrado |

En Studio Two **los límites de cómputo del backend son fijos e iguales para todos los sitios**,
sin importar el plan. **No hay que subir de plan por tener código, ERCs ni dashboard custom.**

### 12.2 Planes de Wix Studio (lista, USD/mes con facturación anual)
| Plan | Precio | Storage | Colaboradores | Ítems de CMS |
|---|---|---|---|---|
| **Basic** | **$19** | 10 GB | 3 | 1.500 |
| Standard | $27 | 50 GB | 5 | 4.000 |
| Plus | $34 | 120 GB | 10 | 20.000 |
| Elite | $159 | ilimitado | 100 | 10.000.000 |

Con facturación **mensual** el precio sube (Basic ≈ $27/mes). Garantía de devolución de 14 días.
**Cada plan vale para un solo sitio**: este portfolio necesita su propia suscripción.

### 12.3 Recomendación: **Basic**
- 10 GB con el pipeline de §13 y el video afuera: de sobra para un portfolio.
- Los 1.500 ítems de CMS no se tocan ni de casualidad con proyectos de portfolio. Además el límite
  cuenta solo colecciones propias; **los ítems de colecciones de apps de Wix no cuentan**, así que
  las consultas de Wix Forms no consumen cupo. ⚠️ Si se migra a una colección `consultas` propia
  (camino B de §4.3), esas **sí** cuentan.
- Basic no acepta pagos online: irrelevante, el sitio no vende.

### 12.4 El dominio ya está comprado: qué implica
- **No abarata el plan.** Conectar un dominio propio **igual requiere plan pago**.
- El voucher de dominio gratis del primer año (que viene con los planes anuales) **no aplica** acá
  y **no se canjea por descuento**. Simplemente no se usa.
- Lo que sí cambia: **el plan mensual deja de tener su segunda desventaja.** Para este proyecto,
  mensual solo cuesta más caro, no hace perder un dominio gratis.
- `nachissimo.com` se **apunta** a Wix por DNS o nameservers. Sin costo extra, solo configuración.
  Nacho sigue pagando la renovación donde lo tenga registrado.

### 12.5 Lo que hay que sumarle en Argentina
El precio es en USD, así que al pagar con tarjeta argentina se suma:
- **IVA servicios digitales 21%** (RG 4240) — se paga siempre, incluso pagando con dólares.
- **Percepción 30% a cuenta de Ganancias** (RG 5617) — **se evita pagando con dólares propios**,
  porque no hay cambio de divisas. El Impuesto PAIS ya no se aplica (terminó en dic-2024).
- **Percepción provincial de IIBB**, según la provincia.

### 12.6 Números en ARS (al 17/11/2026, oficial venta BNA $1.575)
| Forma de pago | Basic |
|---|---|
| **Anual pagando con dólares propios** | ≈ **$435.000** de una vez |
| Anual pagando en pesos (dólar tarjeta) | ≈ $542.000 |
| **Mensual pagando con dólares propios** | ≈ **$51.500/mes** (≈ $617.500/año) |
| Mensual pagando en pesos | ≈ $64.200/mes (≈ $770.000/año) |

- **No hay cuotas sin interés**: los consumos en moneda extranjera no se financian en cuotas con
  tarjeta argentina, y Wix cobra el período completo de una vez. El mensual **es** la cuota.
- Financiarlo mes a mes cuesta ≈ $182.000 más al año (~42%), sin la pérdida del voucher porque el
  dominio ya lo tiene.
- Se puede **cambiar el ciclo** de mensual a anual más adelante.
- **No hay que pagar nada hoy:** el sitio se construye y se prueba completo en el plan Free
  (preview, test site y la URL `wixstudio.com`). Solo se paga para publicarlo con su dominio.
- El dólar está en máximo nominal: **recalcular la cotización el día de la compra**.

### 12.7 A confirmar
- [ ] Precio exacto y familia de plan en el checkout del propio sitio (*Dashboard → Upgrade*).
      Los sitios del editor de Studio van a **planes Studio**, no a Premium del Wix Editor.
- [ ] Promoción vigente en la cuenta (suele haber descuento de primer año).

---

## 13. Pipeline de imágenes (y la regla de video)

**Objetivo:** que ninguna pieza se almacene en su tamaño original, para quedarse en Basic sin
sacrificar la calidad visual de la obra.

### 13.1 Lo que Wix ya hace y lo que no
- **Entrega:** resuelta. Wix sirve las imágenes por su servicio de imágenes y escala o recorta al
  vuelo (`scaleToFit`, `scaleToFill`, `crop` de la SDK Media API) desde el archivo guardado.
  **No hay que subir varios tamaños**: se guarda un master optimizado y el sitio pide la medida que
  necesita en cada breakpoint.
- **Almacenamiento:** NO resuelto. Cada archivo subido se hostea y **cuenta contra el storage del
  plan**. Un PNG de Procreate de 25 MB ocupa 25 MB para siempre.
- Conclusión: hay que **procesar antes de subir**. Comprimir después no existe.

### 13.2 Arquitectura: procesamiento en el browser
No se usa backend: Studio Two tiene cómputo fijo y sin garantía de librerías nativas de imagen.
El navegador de Nacho hace el trabajo, gratis.

```
[Nacho elige las piezas en NUESTRA dashboard page]
        ↓
1. createImageBitmap(file, { imageOrientation: 'from-image' })
2. dibujar en OffscreenCanvas redimensionado (ver medidas en 13.3)
3. convertToBlob({ type: 'image/webp', quality: 0.9 })        ← fallback JPEG
4. si el blob supera el techo de su categoría → bajar quality hasta 0.82, no menos
        ↓
5. HTTP endpoint propio → auth.elevate(files.generateFileUploadUrl)(mimeType, {
     fileName, filePath: 'proyectos/<slug>' })                 ← devuelve uploadUrl
        ↓
6. PUT uploadUrl?filename=<nombre> con el blob y su Content-Type
        ↓
7. guardar el id/url devuelto en `portada` / `galeria` del ítem
```
**El archivo original nunca sale de la máquina de Nacho** (él conserva sus masters).

Detalles de implementación:
- `files.generateFileUploadUrl` requiere scope **MANAGE-MEDIAMANAGER**: va por HTTP endpoint con
  `auth.elevate()`, no directo desde el browser.
- Para archivos > 10 MB o red mala existe **Generate File Resumable Upload URL**.
- Un archivo recién subido **no está disponible de inmediato**: manejar el estado "file ready"
  antes de escribir el ítem o mostrarlo.
- Errores a manejar: `SITE_QUOTA_EXCEEDED`, `FILE_SIZE_OVER_LIMIT`, `UNSUPPORTED_FILE_FORMAT`,
  `MISMATCH_MIME_TYPE`, `ZERO_FILE_SIZE`.
- Carpetas por slug de proyecto (`filePath`) para auditar y limpiar.

### 13.3 Parámetros — más conservadores que en un sitio común
Es obra de un ilustrador: la compresión agresiva se ve, sobre todo **banding en colores planos y
degradados**, que es exactamente lo que abunda en ilustración.

| Uso | Lado largo | Formato | Techo de peso |
|---|---|---|---|
| Portada de grid | 1600 px | WebP q0.88 | 250 KB |
| Pieza de galería | **2560 px** | WebP q0.90 | **600 KB** |
| Line art / colores planos | 2560 px | **WebP lossless o q0.95** | 800 KB |
| Retrato, imágenes de UI | 1200 px | WebP q0.85 | 150 KB |

- **Nunca bajar de q0.82.** Si una pieza no entra en su techo con q0.82, se sube igual y se anota:
  mejor un archivo pesado que obra arruinada.
- Revisar a ojo la primera tanda al 100% de zoom antes de fijar los parámetros definitivos.
- GIFs: convertirlos a **MP4/WebM** y tratarlos como video (§13.5). Un GIF de animación pesa
  10-20x más que el mismo clip en MP4.

### 13.4 Consecuencia de arquitectura ⚠️
**El pipeline solo funciona si Nacho sube por NUESTRA dashboard page.** Si usa el uploader nativo
del CMS, el Media Manager o `openMediaManager()`, el original entra sin procesar. Por lo tanto:
- El **uploader custom es parte obligatoria del alcance**, en **fase 1.5**, antes de que cargue
  obra real.
- Enseñarle **una sola vía de carga**.
- **Panel de auditoría** en el dashboard: `files.listFiles` / `searchFiles` devuelven
  `sizeInBytes`, así que se listan los archivos > 1 MB para detectar los que entraron por otra vía.
  Limpieza con `bulkDeleteFiles`.

### 13.5 La regla de video — crítica para un animador
**Nunca subir video al Media Manager de Wix.** Es lo que haría estallar el storage y forzaría un
plan más caro: un solo clip de animación en buena calidad puede pesar más que todo el resto del
portfolio junto.
- Los videos van a **Vimeo o YouTube** y se guarda la URL en `videoUrl`.
- `VideoEmbed` muestra el `videoPosterUrl` (una imagen, que sí pasa por el pipeline) y **carga el
  iframe solo al click**.
- Vimeo: sin publicidad ni sugeridos, mejor para portfolio, tiene costo. YouTube: gratis, con
  marca. **Decisión pendiente (§10)**, y es de Nacho.

### 13.6 Reglas complementarias de storage
- **Carga inicial masiva**: script local en Node con `sharp` que convierte la carpeta entera de
  Nacho, y después se sube con el mismo endpoint o con `bulkImportFile` desde URLs públicas.
  Lo corremos nosotros, no Nacho.
- El **CV en PDF** conviene optimizarlo también (PDFs de diseño suelen pesar de más).
- Revisar el consumo en *Settings → Manage Storage* del dashboard cada tanto.

---

## 14. Fuentes
- Studio Two Coding Features — https://dev.wix.com/docs/studio-two/get-started/studio-two-coding-features
- Compare Studio Two vs Wix Editor/Studio — https://dev.wix.com/docs/studio-two/get-started/compare-extending-websites-in-studio-two-to-wix-editor-and-wix-studio
- About Editor React Components — https://dev.wix.com/docs/studio-two/code-your-site/code-model/editor-react-components/about-editor-react-components
- Component Integration Patterns — https://dev.wix.com/docs/studio-two/code-your-site/code-model/component-integration-patterns
- About CMS Data Sources — https://dev.wix.com/docs/studio-two/code-your-site/work-with-data/data-sources-and-dynamic-pages/about-cms-data-sources
- About Dynamic Pages — https://dev.wix.com/docs/studio-two/code-your-site/work-with-data/data-sources-and-dynamic-pages/about-dynamic-pages
- Connect to Data vs. the Data API — https://dev.wix.com/docs/studio-two/code-your-site/work-with-data/introduction/connect-to-data-vs-the-data-api
- About the Data API — https://dev.wix.com/docs/studio-two/code-your-site/work-with-data/data-api/about-the-data-api
- About Collections — https://dev.wix.com/docs/studio-two/code-your-site/work-with-data/introduction/about-collections
- CMS: Collection Storage Limits and Quotas — https://support.wix.com/en/article/cms-understanding-collection-storage-limits-and-quotas
- Dashboard Pages (extensions framework) — https://dev.wix.com/docs/overview/extensions-framework/dashboard-extensions/dashboard-pages
- About Dashboard Pages (Studio Two) — https://dev.wix.com/docs/studio-two/code-your-site/build-a-custom-dashboard/dashboard-pages/about-dashboard-pages
- openMediaManager() — https://dev.wix.com/docs/sdk/host-modules/dashboard/open-media-manager
- Generate File Upload Url — https://dev.wix.com/docs/api-reference/assets/media/media-manager/files/generate-file-upload-url
- Media Manager Upload API — https://dev.wix.com/docs/api-reference/assets/media/media-manager/files/upload-api
- Wix Media: managing storage — https://support.wix.com/en/article/wix-media-managing-your-storage
- Roles & Permissions: inviting collaborators — https://support.wix.com/en/article/inviting-people-to-contribute-to-your-site
- Choosing a Studio Plan — https://support.wix.com/en/article/wix-studio-upgrading-sites
- Wix Billing for Site Plans — https://support.wix.com/en/article/wix-billing-for-premium-plans
- About Exporting Your Site to Code — https://dev.wix.com/docs/studio-two/code-your-site/export-your-site/about-exporting-your-site-to-code
