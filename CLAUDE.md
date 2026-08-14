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
   - **Relleno estratificado (`suelo.capasRelleno[]`, 2026-08-10, estilo GeoCim)**: el backfill ya
     no es un único `gamma_r/phi_r/c_r` plano, sino una lista de estratos
     `{nm,hatch,color,espesor,gamma,phi,c}` — mismo esquema visual que el `profile` de GeoCim
     (`C:\Users\cespi\Downloads\GeoCim\index.html`, de donde se portaron literalmente
     `HATCH_DEFS`/`COLOR_PALETTE`/`hatchPatternSVG` para las 12 texturas de suelo). La UI es una
     tarjeta compacta por estrato (pestaña Suelo y materiales, `renderCapasRelleno()` →
     `#capasRellenoList`) con textura+nombre+editar/eliminar, más un modal `#strat-modal` (nombre,
     **espesor** del estrato, grilla de texturas, paleta de color, γ/c/φ) para agregar/editar. El
     modal usa `toDisplay`/`toBase` con las categorías `longitud`/`pesoVol`/`presion` del sistema de
     unidades — no las suyas propias de GeoCim. **Importante:** el script vive en un IIFE, así que el
     modal usa `addEventListener` en vez del `onclick`/`oninput` inline que tiene GeoCim (esas
     funciones no son globales aquí). Solo el RELLENO se estratifica — el suelo de fundación
     (`gamma_f/phi_f/c_f`) sigue siendo un único material, a propósito.

     **Apilado de abajo hacia arriba (2026-08-10, mismo día, a pedido explícito del usuario):** cada
     estrato solo guarda su ESPESOR propio, no una posición. El arreglo `capasRelleno` va "de arriba
     hacia abajo" en orden de PANTALLA/creación (índice 0 = "Estrato 1" = el más cercano a la
     corona) — pero `addEstrato()` siempre EMPUJA el estrato nuevo al FINAL del arreglo (`capas.push`),
     es decir en la BASE, junto a la zapata; los estratos que ya existían no se tocan, simplemente
     quedan recorridos hacia arriba porque su posición se deriva, no se guarda. Ejemplo verificado
     (el mismo que dio el usuario): con "Estrato 1" solo (espesor 3) va de 0 a 3; al agregar "Estrato
     2" (espesor 3) éste se agrega en la base (0 a 3) y "Estrato 1" queda recorrido a 3-6 — sin tocar
     su registro, solo cambia lo que `derivarPosicionesCapas()` calcula.
     `derivarPosicionesCapas(capas)` recorre el arreglo EN REVERSA (del último índice, pegado a la
     zapata con `from=0`, hacia el índice 0) acumulando espesores, y devuelve `from`/`to` (0 = base)
     por estrato sin mutar `capas`. `resolverCapas(capas, Hobjetivo)` llama a
     `derivarPosicionesCapas` y luego RECORTA cada estrato a la ventana `[0, Hobjetivo]` en esa
     coordenada (`from=min(from,Hobjetivo)`, `to=min(to,Hobjetivo)`) — el recorte por rango (no por
     presupuesto acumulado en orden de arreglo) es lo que garantiza que el sobrante, cuando la suma >
     Hobjetivo, se quite SIEMPRE de ARRIBA (lo que quedaría por encima de la corona) y se conserve
     completo lo que está pegado a la zapata, sin importar en qué posición del arreglo esté cada
     estrato. Devuelve la lista en el mismo orden que `capas` (índice 0 = el de más arriba que
     sobreviva), con espesor ya resuelto en `h` — nunca falla ni deja huecos. El resto del motor de
     cálculo (integración de Pa top-down, `dibujarMuro2D`, etc.) no cambió: solo consume `h` en orden
     de arreglo, igual que antes.

     **`alturaRellenoEfectiva(suelo, H)` — altura real del relleno, ya no `geom.Hr` (2026-08-10,
     corrección el mismo día que se agregó el modal, a pedido del usuario):** el viejo campo
     editable `geom.Hr` desapareció. La altura real del relleno ahora es
     `Math.min(sumaDeEstratos, geom.H)` — si el usuario captura MENOS que `H`, esa altura menor se
     queda tal cual (el fuste se dibuja con un tramo expuesto arriba, sin relleno) y el cálculo usa
     esa altura real, NO se estira artificialmente hasta `H` como pasaba antes; si captura MÁS que
     `H`, se sigue recortando como ya ocurría. El campo `Hr` en la UI (pestaña Geometría del muro)
     quedó como `disabled`, poblado en `renderGeomWarning()` — es de solo lectura, igual que `heel`.
     `calcularPresiones` usa esta altura efectiva para TODO lo relacionado al relleno (Pa integrado
     capa por capa, Pq_h de la sobrecarga, ΔPAE sísmico — las tres formulas usaban `geom.H`
     directamente antes de este cambio, ahora usan la altura efectiva) y la devuelve en `pres.H`
     para que `calcularGeotecnia` reutilice el MISMO valor en los brazos de momento de `Pq_h`/`ΔPAE_h`
     de `Mov` y en el peso `Wr` (antes `Wr` se resolvía aparte contra `geom.Hr`) — un solo número,
     sin riesgo de que diverja entre fuerza y brazo. `dibujarMuro2D`/`actualizarModelo3D` la calculan
     igual para posicionar el tope del relleno — por eso el tramo expuesto del fuste (si lo hay)
     aparece automáticamente en 2D y 3D sin código de dibujo extra. La integración de Pa por capas
     descompone el diagrama de cada estrato en rectángulo (presión constante = la del tope, según
     el Ka de ESE estrato) + triángulo (el incremento hasta la base), sumando fuerza y momento
     respecto a la base del muro — de ahí sale `pres.arm` (brazo real de Pa, ya no fijo en `H/3`,
     aunque con un solo estrato que cubra toda la altura efectiva la fórmula se reduce exactamente
     al caso verificado de siempre). Simplificaciones documentadas en el propio código: la cohesión
     de cada estrato (`capa.c`) se captura pero no se usa (igual que `suelo.c_r` antes); el método
     Fluido equivalente y el incremento sísmico Mononobe-Okabe NO se estratifican (EFP ya es un
     valor único por definición; el sismo usa la φ del PRIMER estrato + el peso volumétrico promedio
     ponderado del relleno como referencia única). `migrarSueloViejo()` tiene TRES ramas, para no
     perder valores al abrir un `.gcon`/localStorage de cualquier versión previa: (1) sintetiza un
     estrato único con `espesor=H` a partir de un `gamma_r/phi_r/c_r` plano (formato de antes del
     relleno estratificado); (2) copia `h`→`espesor` tal cual (formato intermedio de la tabla
     editable que existió brevemente antes del modal — ya traía espesor por capa, en el mismo orden
     de arreglo que ahora); (3) `to-from`→`espesor` (formato del modal con posición acumulada
     `from`/`to`, usado un rato el mismo día 2026-08-10 antes de pasar al apilado desde abajo) —
     el orden del arreglo (índice 0 = el más cercano a la corona) es igual en los tres formatos
     viejos, así que ninguna rama necesita reordenar, solo calcular el espesor. Ninguno de los
     formatos viejos tenía tampoco el problema de `geom.Hr` porque ese campo simplemente se ignora
     al fusionar con el nuevo `defaultState()`, que ya no lo declara.

     La tarjeta de cada estrato (`renderCapasRelleno()`) muestra el rango EFECTIVO ya resuelto
     contra la altura efectiva (vía `derivarPosicionesCapas` + el mismo recorte `[0,Hobjetivo]` que
     usa `resolverCapas`) — con "(recortado)" cuando el total se pasa de H, y "fuera de H" si el
     estrato queda totalmente excluido por el recorte (por ejemplo, si se agrega un segundo estrato
     sin subir H, el primero puede quedar 100% por encima de la corona). Ya no existe un caso
     "(extendido)": antes, un solo estrato más corto que H se mostraba como "0–3 m" en la tarjeta
     mientras el dibujo y el cálculo lo estiraban a
     "0–H m" — inconsistencia que reportó el usuario y que llevó primero a corregir el TEXTO de la
     tarjeta, y ese mismo día a retirar el estiramiento por completo (ver párrafo de
     `alturaRellenoEfectiva` arriba).
   - `calcularPresiones(state)` → los 4 métodos del Bloque 4 (`rankineKa`/`coulombKa` como
     funciones auxiliares reutilizables; At-rest usa `suelo.Ks` directo, no la fórmula de Jaky;
     Fluido equivalente usa `state.EFP` sin concepto de Ka, que queda `null`) + incremento sísmico
     Mononobe-Okabe (reutiliza `coulombKa` con `phi→phi-psi`, `theta→theta-psi`) o "EFP sísmico"
     (`sismo.EFP_sism`, reinterpretado en kg/m³ métrico — el prompt original daba un valor en
     unidades imperiales incompatible con esta app). Con `beta > 0` (Rankine/Coulomb) reparte la
     resultante en componentes horizontal/vertical paralelas a la pendiente.

     **H' en vez de H para Pa/Pq_h/ΔPAE (2026-08-10, verificado contra el Ejemplo 8.1 de Das —
     Fundamentos de Ingeniería de Cimentaciones, p. 390-393):** el empuje activo NO se integra solo
     sobre la altura del relleno (H) — se integra sobre el plano vertical que pasa por el talón,
     desde el FONDO de la zapata hasta donde ese plano cruza la superficie del talud (`Hprime = tf +
     H + heel·tan(beta)`, exactamente la fórmula `H'=H1+H2+H3` del libro). Se arma extendiendo la
     lista de capas resueltas (`capasExt`) con dos tramos ficticios que toman prestadas las
     propiedades del estrato más cercano (no hay estrato real definido ahí): el talud sobre la
     corona (si `beta>0`, usa el PRIMER estrato) y el espesor de la zapata (usa el ÚLTIMO estrato).
     Con `beta=0` y `tf=0` esta extensión desaparece y `Hprime=H` — pero en la práctica `tf` casi
     nunca es 0, así que Pa YA CAMBIÓ incluso en el caso por default (antes de este cambio se
     ignoraba el tramo de la zapata por completo). `pres.arm` (brazo de Pa) queda medido desde el
     FONDO de la zapata en vez de desde el tope — `calcularGeotecnia` ya no le suma `tf` aparte en
     `Mov`. `pres.H` (altura real del relleno, para `Wr`) y `pres.Hprime` (para los brazos de
     `Pq_h`/`ΔPAE_h`) se devuelven por separado porque sirven para cosas distintas.

     **Cohesión en la presión pasiva (ecuación 8.9, Das):** `Pp` ahora suma
     `2·c_p·√Kp·Df` al término friccionante de siempre — antes solo tenía el primero. `suelo.c_p`
     es un campo nuevo (cohesión de la zona pasiva, independiente de `c_f` por si el usuario quiere
     modelar un suelo distinto ahí, igual que ya pasaba con `gamma_p`/`phi_p`).

     Estas tres correcciones (más la cohesión en deslizamiento de abajo) se verificaron reproduciendo
     el Ejemplo 8.1 completo del libro (H=6, x1=0.5/x2=0.7/x3=0.7/x4=2.6 m, D=1.5 m, β=10°, γ1=18/
     φ1=30°/c1=0 kN, γ2=19/φ2=20°/c2=40 kN/m²) contra una réplica independiente en Python: FS
     volteo=2.91 (libro 2.95), deslizamiento=2.74 (libro 2.70), capacidad de carga=2.79 (libro 2.98)
     — mucho más cerca que antes (3.25/1.30/4.96). La brecha restante es la Ka por fórmula cerrada
     vs. tabla interpolada del libro (~1%) y que `Wr` todavía no incluye el triángulo de suelo sobre
     el talud (zona ⑤ de la figura 8.12) — mejora pendiente, no pedida en esta ronda.
   - `calcularGeotecnia(state, pres)` → revisiones 1-5 del Bloque 6 (volteo, deslizamiento,
     capacidad portante, excentricidad, estabilidad global informativa), más el momento/fuerza del
     cerco perimetral si `cargas.cercoEnabled` (ver "Cargas de torre y cerco" abajo). Todos los
     momentos se toman respecto a la punta de la puntera (`x=0`).
     **Simplificación de sismo**: cuando `sismo.enabled`, el incremento sísmico ya viene sumado
     dentro de `Mov`/`Fh` y las revisiones comparan contra `FS.volteo_sismico`/`desliz_sismico` en
     vez de los estáticos — es un solo caso de carga combinado, no dos casos (estático puro +
     sísmico puro) evaluados por separado.

     **Cohesión en deslizamiento (ecuación 8.11, Das, 2026-08-10):** la resistencia `R` ahora suma
     `Bf·k2·suelo.c_f` (antes solo tenía la fricción `Fr=mu·N` y la pasiva). `suelo.k2` es un campo
     nuevo, mismo patrón que `suelo.mu`: vacío = 2/3 (igual que `k1` en `mu`, "Sea k1=k2=2/3" del
     libro), o se puede sobreescribir.

     **Capacidad de carga — Meyerhof/Vesic completo en vez de Terzaghi liso (ecuaciones 8.22-8.24,
     Das, 2026-08-10):** `qult` ahora incluye factores de profundidad (`Fcd`, `Fqd`, `Fgd=1`) e
     inclinación de la carga (`Fci=Fqi=(1-ψ°/90°)²`, `Fgi=(1-ψ/φf)²`, con
     `ψ=atan(Fh/N)`) y usa el ancho EFECTIVO `B'=Bf-2|e|` en el tercer término, en vez del `Bf`
     completo. `Nc`/`Nq`/`Nγ` no cambiaron (ya coincidían exactamente con las tablas del libro).
     Este cambio por sí solo es el que más baja `FS_carga` respecto al Terzaghi liso de antes (el
     ejemplo verificado pasó de FS=4.96 a FS=2.79, mucho más cerca del 2.98 del libro).

     **`cargas.qsEnabled` ahora arranca en `false`** (2026-08-10): antes el estado por default traía
     una sobrecarga de 500 kg/m² activada, lo que ensuciaba cualquier comparación contra un ejemplo
     de libro que no la tuviera (hubo que descubrir esto a mano al no cuadrar un resultado). Ahora
     hay que activarla a propósito desde el ribbon si el caso la necesita.
   - `calcularTodo()` encadena las tres anteriores y sincroniza `state.geom.heel` (campo calculado,
     de solo lectura en la UI).
