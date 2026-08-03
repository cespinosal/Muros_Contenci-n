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
  estado completo como `.json` con los botones "Guardar JSON" / "Cargar JSON" del topbar.
- El tema (claro/oscuro) se persiste en `localStorage` bajo `murosContencion.theme`.

## Arquitectura de `index.html`

`<style>` (tokens de diseño) + marcado de pestañas (`.tab-panel`, una por grupo del sidebar) + un
único `<script>` en IIFE al final, organizado en bloques numerados con banners
`/* ==== N. NOMBRE ==== */` (buscar `====` para navegar el archivo):

1. **Constantes y utilidades** — `toRad`, `fmt`/`fmtT` (kg→t), `debounce`, `getPath`/`setPath` (para
   los bindings genéricos por `data-bind="a.b.c"`).
2. **Estado** — `defaultState()` reproduce el esquema completo del Bloque 3 del prompt original,
   *incluyendo campos que el motor de cálculo todavía no usa* (sismo, otros métodos de presión,
   otros tipos de muro, diseño estructural) para que los `.json` de hoy sigan siendo compatibles
   cuando esas fases se activen. `mergeState(def, loaded)` hace merge profundo recursivo (a
   diferencia de Cimentaciones FEM, aquí sí baja a los subobjetos anidados automáticamente).
3. **Persistencia** — `saveToStorage`/`loadFromStorage`/`descargarJSON`/`cargarJSONFile`.
4. **Motor de cálculo** (funciones puras, sin side effects sobre el DOM):
   - `calcGeometria(geom)` → geometría derivada del fuste trapezoidal (cara posterior vertical,
     cara frontal con batter si `tc < ts`, descompuesto en rectángulo+triángulo para el centroide).
   - `calcularPresiones(state)` → Rankine (Bloque 4, método 1 únicamente en esta fase). Con
     `beta > 0` usa la formulación generalizada (Ka en función del talud) y reparte la resultante en
     componentes horizontal/vertical paralelas a la pendiente.
   - `calcularGeotecnia(state, pres)` → revisiones 1-5 del Bloque 6 (volteo, deslizamiento,
     capacidad portante + Terzaghi informativo, excentricidad, estabilidad global informativa).
     Todos los momentos se toman respecto a la punta de la puntera (`x=0` en el sistema interno).
   - `calcularTodo()` encadena las tres anteriores y sincroniza `state.geom.heel` (campo calculado,
     de solo lectura en la UI).
5. **Dibujo 2D** (`dibujarMuro2D`) — SVG generado a mano (sin librería), con capas de suelo/relleno/
   agua, geometría del muro, fuerzas esquemáticas (triángulos de Pa/Pp no están a escala de presión
   real, son indicativos) y la zona del tercio central con el punto de la resultante.
6. **Render de tablas/badges** y **7. Render general** (`renderAll`/`renderAllFull`) — único punto
   de recálculo y repintado; cualquier cambio de `state` termina en `scheduleRecalc()`
   (`debounce(renderAll, 100)`).
7. **Bindings** — genérico vía `[data-bind]` (inputs/selects/checkboxes) más un puñado de bindings
   manuales (preset de sobrecarga, tabla de revisiones, toolbar del SVG, topbar, tabs).
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

## Roadmap (fases, ver `prompt_original.txt` para el detalle completo de fórmulas)

- **Fase 1 (implementada)** — núcleo de la app, muro Cantilever únicamente, método de Rankine
  únicamente (sin sismo), revisiones geotécnicas 1-5, vista 2D, persistencia JSON/localStorage,
  tema claro/oscuro.
- **Fase 2** — Coulomb, At-rest, Fluido equivalente (selector de 4 métodos con recálculo en tiempo
  real) + sismo Mononobe-Okabe + cargas vehiculares/de la torre/cerco perimetral integradas al
  cálculo (Bloques 4, 17).
- **Fase 3** — Diseño estructural ACI 318-19: flexión y cortante del fuste, diseño de zapata
  (talón/puntera), llave de corte, combinaciones ASCE 7-22 LRFD/ASD (Bloques 7, 16).
- **Fase 4** — Vista 3D con Three.js r128 embebido (sin CDN) + acero de refuerzo 3D (Bloques 8, 9).
- **Fase 5** — Diagramas Plotly (presiones, M/V del fuste), comparador de métodos, análisis de
  sensibilidad (Bloques 10, 12).
- **Fase 6** — Exportación de memoria de cálculo a Word (`docx.js` + `FileSaver.js` desde CDN,
  mismo patrón que Cimentaciones FEM) (Bloque 14).
- **Fase 7** — Otros tipos de muro: Gravedad, Contrafuertes, Gavión (Bloque 11).
- **Fase 8** — Pulido: auto-dimensionado, validación de rangos con tooltips, proyectos recientes,
  accesibilidad completa (Bloque 12, 13, 15).

No iniciar una fase sin confirmación explícita del usuario — el patrón acordado en este proyecto es
igual al de [[project_geocim]] y [[project_tsa]]: construir y verificar una fase a la vez.
