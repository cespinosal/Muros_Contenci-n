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

   **Se quitaron `Pa` y `Pp` de la card "Resultados de presiones" (2026-08-15):** el usuario pidió
   quitarlos (los describió como de la pestaña "Suelo y materiales", pero al preguntarle confirmó
   que se refería a esta card, en "Presiones laterales") — quedan redundantes con el hover+resaltado
   ya construido (`data-calc="Pa"`/`"Pp"` siguen existiendo como claves de `CALC_INFO`, solo ya no
   hay una etiqueta `<label data-calc="Pa">`/`"Pp"` en ESTA card que los dispare — `Pa,h`/`Kp` sí se
   quedan, son los que de verdad se usan en las revisiones geotécnicas). `renderPresionesOutputs()`
   ya no escribe en `out-Pa`/`out-Pp` (esos ids ya no existen en el DOM). Se agregó un `<p class="hint">`
   aclarando que `Ka` usa el primer estrato del relleno y `Kp` usa γp/φp/cp del suelo de fundación
   (capturadas en la OTRA pestaña, "Suelo y materiales" — no son visibles en "Presiones laterales").
   **Nota:** la sección 2 de "Memoria de cálculo" (pestaña Resultados) SÍ sigue mostrando `Pa`/`Pp`
   completos — esa card es un registro exhaustivo a propósito, no se tocó.

   **[Extensión, mismo día] Pa/Pp en el DIBUJO 2D también se ocultan fuera de "Presiones laterales":**
   con una captura marcada a mano, el usuario aclaró que además de la card (arreglada arriba) las
   etiquetas `Pa=...`/`Pp=...` que se dibujan directo en el canvas (triángulo esquemático + rótulo,
   dentro del bloque `// ---- Fuerzas ----` de `dibujarMuro2D`) deben verse SOLO cuando la pestaña
   activa del ribbon es "Presiones laterales" — en Geometría del muro/Suelo y materiales/Cargas no.
   El resto de fuerzas dibujadas ahí (ΔPAE sísmico, cerco perimetral, sobrecarga qs) NO se pidieron
   con esta regla, siguen dependiendo solo del toggle global `uiState.showForces` como antes.
   Implementado leyendo `document.querySelector(".ribbon-tab.active")?.dataset.tab` directo del DOM
   dentro de `dibujarMuro2D` (variable `mostrarPaPp`, sin duplicar el estado de la pestaña activa en
   `state`/`uiState`) y envolviendo los dos bloques de dibujo (triángulo+rótulo de `Pa`, de `Pp`) con
   esa condición. Como el visor 2D vive FUERA del sistema de `.tab-panel` (es un panel persistente a
   la izquierda, no se oculta/muestra por pestaña), cambiar de pestaña por sí solo no disparaba un
   redibujo — se agregó `renderAll()` al final del handler de click de `.ribbon-tab` para que
   Pa/Pp aparezcan/desaparezcan de inmediato al cambiar de pestaña. Verificado con headless Chrome
   recorriendo las 6 pestañas: `Pa`/`Pp` solo aparecen en "presiones", ausentes en las otras 5.

   **Anotaciones del canvas condicionadas a la pestaña "Suelo y materiales" (2026-08-15):** el
   usuario pidió (tras una ronda de aclaración, porque su primera descripción no coincidía con lo
   que había en el código — resultó ser sobre el CANVAS, no sobre campos de formulario): (1) que la
   etiqueta de propiedades de cada estrato (nombre+γ+c+φ) YA NO se vea en pestañas que no sean
   "Suelo y materiales" (antes se mostraba en todas), (2) que las cotas queden desactivadas mientras
   se está en "Suelo y materiales" (compiten visualmente con las etiquetas de propiedades), y (3) que
   se agregue una etiqueta nueva con las propiedades del suelo de fundación (γf/cf/φf) en el canvas,
   mismo patrón que la del estrato, también solo visible ahí. La detección de pestaña activa
   (`tabActiva`, `mostrarSuelo = tabActiva==="suelo"`) se MOVIÓ al inicio de `dibujarMuro2D` (antes
   vivía más abajo, solo para `mostrarPaPp`) para poder reutilizarla también en el bloque del
   relleno estratificado, que se dibuja ANTES del bloque de fuerzas. La etiqueta de suelo de
   fundación usa el mismo patrón `labelBgLines()` que la del estrato, ubicada en el mismo carril
   horizontal donde ya vive "NTN" (a media profundidad de `Df`, dentro del polígono `soilPts`).

   **Corrección (mismo día, 2026-08-15):** la primera versión apagaba las cotas con un override
   silencioso en el dibujo (`&& !mostrarSuelo` agregado a la condición del bloque `// ---- Cotas
   ----`, mismo patrón que `mostrarPaPp` para Pa/Pp), dejando el checkbox `chkCotas` visualmente
   marcado aunque sin efecto. El usuario mandó una captura señalando el checkbox realmente
   desmarcado y pidió "solo desactiva la opción de cotas por default en suelo y materiales" — es
   decir, el checkbox mismo debe reflejar el estado. Se corrigió moviendo la lógica a
   `actualizarVisorSegunPestana(tabId)`: al entrar a `tabId==="suelo"` se hace
   `uiState.showCotas=false` y `document.getElementById("chkCotas").checked=false` de verdad (un
   default por visita, no un bloqueo — el usuario puede volver a marcarlo a mano mientras sigue en
   esa pestaña, y se resetea a apagado cada vez que se vuelve a entrar). El bloque `// ---- Cotas
   ----` de `dibujarMuro2D` se revirtió a la condición simple `if(uiState.showCotas){`, sin
   referencia a `mostrarSuelo`. Verificado con headless Chrome (5 escenarios: entrar a "suelo"
   desmarca y oculta cotas; marcar a mano mientras se está ahí las vuelve a mostrar; salir y
   reentrar resetea a desmarcado; otras pestañas no fuerzan el estado).

   Verificado con headless Chrome en 4 pestañas (geometria/suelo/cargas/presiones): las 3
   anotaciones se comportan exactamente como se pidió, con captura de pantalla confirmando que no
   hay traslapes visuales en "Suelo y materiales".

   **Pestaña Proyecto sin visor, estilo GeoCim (2026-08-15):** el usuario pidió "quita el canvas de
   la pestaña Proyecto... para que se vea como GeoCim, solo colocar los datos del proyecto". El visor
   (`#viewer-pane`) vive FUERA del sistema de `.tab-panel` — es un panel persistente a la izquierda,
   `#main-layout{display:flex}` con `#viewer-pane{flex:1 1 auto}` + `#content{width:420px}` como
   columna fija a la derecha — así que "ocultarlo en una pestaña" no es tan simple como un
   `display:none` en un `.tab-panel` más. Se agregó una clase `#main-layout.sin-visor` (CSS: oculta
   `#viewer-pane` por completo, y `#content` pasa de columna fija de 420px a formulario centrado de
   hasta 640px, ya que deja de compartir la fila con el visor) que se activa/desactiva vía
   `actualizarVisorSegunPestana(tabId)` — llamada tanto en el handler de click de `.ribbon-tab` (con
   el `tabId` del botón) como una vez en `init()` con `"proyecto"` fijo (porque esa pestaña ya está
   activa por default al cargar, antes de cualquier clic — sin esa llamada iba a faltar el estilo en
   la primera carga). Verificado con headless Chrome + captura de pantalla: el visor está oculto al
   cargar (pestaña Proyecto por default), reaparece al cambiar a otra pestaña, y se vuelve a ocultar
   al regresar a Proyecto.

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