5. **Dibujo 2D** (`dibujarMuro2D`) — SVG generado a mano (sin librería), con capas de suelo/relleno/
   agua, geometría del muro, fuerzas esquemáticas (triángulos de Pa/Pp/ΔPAE no están a escala de
   presión real, son indicativos), cerco perimetral, y la zona del tercio central con el punto de
   la resultante. El `viewBox` se recalcula en cada render contra el tamaño real en píxeles del
   panel (`clientWidth`/`clientHeight`), no una caja fija, para maximizar el tamaño del dibujo sin
   scrollbar. **Relleno estratificado en el dibujo (2026-08-10)**: ya no es un solo polígono de
   color plano — se resuelve `suelo.capasRelleno` contra `Hr` (`resolverCapas`, la misma función
   del motor de cálculo, que ahora conserva `nm/hatch/color` además de `gamma/phi/c/h` para que el
   dibujo no tenga que recalcular nada por su cuenta) y cada estrato se pinta con su propia textura
   SVG (`hatchPatternSVG`, inyectada vía `defs.insertAdjacentHTML` — funciona porque `defs` ya es
   un nodo SVG real) + una etiqueta con nombre y φ si el estrato mide más de 0.35 m (para no
   saturar de texto los estratos delgados). El talud (si `beta>0`) solo levanta el borde superior
   del PRIMER estrato (el que aflora); los estratos de abajo son horizontales. **Ojo:** el
   triángulo de Pa/ΔPAE (sección "Fuerzas" más abajo en esta misma función) tiene que usar esta
   misma `Hr`, NO `H` (altura del fuste) — se corrigió un bug (2026-08-10) donde el triángulo seguía
   dibujándose hasta la corona aunque el relleno real fuera más corto, viéndose como si el suelo
   cubriera todo el fuste cuando no era así (el número de Pa ya estaba bien calculado con `Hprime`,
   solo el dibujo del triángulo tenía la altura equivocada).

   **Bug propio + arreglo: el suelo sobre la puntera no seguía la cara inclinada del fuste con
   batter (2026-08-14):** el polígono `soilPts` (suelo de fundación + franja sobre la puntera, una
   sola figura sin costuras) usaba una línea VERTICAL fija en `x=toe` como borde derecho de la
   franja de `tf` a `Df` — correcto solo cuando `ts=tc` (sin batter). El usuario reportó "el área de
   Wp... parece que está mal, por la parte inclinada del muro"; se verificó primero que el VALOR de
   `Wp` (entonces `γf·toe·Df`) seguía siendo correcto en todos los casos probados (batter normal, sin
   batter, batter invertido, Df>H) — el problema inicial no estaba en el número, era el dibujo. Se
   confirmó visualmente con una captura headless con batter fuerte (ts=0.90, tc=0.25): un hueco
   triangular sin rellenar (fondo del canvas asomando) entre la línea vertical y la cara real del
   fuste, que se recorre hacia adentro/afuera al subir. Se corrigió calculando `xFrenteEnDf` (el
   punto donde la RECTA real del frente del fuste — la misma que va de `(toe,tf)` a
   `(xBack-tc,tf+H)` — cruza la altura `Df`, acotada a `tf+H` por si `Df` fuera mayor que todo el
   fuste) y usándolo como el vértice superior de esa franja en vez de `toe` fijo — con `ts=tc` da
   exactamente `toe` (sin cambios en el caso sin batter, verificado matemáticamente:
   `xBack-tc-toe = ts-tc = 0`).

   **[Extensión, mismo día] La fórmula de `Wp` también se corrigió, no solo el dibujo:** el usuario
   pidió explícitamente "revisa el peso y la fórmula también". Se comparó cuantitativamente (5 casos,
   headless Chrome) el valor de `γf·toe·Df` contra el área REAL del trapecio (la misma
   `xFrenteEnDf` del fix del dibujo, restando `tf` como corresponde) — la diferencia resultó grande y
   de signo variable: de **-37% a +36%** según la combinación de `tf`/`Df`/batter (el caso "sin
   batter" YA tenía ~23% de error por sí solo, porque la fórmula vieja nunca restaba `tf` —
   contaba de más desde el fondo de la zapata, no desde su tope; el batter solo modula ese error,
   a veces lo compensa por casualidad, a veces lo agrava). El usuario confirmó actualizar la
   fórmula. Ahora `calcularGeotecnia` calcula `areaWp = ½·(toe+xFrenteEnDfWp)·(Df-tf)` (trapecio
   real, con `xFrenteEnDfWp` calculado igual que en `dibujarMuro2D`) y `Wp = γf·areaWp` — devuelve
   también `areaWp`/`xFrenteEnDfWp` en el objeto `geo` para que `formulaWp()` (ahora 2 renglones,
   desglosando el área antes del peso) y `highlightRegionPoints()` (el resaltado de `Wp` ahora SÍ
   seguí la misma geometría trapezoidal, ya no es el rectángulo simple) lean el mismo cálculo sin
   duplicar nada. Re-verificado contra el Ejemplo 8.1 de Das: FS volteo 2.989 (antes 2.998, libro
   2.95), deslizamiento 2.744 (antes 2.758, libro 2.70), capacidad de carga **2.918** (antes 2.839,
   libro 2.98 — mejoró, quedó más cerca del libro que antes). **Why:** confirma que la corrección era
   la dirección correcta, no solo más "honesta" geométricamente sino también más precisa contra la
   fuente autoritativa ya usada para verificar toda la app.
   **5b. Vista 3D** (`actualizarModelo3D`, Three.js r128 + OrbitControls embebidos offline) — MVP:
   concreto + relleno + suelo + agua, largo de muro fijo en 6 m. Solo se renderiza mientras
   `uiState.viewMode==="3d"`.

   **Suelo alrededor del muro como UNA sola malla, no 3 cajas (2026-08-14):** el usuario mandó una
   captura del 3D marcando dos problemas: (1) faltaba por completo el tramo de suelo que cubre la
   puntera por arriba (la región de `Wp`, ver `formulaWp`/`highlightRegionPoints` en el bloque de
   presiones — en 3D nunca se dibujaba esa franja); (2) el "suelo de cimiento" se veía partido "en 2
   secciones" (una caja rectangular suelta a la izquierda y otra pieza como cuña, marcada con una X
   en la captura). Las 3 cajas viejas (`addBox` para "suelo de fundación" y=[-0.75,0] ancho completo,
   "suelo frente al muro" x=[-1.2,0] y=[0,Df], y el parche "tapa el hueco junto al borde derecho"
   x=[Bf,Bf+1] y=[0,tf]) se reemplazaron por UNA sola malla (`soilShape`, `THREE.ExtrudeGeometry`,
   mismo patrón que ya usa el fuste trapezoidal): un perfil 2D que contornea la franja bajo todo,
   sube hasta `Df` a la izquierda Y sobre la puntera (de `-1.2` a `toe`, un solo tramo horizontal a
   la misma altura `Df` — esto es lo que cubre `Wp`), baja a `y=0` desde `toe` hasta `Bf`, y vuelve a
   subir hasta `tf` en la franja derecha. Al ser UNA sola malla no hay costuras entre piezas
   adyacentes (la causa más probable del efecto "2 secciones": mallas separadas con normales/
   sombreado ligeramente distintos en la unión, aunque geométricamente coincidieran).

   **[Corrección, mismo día] La primera versión NO recortaba la huella de la zapata:** el primer
   intento subía derecho de `y=0` a `y=Df` en `x=[0,toe]`, apostando a que la zapata opaca (dibujada
   aparte) taparía el traslape con el polígono de suelo. El usuario mandó una SEGUNDA captura
   marcando un bloque gris sólido encimado justo sobre la puntera — el supuesto de "la zapata opaca
   tapa el traslape" no se sostuvo (probablemente por cómo Three.js ordena el dibujo de mallas
   `transparent:true` contra mallas opacas, o algo similar; no se investigó la causa exacta porque
   la solución de fondo era más simple). Se corrigió recorriendo el contorno alrededor de la huella
   real de la zapata en vez de atravesarla: sube hasta `Df` en `x=toe` pero baja solo hasta `tf`
   (no hasta 0), retrocede en `y=tf` hasta `x=0` (la base de la franja de `Wp`, apoyada sobre la
   zapata), y ahí sí baja a `y=0` para seguir bajo toda la huella de la zapata hasta `Bf`. Verificado
   con prueba punto-en-polígono a mano (un punto dentro de la huella de la zapata da "afuera"; un
   punto en la franja de `Wp` da "adentro") antes de aplicar el cambio, y con captura de pantalla
   headless (WebGL vía `--screenshot`, ruta ABSOLUTA — una ruta relativa dio "Acceso denegado" en
   este entorno) confirmando que el bloque gris ya no aparece. **Lección:** cuando dos mallas
   (una opaca, una transparente) podrían ocupar el mismo volumen, no asumir que el z-testing normal
   "simplemente lo resuelve" — recortar el traslape explícitamente en la geometría es más seguro y
   predecible que depender del orden de dibujo.

   **Gizmo de ejes globales (2026-08-14, pedido directo del usuario):** fijo en la esquina inferior
   izquierda del visor, siempre visible sin importar zoom/paneo, rota para reflejar la orientación
   actual de la cámara principal. Técnica estándar de gizmo de orientación con un solo canvas/
   renderer: una escena+cámara ortográfica aparte (`three3d.gizmoScene`/`gizmoCamera`, creadas en
   `initThree3D()`, con un `THREE.AxesHelper` + 3 sprites de texto X/Y/Z sobre canvas), renderizada
   en `loopThree3D()` DESPUÉS de la escena principal, dentro de un recorte pequeño
   (`setScissor`/`setViewport`, tamaño fijo en px de pantalla × `pixelRatio`, no en unidades de
   mundo) pegado a `(margin, margin)` — WebGL mide viewport/scissor con Y creciendo hacia ARRIBA
   desde `(0,0)`, así que eso cae directo en la esquina inferior izquierda sin invertir nada.
   **Bug propio + arreglo, mismo día:** la primera versión solo copiaba `gizmoCamera.quaternion` del
   de la cámara principal, dejando la POSICIÓN fija en `(0,0,3)` — con una cámara ortográfica la
   posición y la rotación definen el encuadre juntas, así que en cuanto la rotación copiada apuntaba
   hacia otro lado, el origen (y con él, X e Y) quedaban fuera de cuadro; solo Z sobrevivía visible
   por coincidencia del ángulo. Se verificó proyectando (0,0,0)/(1,0,0)/(0,1,0)/(0,0,1) con
   `Vector3.project(gizmoCamera)` antes y después del arreglo (antes: origen en NDC x=1.15,
   fuera del rango visible ±1; después: origen exacto en (0,0)) — no se confió en la sola inspección
   visual de una captura, que con un plano de fondo negro (ver siguiente bug) era ambigua de por sí.
   El arreglo: reposicionar la cámara del gizmo a lo largo de esa misma dirección (`(0,0,1)` rotado
   por el quaternion de la cámara principal, escalado a una distancia fija) en vez de solo rotar en
   el sitio — el mismo patrón que usa cualquier gizmo de orientación (Blender, etc.). **Segundo bug,
   mismo hallazgo:** el recorte del gizmo se veía como un cuadro negro opaco encimado en la esquina
   (el renderer nunca se creó con `alpha:true`, así que el clear por default es negro opaco); se
   corrigió leyendo el color de limpieza del renderer antes del render del gizmo, poniéndolo
   temporalmente igual al `--canvas` de la escena principal, y restaurándolo después — así el
   recorte se ve integrado con el fondo real en vez de una mancha aparte.

   **Elevación del talud también en 3D (mismo día):** el relleno 3D (`addBox` plano, tope horizontal
   fijo) no reflejaba el ángulo β del talud — el usuario lo pidió explícitamente ("también en el 3D
   crea la elevación por el ángulo"). Se reemplazó por un `THREE.ExtrudeGeometry` (mismo patrón que
   el fuste/el suelo unificado) con un perfil de 4 vértices que sube de `tf+Hr` (en `xBack`) a
   `tf+Hr+slopeRise` (en `Bf+1.0`), usando el MISMO `slopeRun`/`slopeRise` que ya calcula
   `dibujarMuro2D` para el talud del dibujo 2D (`slopeRun = (Bf+1.0)-xBack`,
   `slopeRise = beta>0 ? slopeRun·tan(beta) : 0`) — con `beta=0` el perfil queda idéntico al box
   plano de antes, sin cambiar nada. Verificado leyendo directamente los vértices de la geometría
   (no por inspección visual de una captura, que a cierto ángulo de cámara no dejaba ver el talud con
   claridad): con `beta=0` la altura es igual en ambos extremos; con `beta=20°` la altura en
   `x=Bf+1.0` queda `slopeRun·tan(20°)` por encima de la de `x=xBack`, exacto.

   **Vista por default rotada sobre Y, texto de ayuda reubicado (mismo día):** a pedido del usuario,
   el offset de la posición inicial de cámara (`(0.7, 0.55, 0.9)·dist`, fijado una sola vez por
   sesión vía el flag `actualizarModelo3D.camaraLista`) se rota alrededor del eje Y con
   `Vector3.applyAxisAngle` antes de aplicarse — el offset Y (altura) queda igual porque una
   rotación sobre Y no le afecta, solo cambia la componente X/Z. Se probaron dos valores el mismo
   día: primero 45° (a partir del offset original, no acumulado), y después el usuario pidió -90°
   sobre ESE MISMO offset original — el ángulo actual en el código es **-90°**
   (`-Math.PI/2`). Si se pide otro ajuste, seguir aplicando sobre el offset original
   `(0.7, 0.55, 0.9)`, no acumulando sobre el ángulo ya aplicado. El texto `.viewer-3d-hint`
   ("Arrastra para rotar…") se movió de `bottom:8px` a `top:8px` porque quedaba encimado con el
   gizmo nuevo en la esquina inferior izquierda.
