# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Repo GitHub: `cespinosal/Muros_Contenci-n` (GitHub sanitizó el nombre pretendido
"Muros_Contención" quitando la tilde). Clon local en
`C:\Users\cespi\Downloads\Muros de contención` — la carpeta local sí conserva el nombre completo en
español; solo el slug del repo remoto difiere.

Muros de Contención — Torres Telecom — herramienta web (en español) para el diseño y revisión
geotécnica/estructural de muros de contención en sitios de torres de telecomunicaciones. Hermana de
[[project_cimentaciones_fem]] (repo `Cimentaciones_FEM`, clon local "Hoja FEM") y de TSA: mismo
enfoque de un único `index.html` autocontenido, sin build ni `package.json`, mismos tokens de diseño
(Fluent 2 en claro, azul marino `#0A1628` + acento cian `#00D4FF` en oscuro).

La especificación completa original (16 bloques, ~1375 líneas) está en `prompt_original.txt` en la
raíz del repo — es la fuente de verdad para el alcance final del proyecto. `index.html` hoy solo
implementa un subconjunto (Fase 1, ver abajo); el resto del prompt describe fases futuras.

## Running / developing

- No build step. Abrir `index.html` directamente en el navegador (funciona por `file://`).
- No hay tests ni linter configurados. Para verificar cambios de lógica de cálculo, la forma más
  confiable es replicar la fórmula a mano (o en un script Python suelto) con los valores por
  defecto de `defaultState()` y comparar contra lo que muestra la pestaña "Revisiones geotécnicas".
- El estado se persiste en `localStorage` bajo la clave `murosContencion.v1`. Exportar/importar el
  estado completo como `.gcon` con "Guardar como…" / "Abrir proyecto…" (pestaña Proyecto del ribbon).
- El tema (claro/oscuro) se persiste en `localStorage` bajo `murosContencion.theme`.

## Arquitectura de `index.html`

`<style>` (tokens de diseño) + QAT + ribbon (pestañas + grupos de comandos, estilo Word/Excel,
reemplaza el sidebar+topbar original) + panel de datos (`.tab-panel`, uno por pestaña) + visor
2D/3D persistente + un único `<script>` en IIFE al final, organizado en bloques numerados con
banners `/* ==== N. NOMBRE ==== */` (buscar `====` para navegar el archivo):

1. **Constantes y utilidades** — `toRad`, `fmt`/`fmtT` (kg→t), `debounce`, `getPath`/`setPath` (para
   los bindings genéricos por `data-bind="a.b.c"`).
2. **Estado** — `defaultState()` reproduce el esquema completo del Bloque 3 del prompt original,
   *incluyendo campos que el motor de cálculo todavía no usa* (otros tipos de muro, diseño
   estructural, cargas de la torre — ver más abajo) para que los `.gcon` de hoy sigan siendo
   compatibles cuando esas fases se activen. `mergeState(def, loaded)` hace merge profundo recursivo
   (a diferencia de Cimentaciones FEM, aquí sí baja a los subobjetos anidados automáticamente).
3. **Persistencia** — `saveToStorage`/`loadFromStorage` (localStorage) + `guardarProyecto`/
   `abrirProyecto`/`cargarProyectoDesdeArchivo` (`.gcon` vía File System Access API, con reserva a
   descarga/`<input type=file>`).