**Bug real: preset de sobrecarga no convertía unidades al escribir en el campo (2026-08-16):** el
usuario reportó "revisa qs aplicada" con una captura mostrando el campo "Sobrecarga qs [kPa]" con el
valor `250` pero el rótulo del canvas mostrando `qs=2.45 kPa` — un factor ~102x de diferencia (el
mismo factor de conversión kgf/m²↔kPa, 0.00980665). Verificado con headless Chrome antes de tocar
nada: efectivamente reproducible (unidad de presión = kPa, preset "Ligera" seleccionado → el campo
`#in-qs` mostraba literalmente `250`, no `2.45`). Causa raíz: el handler de `qsPreset` (Bloque 17,
`QS_PRESETS = {ligera:250, normal:500, pesada:1000}`, SIEMPRE en la unidad base kg/m²) hacía
`document.getElementById("in-qs").value = QS_PRESETS[v]` — asignando el número CRUDO de la unidad
base directo al campo, sin pasar por `toDisplay("presion", ...)` como sí hacen todos los demás
caminos (`syncInputsFromState()`, el handler genérico de `bindAll()`). El bug es invisible mientras
la unidad de presión activa es kg/m² (factor 1, coincide por casualidad) y solo se nota al cambiar a
cualquier otra unidad (kPa, Tn/m², psf...). **Importante: el MOTOR DE CÁLCULO nunca estuvo mal** —
`state.cargas.qs = QS_PRESETS[v]` sí queda correcto (250 kg/m² real), y `Pq,h` se calculó siempre
bien (verificado: 279.17 kg/m, coincide con Ka·qs·H' a mano) — el bug era puramente de DISPLAY en ese
campo, pero visualmente indistinguible de un error real de cálculo desde la captura del usuario.
Corregido envolviendo la asignación en `toDisplay("presion", QS_PRESETS[v])` (mismo redondeo a 5
decimales que usa `syncInputsFromState()`). Se grepeó el resto del archivo por el mismo patrón
(asignación directa de un número crudo a un input `data-unit` sin pasar por `toDisplay`) y no
apareció en ningún otro lugar — este era el único caso. **Why:** cualquier valor que se escriba
programáticamente en un campo marcado `data-unit="<categoria>"` DEBE pasar por `toDisplay()`, nunca
asignarse crudo, aunque la fuente (como un preset) ya esté en la unidad base — el campo siempre
refleja la unidad de DESISPLAY activa, no la base. **How to apply:** si se agrega otro preset/atajo
similar en el futuro (o un dropdown que precargue valores) que escriba `.value` de un campo con
`data-unit`, envolver el valor con `toDisplay(categoria, valor)` — no asumir que "ya está en la
unidad correcta" solo porque coincide con el valor guardado en `state`.

**Diagramas de presión lateral separados del esquema, estilo Showcrete (2026-08-16):** el usuario
mandó una captura de otro software (Showcrete) donde los diagramas de empuje activo/pasivo/
sobrecarga se ven como figuras APARTE junto al esquema del muro (con hatch, valores de presión en
los extremos, y una flecha de la resultante con su fuerza R y altura de aplicación H), y pidió
replicar ese estilo en la pestaña "Presiones laterales". Antes, `dibujarMuro2D` dibujaba triángulos
esquemáticos de ancho fijo en PÍXELES (`triW=55`), pegados directamente a la cara del muro/puntera —
sin relación real con la escala de presión ni con la forma real del perfil (que puede tener
escalones si el relleno está estratificado, ya que Ka cambia por capa).

Se reemplazaron esos triángulos por diagramas reales, dibujados a los lados del esquema:
- **Ventana de vista ampliada sin tocar la geometría real:** se separaron `modelMinX/modelMaxX`
  (bounds de la geometría real — suelo, muro, relleno; SIN CAMBIOS, se siguen usando igual en todo
  el resto de la función) de un nuevo par `viewMinX/viewMaxX` (bounds de la VENTANA de dibujo, usados
  solo para calcular `scale`/`offX`/`modelW` y como base del closure `X()`). Cuando
  `mostrarPresiones` (pestaña "Presiones laterales" activa) es true, `viewMaxX` se amplía
  `diagGap+diagW+diagLabelMargin` (0.3+1.7+0.55 m) a la derecha para el diagrama activo/sobrecarga, y
  si además `suelo.pasivoEnabled` es true, `viewMinX` se amplía lo mismo a la izquierda para el
  pasivo. Fuera de esa pestaña, `viewMinX===modelMinX` y `viewMaxX===modelMaxX` (comportamiento
  idéntico al de antes, cero cambio visual en las demás pestañas). `modelMaxY` también gana un cuarto
  candidato `r.pres.Hprime + 0.3` en el `Math.max(...)` (antes solo consideraba `Hr`/cerco/`H`), por
  si el plano de integración H' (que puede incluir el talud + espesor de zapata) fuera más alto que
  esos tres.
- **`pressureDiagram(x0, dir, piezas, scale, fillColor, strokeColor)`** (helper local de
  `dibujarMuro2D`, junto a `poly`/`line`/`labelBg`): dibuja cada "pieza" `{yTop,yBot,sigTop,sigBot}`
  (coordenadas REALES del modelo) como su PROPIO trapecio independiente — no un solo polígono
  continuo — para que un salto de Ka entre estratos se vea como el escalón real que es, en vez de
  conectar diagonalmente valores que no son continuos. `dir` es +1 (crece hacia la derecha, empuje
  activo/sobrecarga) o -1 (crece hacia la izquierda, empuje pasivo).
- **`pressureResultant(x0, dir, piezas, scale, arm, forceVal, forceLabel, color)`**: interpola la
  presión exactamente en `arm` (altura real de la resultante) dentro del tramo que la contiene, para
  que la flecha (línea + triángulo dibujado a mano con `poly`, sin `<marker>`) toque el borde del
  diagrama justo ahí, con etiquetas `forceLabel=valor` y `H=brazo`.
- **`pressureExtremeLabels(x0, dir, piezas, scale, color)`**: etiqueta la presión (no la fuerza) en
  el extremo superior e inferior del diagrama completo, en la unidad de presión activa del usuario
  (`fmtCat("presion",...)`).
- **Diagrama activo (+ sobrecarga superpuesta), lado derecho:** reconstruye `piezas` a partir de
  `r.pres.segments` (el mismo `capasExt` ya anotado con `sigmaTop`/`sigmaBot` por `calcularPresiones`
  — NUNCA recalculado aquí, mismo patrón que el hover de Ws/Wp) recomputando solo `yTop`/`yBot` por
  acumulación de `s.h` desde `Hprime` hacia abajo. Si el método es Fluido equivalente (`segments`
  viene `null`), usa un único tramo sintético `{yTop:Hprime, yBot:0, sigTop:0, sigBot:EFP·Hprime}`
  (el perfil estándar de ese método). Si `cargas.qsEnabled`, se dibuja un rectángulo de sobrecarga
  (presión uniforme `Ka·qs`, mismo cálculo que ya usaba `formulaPqh`) EN LA MISMA escala/origen que
  el activo, superpuesto (como en la imagen de referencia) — ambos comparten `scaleR = diagW /
  maxSigmaR` donde `maxSigmaR` es el máximo de presión entre ambos, para que las dos figuras sean
  comparables visualmente.
- **Diagrama pasivo, lado izquierdo:** un solo trapecio (ecuación 8.9, Das): `sigTop = 2·cp·√Kp`
  (término de cohesión, constante) en `y=Df` y `sigBot = Kp·γp·Df + 2·cp·√Kp` en `y=0` — antes no
  existía un brazo (`arm`) real para Pp, el hover-highlight seguía usando un rectángulo simplificado.
  Se agregó **`pres.armPasivo`** al `return` de `calcularPresiones` (ponderando el brazo triangular
  del término friccionante, `Df/3`, y el rectangular del término de cohesión, `Df/2`, por su fuerza
  respectiva) — expuesto para que el diagrama lea un solo número ya calculado, no lo recalcule.
- Los triángulos esquemáticos VIEJOS de Pa/Pp (pegados a la cara del muro) se eliminaron por
  completo, reemplazados por los diagramas de arriba. El incremento sísmico ΔPAE (triángulo amarillo
  pegado a la cara del muro) y la franja de sobrecarga sobre el relleno (visualización de DÓNDE se
  aplica qs, distinta del diagrama de presión lateral) **NO se tocaron** — quedan fuera de esta regla
  a propósito (el usuario no las mencionó, y la imagen de referencia tampoco mostraba sismo).

**[Corrección + extensión, mismo día] Sobrecarga e hidrostática en carriles propios + flechas
invertidas hacia el muro:** en un mensaje de seguimiento el usuario pidió dos cosas más sobre estos
diagramas: (1) "también separa el gráfico de la sobrecarga y la presión hidrostática" — la
sobrecarga vivía superpuesta al activo en el mismo carril (`x0R` compartido), y la hidrostática NO
tenía ningún diagrama todavía (solo el campo numérico `Pw,h` y el resaltado de hover); y (2), en un
mensaje aparte, "Pa,h y Pp invierte las flechas" — la primera versión de `pressureResultant` ponía la
punta de flecha en `xArrow` (el borde del diagrama, el extremo MÁS ALEJADO del muro), cuando
físicamente la resultante empuja HACIA el muro.

- **Carriles independientes a la derecha:** `laneX0(i) = modelMaxX + (i+1)·diagGap + i·diagW` da el
  origen x del carril i-ésimo; el orden es activo (siempre) → sobrecarga (si `qsEnabled`) →
  hidrostática (si `waterEnabled` y `Hw>0`), cada uno con su PROPIA escala (`diagW / su propio
  máximo de presión`, ya no una escala compartida entre activo y sobrecarga). `lanesRight` (calculado
  arriba, junto a `viewMinX/viewMaxX`) cuenta cuántos carriles hacen falta para reservar el ancho de
  vista correcto — mismo conteo/orden que el bloque de dibujo, si se agrega un carril nuevo hay que
  actualizar los dos lados a la vez.
- **Diagrama hidrostático (nuevo):** triángulo simple `sigTop=0` en `y=Hw` (superficie del agua) a
  `sigBot=γw·Hw` en `y=0` (base), brazo `Hw/3` — mismo perfil hidrostático estándar que ya usaba la
  fórmula `Pw,h = ½·γw·Hw²` (sin cambiar esa fórmula, solo se le agregó representación visual).
- **Títulos de los 3 carriles derechos alineados a una sola altura (`titleYRight`):** un intento
  intermedio (posicionar cada título en `su propio yTop + 0.25`) hacía que el título de un carril más
  bajo (p.ej. "Hidrostática", limitado por `Hw` en vez de `Hprime`) quedara casi a la misma altura
  que la etiqueta de la resultante del carril VECINO (`arm` suele caer cerca de la mitad de `Hp`) y
  se encimaban — confirmado visualmente con una captura antes de corregirlo. Se calcula una sola
  `titleYRight = max(Hp, Hw si aplica) + 0.3` y los 3 títulos ("Empuje activo"/"Sobrecarga"/
  "Hidrostática") la comparten, quedando siempre en una fila limpia por encima de cualquier etiqueta
  de resultante (que vive más abajo, a la altura `arm < Hp`). `diagGap` también subió de 0.3 a 0.45 m
  (más aire entre carriles) — ayuda, pero el fix real fue igualar las alturas de título, no el ancho
  del hueco (verificado: con solo más `diagGap` seguía habiendo choque, con `titleYRight` compartido
  se resolvió del todo).
- **Flechas de la resultante corregidas (`pressureResultant`):** la punta del triángulo (dibujado a
  mano con `poly`, sin `<marker>`) se movió de `xArrow` (borde del diagrama) a `x0` (eje/origen del
  carril, el lado más cercano al muro) — apunta HACIA el muro para los 4 empujes (activo, sobrecarga,
  hidrostática con `dir=+1`; pasivo con `dir=-1`), no hacia afuera del diagrama. Este fix vive en el
  helper compartido, así que corrige los 4 empujes a la vez (no hubo que tocar cada llamada).
- Verificado con headless Chrome: 4 carriles simultáneos (activo + sobrecarga + hidrostática +
  pasivo) con nivel freático y sobrecarga activados a la vez, cada uno con su título/valores/flecha
  propios, sin overlaps ni errores de JS; lectura geométrica de los 4 triángulos de flecha
  (`<polygon>` de 3 puntos) confirmando que el vértice-punta queda siempre del lado del muro (x menor
  para los carriles de la derecha, x mayor para el pasivo a la izquierda).

**[Extensión, mismo día] "Que los diagramas no se encimen entre sí, ni con letreros":** se escribió
un script de verificación reusable (headless Chrome, `getBoundingClientRect()` de cada `<rect>` de
fondo de `labelBg`/`labelBgLines` dentro de `#wall2d`, comparado por pares) para detectar
solapamientos REALES en pantalla — no `getBBox()`, que ignora el `transform="rotate(...)"` de las
cotas verticales y da falsos positivos. Con los 4 carriles + torre + cerco + sismo + talud activos a
la vez encontró 3 problemas reales:
1. **Etiqueta de resultante en carriles uniformes (rectángulo, p.ej. sobrecarga) se salía de su
   carril:** `pressureResultant` anclaba la etiqueta en `xArrow` (el borde del diagrama), que para un
   diagrama de presión CONSTANTE está siempre en el borde LEJANO del carril, pegado al hueco con el
   vecino. Se cambió el ancla a `x0` (el eje/origen del carril, el mismo punto donde ahora apunta la
   flecha) — el texto crece HACIA DENTRO del propio carril, nunca cruza el hueco. También se bajó la
   fuerza de la etiqueta de 2 a 0 decimales (`fmtFuerza(forceVal,0)`) porque un valor largo como
   "3366.75 kg/m" todavía alcanzaba a rebasar un carril angosto.
2. **Título del carril vs. su propia etiqueta de presión "0" en el tope:** el margen de
   `titleYRight`/el del pasivo (antes `+0.3`/`+0.25` m) no bastaba cuando la escala se comprime por
   tener varios carriles a la vez — se subió a `+0.5` en ambos.
3. **Cotas compitiendo con las etiquetas de los diagramas:** con los diagramas ocupando espacio a los
   lados, el muro se dibuja más chico en "Presiones laterales" y las cotas (H, Df, tf, toe, ts, heel,
   Bf, tc) quedaban más apretadas que antes. Se extendió `actualizarVisorSegunPestana` (mismo patrón
   que ya apaga cotas por default en "Suelo y materiales") para que también las apague por default al
   entrar a "Presiones laterales" — el usuario puede volver a marcarlas a mano si las necesita ahí.

**Nota de escala:** aumentar `diagGap` (el espacio entre carriles) en METROS no es un lever confiable
por sí solo — como `diagGap` también entra en el cálculo de `viewMaxX`, agrandarlo ensancha
`modelW` y ENCOGE la escala (px/m), así que el "aire" ganado en metros se pierde parcialmente en
píxeles (confirmado probando `diagGap=0.75`: no mejoró el cruce entre carriles y hasta reintrodujo el
choque título/etiqueta-0 que ya se había resuelto). El fix real fue estructural (anclar en `x0`,
menos decimales, más margen en el eje que sí importaba) — `diagGap` se dejó en 0.45 (subido desde el
0.3 original, pero sin seguir subiéndolo). Verificado de nuevo con el mismo script: de 14 overlaps
reales a 5, y los 5 restantes son de ≤2px (o un roce de 2px en un solo caso) — el ancho de un trazo
de anti-aliasing, no perceptible visualmente (confirmado también con captura de pantalla). **Why:**
mismo hilo de "cómo se ve" que ya motivó separar los diagramas del esquema — un diagrama correcto que
se lee mal (texto encimado) no cumple el objetivo de transparencia. **How to apply:** el script de
verificación (getBoundingClientRect + comparación por pares, filtrando por `<rect>` dentro de
`#wall2d`) es reusable para cualquier ajuste futuro de layout del canvas — no juzgar overlaps a ojo
desde una sola captura, medir.

**[Segunda extensión, mismo día] Más separación entre carriles + flechas invisibles:** el usuario
pidió, en un mensaje de dos líneas, "separalos más" (más aire entre los 4 diagramas) y "también las
flechas no se ven".

- **Flechas invisibles — bug real de orden de dibujo:** al anclar la etiqueta de la resultante en
  `x0` (fix anterior, mismo punto donde ahora también apunta la flecha), `pressureResultant` seguía
  dibujando la línea+triángulo de la flecha ANTES que `labelBg` — en SVG el orden del DOM es el orden
  de pintado, así que el fondo opaco (`opacity:0.82`) de la etiqueta terminaba tapando la flecha por
  completo, al estar ambas casi en el mismo punto. Se invirtió el orden (etiquetas primero, flecha al
  final) para que la flecha SIEMPRE quede encima, sin importar cuánto se acerque la etiqueta.
  Verificado con un segundo script (además de medir overlaps, recorre los nodos del SVG en orden de
  DOM y confirma que ningún `<rect>` posterior a cada flecha contiene su centro) — los 4 triángulos de
  flecha dieron `covered=false`.
- **Más separación — el fix real fue estructural, no solo "más diagGap":** subir `diagGap` solo (ya
  documentado arriba: no es un lever confiable por el encogimiento de escala) había hecho el problema
  peor en el intento anterior. Esta vez se subió `diagGap` de 0.45 a 0.8 m Y se bajó `diagW` de 1.7 a
  1.5 m (compensando parte del ancho ganado por el hueco, así el total no crece tan agresivo) Y se
  subieron a la vez los márgenes que dependen de esa misma escala más chica (`titleYRight` de +0.5 a
  +0.7; el margen del título "Empuje pasivo" de +0.5 a +0.7) — subir SOLO el hueco sin subir también
  los márgenes de altura es lo que había fallado la vez anterior.
- Verificado de nuevo con el script de overlaps: 5 coincidencias restantes, todas de ≤2.7px (las
  mismas líneas-hermanas de una etiqueta de 2 renglones, tocándose por diseño) — cero cruces reales
  entre carriles. Confirmado también con captura de pantalla: los 4 diagramas se ven claramente
  separados y las 4 flechas (triángulos de color en el borde de cada carril) son visibles.
- **Why:** mismo patrón de esta sesión — un diagrama sin su flecha visible no comunica dónde actúa la
  resultante, que era justamente el objetivo de todo este bloque de trabajo. **How to apply:** en
  `pressureResultant`, cualquier elemento nuevo que se agregue anclado cerca de `(x0, arm)` debe
  dibujarse ANTES de la flecha (línea+triángulo), no después, para no repetir el bug de tapado.

**[Tercera extensión, mismo día] "Pero separa más los gráficos":** el usuario pidió, tras ver el
resultado del ajuste anterior (`diagW=1.5`/`diagGap=0.8`), todavía más aire entre los 4 diagramas.
Mismo patrón que la vez pasada (subir el hueco solo no alcanza, hay que subir junto con él el ancho
del diagrama hacia abajo y los márgenes de altura dependientes de la escala): se bajó `diagW` de 1.5
a 1.3 m, se subió `diagGap` de 0.8 a 1.3 m, y los márgenes de título (`titleYRight`, título de
"Empuje pasivo") subieron de +0.7 a +0.9. Verificado con el mismo script de overlaps + visibilidad de
flechas: 0 cruces reales entre carriles, las 4 flechas siguen `covered=false`, confirmado con captura
de pantalla que los 4 diagramas quedan claramente más separados que en el intento anterior. **How to
apply:** si se pide aún más separación en el futuro, seguir el mismo patrón de 3 parámetros a la vez
(`diagW` abajo, `diagGap` arriba, márgenes de título arriba) — nunca subir `diagGap` solo.

**Flecha de carga de la pestaña Cargas (mismo día):** el usuario pidió, en el mismo mensaje, "en
cargas coloca una flecha en donde se está aplicando la carga y su valor". De las 3 cargas de esa
pestaña (sobrecarga qs, cargas de la torre, cerco perimetral), qs y cerco YA tenían representación
visual (franja sobre el relleno y línea vertical sobre la corona, respectivamente) — la de la torre
(`cargas.Pu_torre`/`dist_torre`) era la única sin ningún dibujo. Se agregó una flecha vertical
descendente en `x = xBack + dist_torre` (misma referencia horizontal que ya usa la condición "Torre
cercana al muro" de `calcularGeotecnia`), apoyada sobre el terreno en ese punto (sigue el talud si
`beta>0`, vía `backTopY + (xTorre-xBack)·tan(beta)`), con la etiqueta `Pu_torre=valor kN` — dibujada
a mano con `line`+`poly` (mismo patrón que la punta de flecha de `pressureResultant`, sin
`<marker>`). Solo visible cuando `cargas.torreEnabled` Y la pestaña activa es "Cargas" (mismo patrón
de "solo en su pestaña" que Pa/Pp en Presiones laterales) — el valor sigue sin integrarse al cálculo
geotécnico (nota ya existente en el formulario de esa pestaña, sin cambios).

Verificado con headless Chrome (sin errores de JS en ninguna prueba): diagramas activo/pasivo/
sobrecarga con valores y brazos correctos en Presiones laterales; el pasivo desaparece limpiamente
al desactivar `chkPasivo` sin romper el resto; el activo sigue funcionando con el método Fluido
equivalente (rama sin `segments`); la flecha de `Pu_torre` aparece solo en la pestaña Cargas con el
valor correcto y NO aparece "Empuje activo" ahí; capturas de pantalla confirmando que no hay
traslapes ni elementos fuera del lienzo. **Why:** mismo hilo de esta sesión — transparencia total de
"cómo se llegó a cada número" (ver el bloque de hover Ws/Wp/Pa más arriba), llevado ahora también a
la representación permanente (no solo al pasar el cursor) de las presiones laterales. **How to
apply:** si se agrega un diagrama de presión nuevo (p.ej. ΔPAE si el usuario lo pide después),
reusar `pressureDiagram`/`pressureResultant`/`pressureExtremeLabels` con las piezas ya calculadas por
el motor — no inventar una forma de perfil que el motor no exponga.

**Tarjeta "Factores de seguridad" en Proyecto + capacidad de carga conectada a un chequeo real
(2026-08-16):** el usuario pidió "en la pestaña proyectos crea un nuevo apartado para que el usuario
pueda ingresar todos los factores de seguridad". `state.FS = {volteo_estatico, volteo_sismico,
desliz_estatico, desliz_sismico, capacidad_carga, eccentricidad}` YA EXISTÍA en `defaultState()`
desde antes (usado en vivo por `renderRevisiones`/`renderResultadosDetalle`/`renderResultsBar` para
volteo y deslizamiento), pero no tenía NINGÚN formulario para editarlo — eran constantes de facto.
Al investigar antes de construir el formulario se encontró que **`FS.capacidad_carga` (3.0) estaba
completamente MUERTO**: `geo.FS_carga` (=qult/q_max) se calculaba y se mostraba (`out-FScarga`/
`res-FScarga`), pero el badge CUMPLE/NO CUMPLE de capacidad de carga solo comparaba `q_max ≤ QADM`,
nunca `FS_carga` contra `FS.capacidad_carga` — el hint de la UI incluso decía literalmente "qult es
informativo, no de diseño". Se le presentó esto al usuario vía pregunta de alcance (3 opciones: solo
exponer los 4 activos / exponer los 5 pero dejar capacidad de carga decorativo / exponer los 5 y
conectar capacidad de carga a un chequeo real) y confirmó la tercera.

- **Formulario nuevo:** tarjeta "Factores de seguridad" en la pestaña Proyecto (después de
  "Unidades"), 5 campos numéricos `data-bind="FS.*"` (sin `data-unit`, son adimensionales) —
  `eccentricidad` ("B/6", un método no un número) se dejó FUERA a propósito, sigue hardcoded en
  `eCategoria` (`calcularGeotecnia`), no se expuso como editable.
- **Capacidad de carga ahora exige DOS condiciones** (antes solo una): `okCarga = (q_max≤QADM &&
  q_min≥0) && (FS_carga≥FS.capacidad_carga)` — se actualizó en las 3 rutinas que lo calculaban por
  separado (no había una sola fuente): `renderRevisiones` (el badge real, única fuente de verdad para
  `checks.okCarga`, que ya se propaga a `renderSummaryCards`/`badgeRevisiones` sin tocarlas), el
  arreglo `rows` de `renderResultadosDetalle` (se agregó una fila nueva "FS Capacidad de carga
  (qult/q máx)", paralela a las de volteo/deslizamiento — las filas de "q máx"/"q mín" contra QADM se
  dejaron intactas, son un criterio distinto y complementario) y el arreglo `items` de
  `renderResultsBar` (la "utilización" ahora es `Math.max(q_max/QADM, FS.capacidad_carga/FS_carga)`
  — el PEOR de los dos criterios, para que la barra se ponga roja/ámbar si cualquiera de los dos
  falla, no solo QADM). Se corrigió el texto ya desactualizado "qult es informativo, no de diseño" en
  la tarjeta de Capacidad portante (pestaña Revisiones geotécnicas).
- **Cambio de comportamiento real y visible, no solo cosmético:** con los valores por default del
  proyecto de ejemplo (QADM=10 Tn/m², q_max=8.55 Tn/m², `FS_carga`=2.55), la revisión de capacidad de
  carga ANTES CUMPLÍA (solo miraba QADM, y 8.55≤10) y DESPUÉS de este cambio **NO CUMPLE** (2.55 < el
  `FS.capacidad_carga` por default de 3.0) — confirmado con headless Chrome antes de dar el trabajo
  por terminado. Cualquier proyecto/plantilla guardada previamente que "cumplía" capacidad de carga
  puede empezar a mostrar NO CUMPLE al abrirse con esta versión, si su `FS_carga` real está por debajo
  de 3.0 — es el comportamiento esperado (la revisión ahora es más completa/estricta), pero vale la
  pena que el usuario lo sepa de antemano si compara contra memorias de cálculo ya entregadas con la
  versión anterior.
- Verificado con headless Chrome: la tarjeta y sus 5 campos existen y están correctamente enlazados;
  bajar `FS.capacidad_carga` a 1.0 hace que el mismo caso pase a CUMPLE; subirlo a 100 lo vuelve a
  fallar; la fila nueva aparece en Resultados; la barra de resultados fija refleja el peor de los dos
  criterios; sin errores de JS. **Why:** el usuario quiere que TODOS los umbrales de aceptación del
  proyecto sean ajustables desde un solo lugar (norma/cliente distintos piden factores distintos), no
  solo los que ya estaban expuestos por casualidad. **How to apply:** si se agrega un factor de
  seguridad nuevo al motor en el futuro, agregarlo también a esta tarjeta — no dejarlo como constante
  hardcoded ni agregarlo a `state.FS` sin conectarlo a un chequeo real (como pasó con
  `capacidad_carga` original y con `sismo.FS_min_sism`, que sigue sin usarse en ningún lado — no se
  tocó en este cambio porque el usuario no lo mencionó, pero es candidato a revisar si se pregunta por
  ella más adelante).

**Sobrecarga oculta en "Suelo y materiales" y "Geometría del muro" (2026-08-16):** el usuario pidió,
en dos mensajes seguidos, "no muestres la sobrecarga en la pestaña de Suelo y materiales" y luego
"también de geometría del muro". La franja de sobrecarga (`state.cargas.qsEnabled`, dibujada sobre el
relleno en `dibujarMuro2D`) NO estaba condicionada a ninguna pestaña — se veía en todas mientras
estuviera activa. Se agregó un flag propio `mostrarQs = tabActiva!=="suelo" && tabActiva!=="geometria"`
(declarado junto a `mostrarSuelo`/`mostrarPresiones` al inicio de la función, mismo patrón) y se
condicionó su bloque a `state.cargas.qsEnabled && mostrarQs` — sigue visible en Cargas/Presiones
laterales, que son las dos pestañas donde sí es el foco. Verificado con headless Chrome en las 4
pestañas. **Why:** mismo patrón de decluttering por pestaña de toda la sesión (Pa/Pp solo en
Presiones, cotas apagadas en Suelo/Presiones) — cada pestaña muestra en el canvas solo lo que es su
foco.

**Cotas activadas por default en "Geometría del muro" (mismo día):** el usuario pidió, justo después
de lo de la sobrecarga, "en geometría del muro activa por default las cotas". Patrón INVERSO al ya
usado para Suelo y materiales/Presiones laterales (que las apagan por default): se agregó a
`actualizarVisorSegunPestana(tabId)` un segundo `if(tabId === "geometria")` que hace
`uiState.showCotas = true` y marca `#chkCotas` de verdad — así, si el usuario venía de una pestaña
donde quedaron apagadas (por default o a mano), Geometría del muro las vuelve a prender porque ahí sí
son el foco (es la pestaña donde se ajustan las dimensiones). Mismo mecanismo que las demás reglas de
esta sesión: es un default por visita, no un bloqueo — el usuario puede apagarlas a mano mientras está
ahí, y se reactivan si sale y vuelve a entrar. Verificado con headless Chrome (secuencia completa:
suelo→apaga, geometría→prende, apagar a mano en geometría, salir y volver→vuelve a prender). **Why:**
mismo patrón de "cada pestaña con su default útil" — Geometría es donde se editan H/ts/tc/Bf/tf/toe/
heel/Df/β, las cotas son directamente su foco, a diferencia de Suelo/Presiones donde compiten con
otras anotaciones.