6. **Render de tablas/badges** — incluye `renderResultadosDetalle(r)` (pestaña Resultados → card
   "Memoria de cálculo", agregada 2026-08-10): consolida en un solo lugar TODOS los valores
   intermedios (pesos, presiones laterales, Ka/Kp, momentos, Fh/R, q máx/mín, qult, e) que ya se
   muestran repartidos en las pestañas Cargas/Presiones laterales/Revisiones geotécnicas — mismos
   números, solo reunidos para no ir y venir entre pestañas al armar un reporte. No duplica lógica
   de cálculo, solo lectura del mismo objeto `r` de `calcularTodo()`. También incluye
   `renderResultsBar(r, checks)` (`#resultsBar`, agregada 2026-08-10, estilo GeoCim `.results`):
   franja oscura `position:fixed` al pie de toda la app (mismos colores "acero" que la QAT,
   siempre oscura sin importar el tema), visible sin importar la pestaña activa. Convierte cada
   revisión de FS (capacidad/demanda, "más alto mejor") a una "utilización" demanda/capacidad
   ("más bajo mejor", vía `colorScaleGeo()` con los mismos umbrales de GeoCim: verde ≤0.85, ámbar
   ≤1.0, rojo >1.0) — Volteo/Deslizamiento usan `umbral/FS`, Capacidad de carga usa `q_max/QADM`
   directo, Excentricidad usa `|e|/(B/6)` directo. El veredicto CUMPLE/NO CUMPLE grande de la
   izquierda usa `checks.anyBad` (la lógica real de la app), no solo "¿utilización máxima ≤1.0?",
   para no desalinearse con los badges de cada pestaña en casos borde (q_min<0, excentricidad
   "warn"). El botón "Ver memoria de cálculo" salta a la pestaña Resultados. Reemplaza al viejo
   `#summaryCardViewer` (un banner simple que vivía en el visor 2D/3D, ahora retirado por
   redundante). **Ojo con la altura del layout**: al ser `position:fixed`, `#resultsBar` NO empuja
   contenido — su alto vive en la variable `--results-bar-h` (junto a `--qat-h`/`--tabs-h`/
   `--ribbon-h`/`--header-h` en `:root`) y hay que restarlo donde se calculen alturas contra
   `100vh` (`#viewer-pane{height:calc(100vh - var(--header-h) - var(--results-bar-h))}`) además de
   dársela como `padding-bottom` al `body` — si se agrega otro elemento `position:fixed` al layout,
   revisar estos dos puntos o el contenido queda tapado/recortado detrás de la barra nueva (bug
   real que pasó al agregar `#resultsBar` en esta misma fecha, corregido el mismo día).

   **6b. Fórmula + resaltado al pasar el cursor, directo en el dibujo REAL del muro (2026-08-10,
   iteración final tras 3 versiones el mismo día):** los 5 pesos (`Ws/Wf/Wr/Wtri/Wp`) de la card
   "Peso propio (calculado)" (pestaña Cargas) y "Memoria de cálculo" (pestaña Resultados) —
   `<label data-calc="Ws|Wf|Wr|Wtri|Wp">` — al pasar el cursor sobre el nombre, atenúan TODO el
   `#wall2d` real (`initCalcTooltips()`, llamada una vez desde `init()`; los `<label>` son estáticos
   en el HTML, no hay que rebindear por render). Historia de las 3 versiones probadas el mismo día,
   cada una a pedido explícito del usuario: (1) fórmula siempre visible bajo el campo (texto plano);
   (2) el usuario pidió que solo apareciera al hover + "una ventanita donde se vea gráficamente qué
   está considerando" → ventana flotante `#calcTooltip` con la fórmula + `miniWallSVG()`, un esquema
   mini autocontenido del muro; (3) el usuario preguntó "¿O lo puedes indicar directo en el canvas?"
   y después pidió explícitamente "ya no habras las ventanas, pero los cálculos muéstralos en el
   canvas" → se RETIRÓ `#calcTooltip` y `miniWallSVG()` por completo (HTML/CSS/JS eliminados, no solo
   deshabilitados) y todo (velo + resaltado + texto de fórmula) se dibuja ahora directo sobre el
   `#wall2d` real. **Mecánica final (`showCanvasHighlight(key)`):** un `<rect>` `#wallHighlightVeil`
   semitransparente (`opacity:0.85`, color `--canvas`) cubre todo el viewBox; encima, un
   `<polygon>` `#wallHighlightShape` en las coordenadas EXACTAS del dibujo real (naranja fijo
   `#f5a623`, no ligado al tema, para contrastar siempre) marca la región de esa variable; encima de
   todo, un rótulo `#wallHighlightLabel` (`svgTextWithBg()`, mide el texto ya insertado vía
   `getBBox()` para que el fondo se ajuste al contenido — necesario porque la fórmula de `Wr` puede
   tener varios términos y ser mucho más larga que las demás) con la fórmula completa
   (`formulaWs/Wf/Wr/Wtri/Wp(geo)`, cada factor en la unidad de SU PROPIA categoría vía `fmtCat`),
   fijo cerca del tope del canvas (no pegado a la región, que puede ser angosta o estar en una
   esquina). La clave para que el polígono quede perfectamente alineado con el dibujo real
   (dimensiones, texturas, etc.) es que NO se recalculan escala/offset por separado: `dibujarMuro2D`
   guarda sus propias closures `X`/`Y` (ligadas al scale/offset de ESE render) más la geometría cruda
   en `lastWallTransform` (module-level), y `highlightRegionPoints(key, lastWallTransform)` arma el
   polígono en unidades de modelo (m) que `showCanvasHighlight()` transforma con esas mismas `X`/`Y`.
   Como `dibujarMuro2D` limpia el SVG por completo en cada render
   (`while(svg.firstChild) svg.removeChild(...)`), si el cursor sigue sobre una etiqueta cuando algo
   más dispara un re-render, el resaltado se perdería — por eso `renderAll()` guarda
   `hoveredCalcKey` (module-level, seteado en `mouseenter`/limpiado en `mouseleave`) y lo reaplica
   justo después de `dibujarMuro2D()`. `lastCalcResult` (asignado al inicio de cada `renderAll()`) es
   lo que el handler de hover lee — nada se recalcula al pasar el cursor, solo se lee el último
   resultado ya calculado.

   **Bug propio + arreglo, región de `Wp` (mismo día):** la primera versión del polígono de `Wp`
   iba de `y=0` (fondo de zapata) a `y=Df`, calcado literalmente de la fórmula (`γf·toe·Df`, que no
   resta `tf` — ver aproximación documentada donde se define `Wp` más abajo). Sobre el dibujo REAL
   eso pintaba el resalte naranja encima de la losa misma, que el usuario reportó como confuso
   ("Wp también está dibujando sobre la losa"). Se corrigió `highlightRegionPoints()` para que el
   polígono de `Wp` vaya de `y=tf` (tope de zapata) a `y=Df`, mostrando solo el tramo de suelo que
   visualmente está SOBRE la losa — el usuario aclaró explícitamente "no simplifiques el cálculo de
   las áreas": este ajuste es SOLO del polígono que se dibuja/resalta, la fórmula (`formulaWp`) y el
   valor real de `Wp` en `calcularGeotecnia` siguen usando `toe·Df` completo, sin restar `tf` — no se
   tocó ni se simplificó ningún cálculo, solo la región que se pinta.

   **`A_fuste` desglosada en `formulaWs` (mismo día):** el usuario reportó "estás simplificando el
   área del fuste" — se verificó primero con headless Chrome que NO había ningún bug real: el
   polígono de resaltado de `Ws` coincide pixel-por-pixel con el fuste real dibujado, y su área
   (shoelace) coincide exactamente con `geo.areaStem` y con la fórmula de trapecio `H·(ts+tc)/2` (se
   probó con batter real, ts≠tc, para no quedarse en el caso trivial). El problema real, aclarado
   por el usuario al preguntársele, era otro: el rótulo mostraba `A_fuste` como un número ya resuelto
   (ej. "0.750 m²") sin mostrar de dónde salía. Por eso las funciones `formulaWs/Wf/Wr/Wtri/Wp`
   ahora devuelven un ARREGLO de renglones (antes un solo string) — la mayoría sigue siendo un
   renglón único, pero `formulaWs` devuelve dos: primero `A_fuste = H·(ts+tc)/2 = ... = ...`
   (fórmula cerrada del trapecio, matemáticamente idéntica a como `calcGeometria` la calcula por
   rectángulo+triángulo, solo más legible), y luego `Ws = γc × A_fuste = ... = ...` ya usándola.
   `svgTextWithBg()` (antes de una sola línea) ahora arma un `<tspan>` por renglón con `dy`
   acumulado, igual que `labelBgLines()` de `dibujarMuro2D` pero fuera de ese scope.

   **Fórmulas de presiones laterales (mismo día, extensión del bloque anterior):** el usuario pidió
   extender el mismo mecanismo de hover+resaltado a los 9 resultados de la card "Resultados de
   presiones" (pestaña Cargas) / sección 2 de "Memoria de cálculo" (Resultados): `Ka, Pa, Pa,h, Pq,h,
   Pw,h, Kp, Pp, K_AE, ΔPAE,h` — mismos `<label data-calc="Ka|Pa|Pah|Pqh|Pwh|Kp|Pp|KAE|DeltaPAE">`,
   mismo `CALC_INFO`, mismo `showCanvasHighlight()`. La diferencia real frente a los pesos es que
   `Ka`/`Pa` NO son un producto simple de factores: `Pa` se integra sobre `capasExt` (estratos reales
   + un tramo ficticio del talud si `beta>0` + un tramo ficticio del espesor de zapata si `tf>0`,
   cada uno con su propio `Ka_i` según su φ — ver el bloque de H' más arriba). Para no
   recalcular nada por separado (riesgo de que el texto mostrado diverja del número real, que era
   justo la preocupación del usuario con `Wp`/`Ws`), se modificó el loop de integración de
   `calcularPresiones` para anotar `Ka_i`/`sigmaTop`/`sigmaBot`/`Fi` en cada elemento de `capasExt`
   TAL COMO YA SE CALCULABAN (ninguna fórmula nueva, solo se guardan los valores intermedios que
   antes se descartaban), y se devuelven en `pres.segments` (más `pres.gamma_avg`, también ya
   calculado internamente). `formulaPa()` itera `pres.segments` y arma un renglón por segmento
   (etiquetado "talud"/"zapata"/"estrato" con la MISMA condición que usa `calcularPresiones` para
   armar `capasExt`) más un renglón final `Pa = ΣF`. `formulaKa()` es distinta por método
   (`state.metodo_presion`): Rankine general (con β), Coulomb (con δ/θ), At-rest (`Ks` dado o Jaky
   `1-sin(φ)`), o "N/A" en Fluido equivalente (que no usa `Ka`, usa `EFP` directo — `formulaPa`
   también tiene su propia rama de fórmula cerrada `½·EFP·H'²` para ese método, sin segmentos).
   `Kp`/`Pp`, `K_AE`/`ΔPAE,h` y `Pq,h`/`Pw,h` muestran "0"/"—" con una nota cuando el toggle
   correspondiente (`pasivoEnabled`/`sismo.enabled`/`qsEnabled`/`waterEnabled`) está apagado, en vez
   de una fórmula con ceros que no dice nada. `highlightRegionPoints()` ahora puede devolver VARIOS
   polígonos por clave (antes uno) — necesario porque `Ka`/`Pa`/`Pa,h`/`Pq,h`/`K_AE`/`ΔPAE,h`
   resaltan el relleno completo (rectángulo + el triángulo del talud si existe, dos formas a la
   vez); `Kp`/`Pp` resaltan una cuña esquemática frente a la puntera (ancho proporcional a `Df`,
   no es una medida exacta de nada, solo referencia visual); `Pw,h` resalta una franja delgada
   pegada a la cara posterior desde la zapata hasta `Hw` (recortada visualmente a un máximo
   razonable si `Hw` es muy grande — SOLO el dibujo, el cálculo real sigue usando `Hw` sin recortar).
   Verificado con headless Chrome recorriendo los 4 métodos de presión, activando/desactivando cada
   toggle, y con talud (`beta=10°`, 3 segmentos: talud+estrato+zapata) — todos los números mostrados
   en los renglones de segmento suman EXACTO al `Pa` ya calculado por el motor real.

   **`Wtri` — triángulo de relleno sobre el talud (2026-08-10):** hasta esta fecha `Wr` solo pesaba
   el rectángulo de relleno hasta `H` (zona ④ de la fig. 8.12 de Das); el triángulo de suelo que se
   forma POR ENCIMA de `H` cuando `beta>0` (zona ⑤ del libro) estaba documentado como "mejora
   pendiente, no pedida" — el usuario pidió agregarlo explícitamente. Es un triángulo con base=heel
   (de `xBack` a `Bf`) y altura=`heel·tan(beta)` en el borde `Bf` — la MISMA geometría que ya usaba
   `Hprime` para integrar Pa (el tercer tramo, H3 del libro), ahora también convertida en peso.
   `Wtri = gammaTop · 0.5 · heel² · tan(beta)`, donde `gammaTop = capasPeso[0].gamma` (el mismo
   criterio que ya usa el resto de la app para "el estrato que aflora en la corona": `capasExt` en
   `calcularPresiones`, el dibujo del talud). Solo existe si `beta>0` Y `Hrelleno===H` (el relleno
   realmente llega hasta la corona — si no llega, hay un tramo de fuste expuesto y no hay talud que
   dibujar, ver `alturaRellenoEfectiva`); esta condición vive en `geo.hayTalud`. Su brazo respecto al
   toe NO es el mismo que el de `Wr` (`Bf-heel/2`, centroide de un rectángulo): el triángulo es
   angosto en `xBack` y ancho en `Bf`, así que su centroide cae en `Bf-heel/3` (`brazoTri`, más cerca
   de `Bf`). Se suma a `N` y `Mr` como término propio (no se mezcla con `Wr`, para no ensuciar la
   fórmula ya mostrada de `Wr`). El campo en la UI (`fieldWtriOut`/`fieldWtriRes`) se oculta cuando
   `geom.beta<=0` vía el mismo `show()` de `syncMethodButtons()` que ya oculta `cardSismo`/
   `cardSobrecarga` — la condición de visibilidad usa solo `beta>0` (no `hayTalud`) para no
   parpadear según si el relleno alcanza la corona; con `beta>0` pero relleno corto, el campo queda
   visible mostrando `0 kg/m` en vez de ocultarse, que es más honesto que escamotear el campo.
   Verificado con headless Chrome: `beta=0` → `Wtri=0`/campo oculto; `beta=10°` → `Wtri>0`/campo
   visible, y `N`/`brazoTri` recompuestos a mano desde `geo.*` coinciden exactamente con lo que
   calcula la app.

   **Peso específico (γ) agregado a la etiqueta de cada estrato en el dibujo 2D (2026-08-10):** la
   etiqueta multilínea de cada estrato (ver bloque 5 más abajo, `labelBgLines`) ya mostraba
   nombre/c/φ; el usuario pidió agregar también γ — ahora son 4 líneas: nombre, `γ=`, `c=`, `φ=`.

   **7. Render general** (`renderAll`/`renderAllFull`) — único punto
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
- **Sistema de unidades por categoría (2026-08-10, reemplaza el toggle binario kgf/SI anterior)**:
  `state` SIEMPRE guarda la unidad base técnica internamente (kg, kg/m, kg/m², kg/m³, kg·m/m,
  kg/cm², metros — el sistema ya verificado a mano contra Python); el motor de cálculo nunca ve
  otra unidad. El usuario elige la unidad de despliegue **por categoría**, no un sistema global:
  `state.unidades = { longitud, pesoVol, fuerza, presion, momento, material }`, editable en un
  card "Unidades" de la pestaña Proyecto (ya no hay botón en la QAT) — se guarda con el proyecto
  (`.gcon`), no en localStorage como el tema. `UNIT_DEFS` (junto a `defaultState()`) define, por
  categoría, las opciones disponibles y su factor "1 unidad-base → unidad-destino" (incluye
  técnico ampliado, SI e imperial, ej. longitud: m/cm/ft/in; presión: kg/m²/kg/cm²/Tn/m²/kPa/MPa/
  psf/ksf). `unitFactor(cat)`/`unitLabel(cat)`/`toDisplay(cat,val)`/`toBase(cat,val)` son los
  helpers genéricos; `fmtFuerza`/`fmtMomento`/`fmtPresionSuelo`/`fmtLong` son envolturas de
  `fmtCat(cat,val,dec)` con el nombre histórico (para no tocar cada punto de llamada). Los campos
  editables marcados con `data-unit="<categoria>"` (ya no `F|Q|S`) se convierten en la frontera
  display↔state: `syncInputsFromState()` usa `toDisplay()` para mostrar, y el handler de
  `bindAll()` usa `toBase()` para volver a la unidad base antes de `setPath()`; cambiar un
  `<select data-bind="unidades.*">` dispara además un `syncInputsFromState()` inmediato (no solo
  el `scheduleRecalc()` debounced) para refrescar todos los campos afectados al toque. Las
  etiquetas `<span class="unit-tag" data-cat="<categoria>">` se sincronizan con `syncUnitLabels()`.
  Los ocho rótulos de cota del dibujo 2D (H, Df, tf, toe, ts, heel, Bf, tc) y el campo calculado
  `in-heel` también usan la categoría "longitud". **`suelo.QADM` cambió de guardarse en tf/m² a
  kgf/m²** (unificado dentro de la categoría "presion" junto con `c_r`/`c_f`/`qs`) — el valor por
  default pasó de `10` a `10000` para mostrar exactamente lo mismo (10 t/m²) con el default de esa
  categoría. Campos de ángulo/adimensionales (φ, Ka, FS, kh…) no llevan `data-unit`. Excepciones
  que quedan fuera a propósito: `cargas.Pu_torre/Mu_torre/Vu_torre` (el prompt original ya los
  definió en kN/kN·m) y `materiales.recub` (sigue fijo en cm, campo reservado sin uso todavía en
  el motor de cálculo).

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