4. **Motor de cálculo** (funciones puras, sin side effects sobre el DOM):
   - `calcGeometria(geom)` → geometría derivada del fuste trapezoidal (cara posterior vertical,
     cara frontal con batter si `tc < ts`, descompuesto en rectángulo+triángulo para el centroide).
   - `calcularPresiones(state)` → los 4 métodos del Bloque 4 (`rankineKa`/`coulombKa` como
     funciones auxiliares reutilizables; At-rest usa `suelo.Ks` directo, no la fórmula de Jaky;
     Fluido equivalente usa `state.EFP` sin concepto de Ka, que queda `null`) + incremento sísmico
     Mononobe-Okabe (reutiliza `coulombKa` con `phi→phi-psi`, `theta→theta-psi`) o "EFP sísmico"
     (`sismo.EFP_sism`, reinterpretado en kg/m³ métrico — el prompt original daba un valor en
     unidades imperiales incompatible con esta app). Con `beta > 0` (Rankine/Coulomb) reparte la
     resultante en componentes horizontal/vertical paralelas a la pendiente.
   - `calcularGeotecnia(state, pres)` → revisiones 1-5 del Bloque 6 (volteo, deslizamiento,
     capacidad portante + Terzaghi informativo, excentricidad, estabilidad global informativa),
     más el momento/fuerza del cerco perimetral si `cargas.cercoEnabled` (ver "Cargas de torre y
     cerco" abajo). Todos los momentos se toman respecto a la punta de la puntera (`x=0`).
     **Simplificación de sismo**: cuando `sismo.enabled`, el incremento sísmico ya viene sumado
     dentro de `Mov`/`Fh` y las revisiones comparan contra `FS.volteo_sismico`/`desliz_sismico` en
     vez de los estáticos — es un solo caso de carga combinado, no dos casos (estático puro +
     sísmico puro) evaluados por separado.
   - `calcularTodo()` encadena las tres anteriores y sincroniza `state.geom.heel` (campo calculado,
     de solo lectura en la UI).
5. **Dibujo 2D** (`dibujarMuro2D`) — SVG generado a mano (sin librería), con capas de suelo/relleno/
   agua, geometría del muro, fuerzas esquemáticas (triángulos de Pa/Pp/ΔPAE no están a escala de
   presión real, son indicativos), cerco perimetral, y la zona del tercio central con el punto de
   la resultante. El `viewBox` se recalcula en cada render contra el tamaño real en píxeles del
   panel (`clientWidth`/`clientHeight`), no una caja fija, para maximizar el tamaño del dibujo sin
   scrollbar.
   **5b. Vista 3D** (`actualizarModelo3D`, Three.js r128 + OrbitControls embebidos offline) — MVP:
   concreto + relleno + suelo + agua, largo de muro fijo en 6 m. Solo se renderiza mientras
   `uiState.viewMode==="3d"`.
6. **Render de tablas/badges** y **7. Render general** (`renderAll`/`renderAllFull`) — único punto
   de recálculo y repintado; cualquier cambio de `state` termina en `scheduleRecalc()`
   (`debounce(renderAll, 100)`). `renderProjectTitle()` autogenera el nombre del QAT superior desde
   `proyecto.nombre` + `proyecto.assetNum`.
7. **Bindings** — genérico vía `[data-bind]` (inputs/selects/checkboxes) más bindings manuales:
   preset de sobrecarga, botones de elección del ribbon (`syncMethodButtons()` refleja
   `metodo_presion`/`sismo.metodo` con clase `.active` y muestra/oculta los campos específicos de
   cada método — no son `[data-bind]` porque no son checkbox/select), toolbar 2D/3D, scroll del
   mouse sobre inputs numéricos enfocados (±0.1 por tick).
8. **Tema** e **9. Init**.

### Convenciones a mantener

- Unidades internas: kg y m (sistema técnico), igual que el prompt original. Solo se convierte a
  toneladas (`fmtT`) para mostrar en pantalla, comparando contra `QADM` (t/m²).
- El origen del sistema de coordenadas geométrico interno (`calcGeometria`, `calcularGeotecnia`,
  `dibujarMuro2D`) es siempre `x=0` en el borde exterior de la puntera (toe), creciendo hacia el
  talón (heel). No cambiar esta convención sin revisar las tres funciones a la vez.
- `Wp = gamma_f * toe * Df` (peso del suelo sobre la puntera) reproduce literalmente la fórmula del
  Bloque 6 del prompt original, que no resta el espesor de la zapata `tf` a `Df`. Es una
  simplificación del prompt, no un error de transcripción — si se revisa/corrige, actualizar este
  comentario y el del código (`calcularGeotecnia`).
- Cualquier campo nuevo en `defaultState()` no requiere tocar `mergeState` (el merge ya es profundo
  y recursivo), pero sí hay que decidir si el motor de cálculo lo usa ya o si queda "reservado" para
  una fase futura (documentarlo con un comentario y, si aplica, una nota `.future-note` en la UI).
- **Cargas de la torre (`cargas.Pu_torre`/`Mu_torre`/`Vu_torre`/`dist_torre`)**: a pedido explícito
  del usuario (2026-08-03), estos campos se capturan y persisten en el `.gcon` pero **no** afectan
  el cálculo geotécnico todavía — el prompt original daba dos fórmulas alternativas sin especificar
  cómo combinarlas con `Vu_torre`, y se decidió no adivinar. Si se retoma, resolver primero esa
  ambigüedad con el usuario antes de integrarlas a `calcularGeotecnia`.
- **Cerco perimetral (`cargas.P_cerco`/`h_cerco`)**: sí está integrado (a diferencia de torre). Es
  una interpretación razonable del prompt original (que no daba la fórmula explícita), no una cita
  textual: `P_cerco` se trata como carga horizontal de viento actuando a media altura del cerco por
  encima de la corona (brazo = `tf+H+h_cerco/2` respecto al toe). Documentado también en el
  comentario de `calcularGeotecnia`.
- **Sistema de unidades (`sistemaUnidades`, botón "kgf"/"SI" en la QAT)**: `state` SIEMPRE guarda
  kgf-técnico internamente (kg, kg/m, kg/m², kg/m³, kg·m/m, kg/cm² — el sistema ya verificado a mano
  contra Python); el motor de cálculo nunca ve valores en SI. El toggle convierte tanto los
  RESULTADOS (`fmtFuerza`/`fmtMomento`/`fmtPresionSuelo`) como los campos EDITABLES marcados con
  `data-unit="F|Q|S"` (`UNIT_FACTORS`: F=kgf-familia→kN-familia ×0.00980665, Q=tf/m²→kPa ×9.80665,
  S=kgf/cm²→MPa ×0.0980665) — la conversión de los editables ocurre en la frontera
  display↔state: `syncInputsFromState()` convierte kgf→SI solo para mostrar, y el handler de
  `bindAll()` convierte lo que el usuario tecleó de vuelta a kgf antes de `setPath()`. Las etiquetas
  `<span class="unit-tag" data-tec="…" data-si="…">` se sincronizan con `syncUnitLabels()`. Campos
  de longitud/ángulo/adimensionales (H, ts, Bf, φ, Ka, FS, kh…) no llevan `data-unit` — ya son
  compatibles con SI tal cual. `cargas.Pu_torre/Mu_torre/Vu_torre` quedan fuera a propósito: el
  prompt original ya los definió en kN/kN·m (no kgf), así que no tienen conversión kgf→SI que hacer.

## Roadmap (fases, ver `prompt_original.txt` para el detalle completo de fórmulas)

Estado real al 2026-08-03 — varias cosas se adelantaron fuera de orden a pedido del usuario, además
de la Fase 1 "oficial":

- **Fase 1 (implementada)** — núcleo de la app, muro Cantilever únicamente, método de Rankine
  únicamente (sin sismo), revisiones geotécnicas 1-5, vista 2D, persistencia `.gcon`/localStorage,
  tema claro/oscuro.
- **Adelantos fuera de orden (implementados)** — rediseño de arquitectura a ribbon estilo
  Microsoft 365 (reemplaza sidebar+topbar); MVP de vista 3D (Three.js r128 embebido: concreto +
  relleno + suelo + agua, largo de muro fijo en 6 m, sin acero ni vistas Alzado/Planta/Sección);
  guardar/abrir con extensión `.gcon` vía File System Access API; varios pulidos del visor 2D
  (auto-ajuste sin scrollbar, cotas con líneas de proyección, talud desde la corona, etc.).
- **Fase 2 (implementada, 2026-08-03) — sin carga vehicular (excluida a pedido del usuario)**:
  selector funcional de los 4 métodos de presión (Rankine/Coulomb/At-rest/Fluido equivalente),
  sismo Mononobe-Okabe (o "EFP sísmico") integrado a volteo/deslizamiento con umbrales de FS
  sísmicos, y cerco perimetral integrado al cálculo. Cargas de la torre (Pu/Mu/Vu/dist) se
  capturan en el formulario pero NO se integran al cálculo (ambigüedad del prompt original sin
  resolver, ver "Convenciones a mantener" arriba). Verificado con réplica en Python de
  Coulomb+Mononobe-Okabe+cerco y de At-rest+EFP sísmico contra los valores por defecto — coinciden
  exactamente.
- **Fase 3 (pendiente) — DISEÑO ESTRUCTURAL ACI 318-19**: flexión y cortante del fuste, diseño de
  la zapata (talón y puntera), revisión de la llave de corte, combinaciones de carga ASCE 7-22
  LRFD/ASD (Bloques 7, 16). Es la revisión estructural propiamente dicha — hoy la app solo cubre
  geotecnia (volteo/deslizamiento/capacidad/excentricidad/estabilidad global); no hay ningún
  cálculo de acero de refuerzo, cuantías, ni verificación de secciones de concreto todavía.
- **Fase 4 (resto pendiente)** — sobre el MVP ya adelantado: acero de refuerzo en 3D con
  longitudes de desarrollo ACI, vistas preestablecidas (Isométrica/Alzado/Planta/Sección A-A),
  largo de muro editable (hoy fijo en 6 m), export a PNG (Bloques 8, 9).
- **Fase 5 (pendiente)** — Diagramas Plotly (presiones, M/V del fuste — este último depende de que
  Fase 3 exista primero), comparador de métodos, análisis de sensibilidad (Bloques 10, 12).
- **Fase 6 (pendiente)** — Exportación de memoria de cálculo a Word (`docx.js` + `FileSaver.js`
  desde CDN, mismo patrón que Cimentaciones FEM) — necesita Fase 3 para tener algo estructural que
  reportar (Bloque 14).
- **Fase 7 (pendiente)** — Otros tipos de muro: Gravedad, Contrafuertes, Gavión (Bloque 11).
- **Fase 8 (pendiente)** — Pulido: auto-dimensionado, validación de rangos con tooltips, proyectos
  recientes, accesibilidad completa (Bloque 12, 13, 15).

No iniciar una fase sin confirmación explícita del usuario — el patrón acordado en este proyecto es
igual al de [[project_geocim]] y [[project_tsa]]: construir y verificar una fase a la vez. En la
práctica el usuario también pide adelantos puntuales de fases futuras a mitad de otra fase (ver
[[project_muros_contencion]] memoria) — eso es aceptable, pero cada adelanto debe quedar anotado
aquí para no perder de vista qué falta de la fase "oficial" correspondiente.