**Flechas + cotas de brazo para los 5 pesos propios, siempre visibles en Cargas (2026-08-16):** el
usuario pidió "vas a colocar los pesos calculados indicados con una flecha que coincida con el eje
donde se aplica la carga y las cotas van a estar en función de esas longitudes con respecto al punto
de referencia" y pidió explícitamente que le hiciera 5 preguntas de alcance antes de construir
(herramienta `AskUserQuestion`, 2 rondas por el límite de 4 preguntas por llamada). Respuestas
confirmadas: los 5 pesos (Ws/Wf/Wr/Wtri si hay talud/Wp); SIEMPRE visibles en la pestaña Cargas
(conviviendo sin tocar el sistema de hover que ya existía para estos mismos 5 campos — el hover sigue
mostrando velo+polígono de área+fórmula al pasar el cursor, esto es una capa adicional permanente);
punto de referencia x=0 (la punta de la puntera, mismo origen que ya usa `Mr` en
`calcularGeotecnia`); flecha vertical descendente estilo `Pu_torre` (línea + triángulo dibujado a
mano, apunta hacia abajo, con la fuerza como etiqueta).

- **Brazos reutilizados, no recalculados:** `Mr = Ws·brazoStem + Wf·(Bf/2) + Wr·(Bf-heel/2) +
  Wtri·brazoTri + Wp·(toe/2)` ya existía en `calcularGeotecnia` — los mismos 5 términos (`brazoStem`/
  `Bf/2`/`Bf-heel/2`/`brazoTri`/`toe/2`) son los que usan las flechas/cotas nuevas, mismo patrón de
  "transparencia sin duplicar cálculo" de toda la sesión.
- **Carriles escalonados, ordenados de menor a mayor brazo:** las 5 flechas (arriba de la corona) y
  las 5 cotas (debajo de la zapata) comparten el mismo índice de "carril" por peso — se ordenan por
  `x` ascendente una sola vez (`pesos.sort((a,b)=>a.x-b.x)`) y ese mismo índice `i` decide tanto la
  altura de la flecha (`tf+H+pesoArrowBase+i*pesoArrowStep`, escalonada para que ninguna etiqueta
  choque con la de al lado) como la profundidad de la cota (`-(pesoCotaBase+i*pesoCotaStep)`,
  ANIDADA — el brazo más corto en el carril más cercano a la zapata, el más largo en el más alejado,
  convención estándar de acotado ya que las 5 cotas comparten el mismo origen x=0).
- **Espacio vertical reservado solo en "Cargas":** mismo patrón que los diagramas de presión lateral
  reservan espacio horizontal solo en su pestaña — aquí `modelMinY`/`modelMaxY` se extienden (usando
  `pesoArrowBase/Step` y `pesoCotaBase/Step`, definidos junto a los demás parámetros de layout cerca
  del inicio de `dibujarMuro2D`) solo cuando `mostrarCargas` es true, sin tocar el resto de pestañas.
- **Efecto colateral encontrado y corregido antes de dar el trabajo por terminado:** al agrandar
  tanto el lienzo verticalmente, las cotas GENERALES preexistentes (H/Df/tf/toe/ts/heel/Bf/tc, que sí
  se mostraban por default en Cargas) empezaron a chocar entre sí y con las etiquetas nuevas —
  confirmado con el mismo script de `getBoundingClientRect()` de la sesión (varios overlaps reales,
  ej. "Wp=348 kg/m" contra "tc=0.20 m"). Se agregó "cargas" a la lista de pestañas donde
  `actualizarVisorSegunPestana` apaga las cotas generales por default (mismo mecanismo que Suelo/
  Presiones) — tras el cambio, 0 overlaps reales en ambos escenarios probados (con y sin talud).
- Verificado con headless Chrome: los 5 pesos (incluido Wtri condicionado a `hayTalud`) con valores y
  brazos correctos; 0 overlaps reales; NO aparecen en ninguna otra pestaña (confirmado en Geometría);
  captura de pantalla confirmando el resultado visual (flechas escalonadas arriba, cotas anidadas
  abajo, sin choques). **Why:** el usuario quiere ver DÓNDE actúa cada peso (no solo el número), mismo
  hilo de transparencia visual de toda la sesión, ahora aplicado a las cargas verticales además de las
  laterales. **How to apply:** si se agrega un peso/fuerza vertical nueva a la tarjeta de Cargas,
  sumarla al arreglo `pesos` con su brazo YA CALCULADO por el motor (nunca inventar uno nuevo) —
  el ordenamiento y el escalonado de carriles se ajustan solos.

**Bug real: etiquetas de presión en los extremos del diagrama redondeaban a "0" (2026-08-16):** el
usuario mandó una captura del diagrama de Sobrecarga en Presiones laterales, con las dos etiquetas de
presión extrema circuladas a mano, mostrando "0 Tn/m²" arriba y abajo — mientras que arriba del
esquema se leía "qs=1.00 Tn/m²" y la propia resultante decía "Pq,h=2 Tn/m", claramente no-cero.
Causa: `pressureExtremeLabels` (ver el bloque "que los diagramas no se encimen" de más arriba)
llamaba `fmtCat("presion", valor, 0)` — CERO decimales. La sobrecarga es una presión UNIFORME (mismo
valor arriba y abajo, a diferencia de activo/pasivo que sí llegan a 0 real en la superficie), y con
qs=1.00 Tn/m² y Ka≈0.33, el valor real es ~0.33 Tn/m² — que redondeado a 0 decimales da literalmente
"0". El cálculo (`Pq,h`) SIEMPRE estuvo bien, era puramente una etiqueta con muy poca precisión para
una unidad "grande" como Tn/m² — mismo patrón que el bug del preset de sobrecarga de antes en esta
sesión (un problema de display, no de motor, pero indistinguible de un error real desde la captura).
Se subió a 2 decimales. Verificado con headless Chrome (qs=1.00 Tn/m²): ahora muestra "0.33 Tn/m²" en
ambos extremos, sin overlaps nuevos por el texto más largo. **Why:** mismo patrón de "verificar antes
de asumir, pero tomar en serio los reportes del usuario" de toda la sesión. **How to apply:** si
aparece otro "0" sospechoso en una etiqueta de presión/fuerza con una unidad grande (Tn/m², MPa,
kip/ft), revisar los decimales de ese `fmtCat`/`fmtFuerza` antes de asumir que el valor real es cero.

**[Extensión, mismo día] "Que todos los resultados tengan 3 decimales. No redondees a número
entero":** inmediatamente después del bug de arriba, el usuario pidió subir la precisión de forma
general, no solo en ese diagrama puntual.

- **Default global:** `fmtCat(cat, baseVal, dec)` cambió su default de `dec===undefined?2:dec` a
  `dec===undefined?3:dec` — arregla de un solo lugar cualquier campo `out-*`/`res-*` que llame
  `fmtFuerza`/`fmtMomento`/`fmtPresionSuelo`/`fmtLong` SIN pasar decimales explícitos (la mayoría).
- **Overrides explícitos corregidos a 3** en los lugares donde este mismo día se habían bajado a 0
  decimales a propósito (para ahorrar espacio en los carriles de los diagramas — ver las entradas de
  arriba): la fuerza de `pressureResultant`, la etiqueta de `Pu_torre`, las 5 etiquetas de peso
  propio, la de `Cerco P=`. También se subieron a 3 los `fmt(...,2)` de los valores FS
  (`FS_volteo`/`FS_desliz`/`FS_carga`/`FS_talud_simple`) en `renderRevisiones`,
  `renderResultadosDetalle` y `renderResultsBar` — antes en 2. **NO se tocaron** las cifras dentro de
  fórmulas-desglose (ángulos φ/β/δ/θ/ψ en las funciones `formula*`, γ/c dentro de esas mismas
  fórmulas, Ka/Kp que ya vienen en 3-4 decimales) — esos son parámetros de ENTRADA mostrados dentro
  de una fórmula sustituida, no "resultados calculados" en el sentido que pidió el usuario.
- **Efecto colateral esperado y ya corregido: texto más largo volvió a chocar.** Con 3 decimales
  ("Pa,h=3366.750 kg/m" en vez de "3367") los mismos problemas de espacio de antes reaparecieron. Se
  aplicó el mismo patrón de 3-parámetros-a-la-vez de las diagramas de presión (`diagW` 1.3→1.2,
  `diagGap` 1.3→1.6, márgenes de título 0.9→1.0) MÁS bajar la fuente de las etiquetas de resultante y
  de extremos de presión (10px/9px → 9px/8px) — la reducción de fuente fue el ingrediente nuevo que
  faltaba en intentos anteriores de este mismo problema.
- **`dimH` ganó un 7º parámetro opcional `fontSize`** (default `"10px"`, sin romper ninguna llamada
  existente) para poder bajar a 9px solo las cotas de brazo de los pesos (`brazo Ws=...`), sin afectar
  las cotas generales (H/Df/toe/etc.) que siguen a 10px.
- **Bug real encontrado en el camino, con captura:** con `dist_torre=0` (default) Y el cerco activo,
  la etiqueta `Pu_torre=...` se fundía visualmente con `Cerco P=...` (confirmado con captura, el
  texto se leía corrido). La causa NO era solo el caso degenerado `dist_torre=0` — persistía incluso
  con `dist_torre=2.5` real, porque ambas etiquetas viven a alturas parecidas sin importar la x
  (`Cerco` a `tf+H+h_cerco·0.55`, `Pu_torre` a una altura fija `0.9` sobre el terreno). Dos intentos
  fallidos antes de la solución: (1) subir la flecha por encima del cerco → trasladó el choque hacia
  las etiquetas de los pesos Wp/Ws; (2) nada más achicar la fuente → no alcanzaba. Solución real:
  anclar `arrowLen` de `Pu_torre` al MISMO sistema de carriles que los 5 pesos
  (`pesoArrowBase + 5*pesoArrowStep`, un carril por encima del más alto de los 5) — como
  `pesoArrowBase` ya se ajusta a `h_cerco` cuando el cerco está activo, esto despeja tanto a los pesos
  como al cerco de una sola vez. `modelMaxY` se ajustó de `4*pesoArrowStep` a `5*pesoArrowStep` para
  la pestaña Cargas, dándole espacio real a este carril extra.
- Verificado con headless Chrome: 0 overlaps reales en Cargas (`dist_torre=0` y `dist_torre=2.5`,
  ambos con cerco activo); Presiones laterales con overlaps residuales de ≤9px (confirmado con
  captura que no son perceptibles visualmente, dos etiquetas de resultantes de carriles distintos
  quedan cerca pero legibles); FS ahora muestran 3 decimales en Revisiones/Resultados/barra fija.
  **Why:** el usuario prefiere precisión completa sobre densidad visual — prefiere un lienzo más alto/
  ancho con más aire entre etiquetas que perder información por redondeo. **How to apply:** si se
  agrega otra anotación con texto largo al canvas, considerar de entrada un font-size más chico
  (9px/8px) en vez de recortar decimales — recortar decimales ya no es una opción disponible para
  "resultados" desde este pedido explícito del usuario.

**[Corrección, mismo día] Flechas de peso a la mitad del elemento, no apiladas arriba de la corona:**
el usuario pidió "que las indicaciones de las cargas no estén tan arriba, preferentemente que estén a
la mitad del elemento del que están obteniendo el resultado". La primera versión de las flechas de
peso propio (bloque de arriba, "Pesos propios") apilaba las 5 en una escalera artificial de carriles
por encima de la corona (`pesoArrowBase`/`pesoArrowStep`), sin relación con dónde vive realmente cada
peso — exactamente lo que el usuario pidió corregir.

- **Cada flecha ahora usa `yMid`, la altura media REAL del área que ese peso representa** — misma
  geometría que ya usa `highlightRegionPoints()` para el resaltado de hover, nunca inventada aparte:
  Ws y Wr comparten el mismo rango (mitad del fuste = mitad del relleno sobre el talón, de `tf` a
  `tf+H`, porque ambas regiones ocupan ese mismo rango vertical); Wf usa la mitad de la zapata (`0` a
  `tf`); Wp usa la mitad del trapecio de suelo sobre la puntera (`tf` a `Df`, con un piso de
  `tf+0.05` para el caso raro `Df≈tf`); Wtri usa la mitad del triángulo de talud (`tf+H` a
  `tf+H+talRise`). Cada flecha pasó de ser una línea larga en un carril artificial a un vector corto
  (`halfLen=0.22` m) centrado en `yMid`, apuntando hacia abajo.
- **Se eliminaron `pesoArrowBase`/`pesoArrowStep`** (ya no hacen falta) y con ellos el término extra
  de `modelMaxY` para la pestaña Cargas — el lienzo ya no necesita agrandarse verticalmente para las
  flechas (solo las cotas de abajo siguen necesitando su espacio reservado, eso no cambió). `Pu_torre`
  volvió a un `arrowLen` fijo y corto (0.9, como el diseño original) — ya no necesitaba la torre de
  carriles, que existía solo para esquivar a las 5 flechas de peso, que ahora viven mucho más abajo.
- **Efecto colateral encontrado y corregido:** con las flechas en su altura real, Ws y Wr (que
  SIEMPRE comparten `yMid`, por definición de sus regiones) chocaban entre sí cuando su separación en
  x no alcanzaba a la escala del lienzo; por separado, Wp (el brazo más chico, típicamente cerca del
  origen) chocaba con la etiqueta preexistente "NTN" (que vive a la izquierda de x=0). Se corrigió
  cambiando el `text-anchor` de las etiquetas afectadas para que CREZCAN ALEJÁNDOSE del conflicto en
  vez de crecer centradas hacia él: el peso con el brazo más chico del conjunto siempre ancla "start"
  (crece a la derecha, lejos de NTN); cuando dos pesos comparten `yMid` (detectado en vivo comparando
  todas las alturas, no hardcodeado a "Ws y Wr"), el de menor x ancla "end" y el de mayor x ancla
  "start" — crecen en direcciones opuestas en vez de encimarse en el centro.
- Verificado con headless Chrome: 0 overlaps reales sin extras, con talud, y con torre a una
  distancia real (2.5 m) + cerco a la vez — el único choque que sigue apareciendo es el ya documentado
  y aceptado de `dist_torre=0` (sin configurar) contra el cerco, que se resuelve capturando una
  distancia real. Confirmado también con captura de pantalla: cada flecha cae visualmente DENTRO del
  elemento que representa (Wtri en el triángulo del talud, Wr en el relleno, Ws en el fuste, Wp cerca
  de la puntera, Wf en la zapata), el lienzo quedó notablemente más compacto verticalmente. **Why:**
  el usuario quiere que la posición de cada indicador comunique DÓNDE físicamente actúa ese peso, no
  solo agrupar las 5 flechas de forma prolija arriba. **How to apply:** si se agrega un peso/fuerza
  nueva a este bloque, definir su `yMid` a partir de la misma geometría que ya usa
  `highlightRegionPoints()` para esa variable (nunca un carril artificial) y revisar si comparte
  altura con algo existente para replicar el patrón de anclaje "crecer lejos del conflicto".

**Toggle "Cotas" oculto por completo en "Cargas" (2026-08-16):** el usuario pidió "en cargas quita la
opción de ver las cotas, en este dibujo no son necesarias" — un paso más allá del default-off ya
existente ahí (que solo dejaba el checkbox desmarcado, pero seguía visible/clickeable). Se le agregó
`id="wrapCotas"` al `<label class="viewer-toggle">` que envuelve `#chkCotas` (vive en la barra del
visor, fuera del sistema de pestañas) y `actualizarVisorSegunPestana(tabId)` ahora también hace
`document.getElementById("wrapCotas").style.display = (tabId==="cargas") ? "none" : ""` — el control
completo desaparece en Cargas (no solo se desmarca) porque esa pestaña ya tiene su propio sistema de
cotas de brazo por peso propio (siempre visible), y las cotas generales (H/Df/tf/toe/ts/heel/Bf/tc)
no aportan nada ahí. En el resto de pestañas el toggle sigue visible como siempre (Suelo/Presiones
solo lo desmarcan por default, igual que antes). Verificado con headless Chrome: `display:none` en
Cargas, `display:flex` en Geometría/Suelo.

**Tabla "Resumen de pesos y momentos" en Cargas (mismo día):** el usuario mandó una captura de otro
software (columnas Zona/Material/γ/Área/Volumen/Peso=V×γ/Brazo/Momento, con fila de totales ΣFv/
ΣMFv) y pidió una tabla así, "que se ubique del lado derecho del esquema del muro" — como el panel de
formulario (`#content`) ya vive a la derecha del visor (`#viewer-pane`) en todas las pestañas, se
agregó como una tarjeta nueva ahí mismo, en Cargas, después de "Peso propio (calculado)".

- **Columnas:** Zona (número secuencial), Material ("Concreto"/"Suelo"), γ, Área, Peso = V·γ, Brazo,
  Momento, con fila `<tfoot>` de totales ΣFv/ΣM. Se omitió la columna "Volumen" del original (redundante
  con Área ya que el largo asumido es 1 m de muro — aclarado en el texto de la tarjeta) para no forzar
  una 8ª columna en un panel angosto.
- **`renderTablaPesos(r)`** (llamada desde `renderCargasOutputs`, se re-ejecuta en cada render):
  reusa Ws/Wf/Wr/Wtri/Wp, `areaStem`, `areaWp`, `brazoStem`, `brazoTri` YA CALCULADOS por
  `calcularGeotecnia` — nunca recalculados aparte, mismo patrón de toda la sesión. Wr y Wtri no
  traían una "γ propia" expuesta (su peso sale de sumar capa por capa del relleno estratificado) —
  se DERIVA el área real (`heel×Hrelleno` para Wr, `½·heel²·tanβ` para Wtri) y el γ "efectivo" como
  `peso/área` (para Wr) o `capasPeso[0].gamma` (para Wtri, la capa superior — mismo criterio que ya
  usaba `formulaWtri`) — matemáticamente exacto (`peso = área×γ` se sigue cumpliendo), no una
  aproximación nueva. La fila de Wtri solo aparece si `geo.hayTalud`.
- **CSS nueva:** `table.results tr.zona-concreto/zona-suelo td` tiñen cada fila con
  `color-mix(in srgb, var(--concrete-fill)/var(--soil-fill) N%, transparent)` — mismos colores que ya
  usa el dibujo 2D para concreto/suelo, para que la tabla se sienta ligada al esquema; `tfoot td` con
  negritas y borde superior más marcado para la fila de totales. La tabla se envuelve en
  `overflow-x:auto` (7 columnas no caben sin scroll en un panel de ~450px de ancho, confirmado con
  captura — se acepta el scroll horizontal como respaldo, mismo patrón ya usado en otras tablas
  anchas de la app).
- Verificado con headless Chrome: 4 filas sin talud / 5 con talud, valores y suma de la columna Peso
  coinciden con el total del footer, sin errores de JS. **Why:** el usuario quiere una vista tipo
  "memoria de cálculo" de los pesos con área/brazo/momento en una sola tabla, no solo los 5 valores
  sueltos que ya mostraba "Peso propio (calculado)" (que se dejó intacta, esto es adicional). **How
  to apply:** si se agrega un peso/zona nueva, sumarlo al arreglo `zonas` de `calcularZonasPesos` con
  su área/γ/brazo YA CALCULADOS por el motor — nunca inventar una fórmula nueva ahí.

**[Corrección, mismo día] Tabla de pesos movida DENTRO del canvas:** el usuario, tras ver la tabla en
la tarjeta del panel lateral, aclaró "Pero dentro del canvas" — la quería literalmente junto al
esquema del muro, no en el panel de formulario. Se rediseñó por completo el mecanismo de dibujo:

- **Se quitó la tarjeta HTML** ("Resumen de pesos y momentos") de la pestaña Cargas — ya no vive en
  `#content`. `renderTablaPesos(r)` se partió en dos funciones puras (sin tocar el DOM directamente):
  `calcularZonasPesos(r)` (arma el arreglo `zonas` + `sumaFv`/`sumaM`, la misma lógica de antes sin
  cambios) y `pesosTablaHTML(r)` (arma el string HTML de la tabla completa, reusando las mismas clases
  CSS `table.results`/`zona-concreto`/`zona-suelo` que ya existían).
- **Se dibuja como `<foreignObject>` dentro de `#wall2d`** (HTML real embebido en el SVG), en vez de
  reconstruir cada celda a mano con `<text>`/`<tspan>` — mucho más simple, y CSS de la página aplica
  normal dentro del foreignObject (mismo documento). Vive FUERA del bloque `uiState.showForces`
  (es una tabla de datos, no una fuerza que tenga sentido ocultar con ese toggle) pero sigue
  condicionado a `mostrarCargas`.
- **Reserva de espacio horizontal:** se agregó `pesoTablaAncho=3.6` m al cálculo de `viewMaxX` (mismo
  mecanismo que ya usan los diagramas de presión lateral para reservar espacio sin tocar
  `modelMinX/modelMaxX` — la geometría real no se mueve, el lienzo solo se ve más chico dentro de una
  ventana más ancha). El `<foreignObject>` se posiciona en PÍXELES (no vía `X()/Y()` en metros, porque
  su tamaño es de UI, no geométrico), arrancando en `X(modelMaxX) + 22px`.
- **Bug real encontrado y corregido: alto del `<foreignObject>` insuficiente.** Un `<foreignObject>`
  RECORTA su contenido al `height` declarado (no crece solo, a diferencia de un `<div>` normal en
  flujo) — la primera versión estimaba el alto con una fórmula fija (`40 + filas×30 + 34`), que
  resultó ser MENOR que el alto real renderizado del `<table>` (confirmado con una captura: la fila 5
  y el pie de totales quedaban cortados a la mitad cuando había talud). Se corrigió insertando la
  tabla con un alto provisional, dejando que el navegador la renderice (el `<foreignObject>` ya está
  adjunto al DOM vivo en ese momento), y luego leyendo `divTabla.scrollHeight` (fuerza un reflow
  síncrono, técnica estándar) para fijar el `height` real + 8px de margen — ya no es una estimación,
  es una medición.
- Verificado con headless Chrome: el `<foreignObject>` existe solo en Cargas (no en Suelo/Geometría),
  con posición/tamaño correctos; 4 filas sin talud (alto medido 314px) / 5 con talud (alto medido
  368px, antes se estimaba solo 224px — la fila y el pie ya no se cortan); confirmado también con
  captura de pantalla. **Why:** el usuario quiere la tabla como parte del mismo esquema visual, no
  como un dato aparte en el formulario. **How to apply:** cualquier `<foreignObject>` nuevo con
  contenido HTML de alto variable debe medir `scrollHeight` después de insertarlo en el DOM vivo, no
  estimarlo con una fórmula — un foreignObject no se autoajusta como un `<div>` normal.

**[Corrección, mismo día] Más separación + texto en una sola línea, sin ajustarse al ancho:** el
usuario pidió "un poco más separado y que todos los textos se vean en una sola fila, no las ajustes
a la anchura". Se agregó `#tablaPesos th, #tablaPesos td{white-space:nowrap; ...}` en CSS, pero eso
solo evita el wrap DENTRO de cada celda — si el `<foreignObject>` es más angosto que el contenido sin
wrap, el texto se desborda o se recorta en vez de verse. El primer intento de agrandar el
`anchoTablaPx` (470→640) y reposicionar expuso un **bug real de fondo en cómo se reservaba el
espacio**:

- **La reserva en METROS (`pesoTablaAncho`, como ya hacían los diagramas de presión lateral) NO
  garantiza espacio en PÍXELES.** En "Cargas" la escala casi siempre está limitada por el ALTO del
  lienzo (las cotas de brazo apiladas debajo de la zapata), no por el ancho — agrandar el modelo
  horizontalmente no cambia esa escala (sigue atada al alto) hasta un punto de cruce muy lejano, así
  que la tabla terminaba encimada con el muro o recortada fuera del viewBox (confirmado midiendo con
  `getBBox()` de todos los elementos del SVG contra la posición de la tabla, en dos iteraciones
  fallidas antes de encontrar la causa real).
- **Además, las ETIQUETAS de texto (labelBg, cotas) usan `font-size` en píxeles FIJOS, no escalado
  con el zoom del modelo** — así que aunque se lograra encoger la escala geométrica al mínimo, las
  etiquetas (p.ej. "brazo Wr=1.400 m") no encogen con ella y siguen ocupando su mismo ancho en
  píxeles, poniendo un piso real a cuánto se puede comprimir el muro.
- **Solución real: reservar PÍXELES directamente, del lado derecho únicamente**, mismo mecanismo que
  ya usa `pad` pero asimétrico. Se agregó `reservaTablaPx = anchoTablaPx+gapTablaPx` (cuando
  `mostrarCargas`) restado del ancho disponible tanto en el cálculo de `scale`
  (`(viewW-2*pad-reservaTablaPx)/modelW`) como en `offX` (centra el muro dentro de
  `[pad, viewW-pad-reservaTablaPx]`, no dentro de todo `viewW`) — por construcción, el muro (y sus
  etiquetas) NUNCA invaden esa franja reservada, sin importar si la escala termina atada al ancho o
  al alto. `anchoTablaPx`/`gapTablaPx` se declaran junto a `scale`/`offX` (antes vivían más abajo,
  junto al bloque que dibuja el `<foreignObject>`) para que ambos lugares usen los mismos números.
- **Nota de verificación:** la primera ronda de pruebas con headless Chrome usó el tamaño de ventana
  por default (sin `--window-size`), dando un viewBox angosto (~708px) no representativo del uso real
  de la app (ventana de escritorio, típicamente 1400-1900px) — con ese viewBox angosto SÍ seguía
  habiendo traslape aun con la reserva en píxeles, lo que llevó a dudar de la solución. Repetir la
  prueba con `--window-size=1700,1000` (tamaño realista) confirmó CERO traslape y CERO recorte —
  lección: al verificar layout de este visor, usar un tamaño de ventana representativo del uso real,
  no el default de headless Chrome.
- Verificado con headless Chrome + captura a tamaño realista: alturas de celda uniformes (sin wrap),
  tabla completa visible dentro del viewBox, sin encimarse con el muro/sus etiquetas, sin errores de
  JS. **Why:** el usuario quiere leer cada valor de un vistazo, sin que el texto se corte o se apile.
  **How to apply:** para reservar espacio de UI fijo (no geométrico) dentro de este canvas, reservar
  PÍXELES en el cálculo de `scale`/`offX` (patrón `reservaTablaPx`), nunca metros del lado que sea
  más angosto — y recordar que las etiquetas de texto no encogen con el zoom del modelo.

**[Extensión, mismo día] Sobrecarga agregada a la tabla de pesos:** el usuario notó "Falta considerar
la sobrecarga" — el ΣFv/ΣM de la tabla solo sumaba los 5 pesos propios (Ws/Wf/Wr/Wtri/Wp), sin la
carga vertical que aporta `qs` cuando está activa. Se agregó una 6ª fila condicional (solo si
`state.cargas.qsEnabled && geo.Pq_v > 1e-9`) en `calcularZonasPesos`, reusando `geo.Pq_v` (=qs·heel)
y su brazo (`Bf-heel/2`) EXACTAMENTE como ya los usa `Mr` en `calcularGeotecnia`
(`Mr += ... + Pq_v*(Bf-heel/2) + ...`) — nunca recalculado aparte.

- **No es un peso propio** (no tiene γ/volumen, es una presión sobre un área) — se le agregó un campo
  opcional `gammaCat` a cada zona (default `"pesoVol"`, esta fila usa `"presion"`) para que la columna
  γ muestre `qs` en la unidad correcta en vez de forzar la categoría de peso volumétrico. La
  aritmética `Peso = Área×valor` se sigue cumpliendo igual (`qs×heel = Pq_v`).
- **`Pa_v` (componente vertical del empuje activo, también sumado en `N`/`Mr`) se dejó FUERA a
  propósito** — es una fuerza lateral con componente vertical, conceptualmente distinta a una carga
  vertical tipo "peso"; el usuario pidió específicamente la sobrecarga, no todas las componentes
  verticales de `N`.
- Nueva clase CSS `zona-sobrecarga` (tinte con `--pq-color`, el mismo naranja que ya usa `qs` en el
  resto del dibujo) para diferenciarla visualmente de `zona-concreto`/`zona-suelo`.
- Verificado con headless Chrome: con qs=0.5 Tn/m² (500 kg/m²) y heel=1.4 m, la fila muestra
  Peso=700.000 kg/m y Momento=980.000 kg·m/m (qs×heel y Peso×brazo, exacto); ΣFv/ΣM crecen en esa
  misma cantidad respecto al caso sin sobrecarga; el `<foreignObject>` crece solo (170→197px,
  `scrollHeight` real) para dar cabida a la fila nueva sin cortarla; sin sobrecarga activa la fila NO
  aparece (vuelve a 4 filas); sin errores de JS. **Why:** el usuario quiere que la tabla represente el
  total real de carga vertical actuando sobre la zapata, no solo el peso propio del muro/relleno.
  **How to apply:** si se agrega otra carga vertical al motor en el futuro (además de sobrecarga),
  seguir el mismo patrón: reusar el valor y brazo YA CALCULADOS en `calcularGeotecnia`, decidir su
  `gammaCat` si no es un peso volumétrico, y agregar una clase de color si aplica.

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
