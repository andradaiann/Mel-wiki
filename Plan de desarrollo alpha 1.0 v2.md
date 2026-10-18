Plan para llevar el motor desde el estado actual (prealpha 0.1.0) hasta la **alpha 1.0**, momento en el que las bases quedan congeladas y se pasa a escribir contenido real.

Dos reglas que valen para todo el plan:

1. **Cada versión tiene que dejar el juego corriendo.** Si un bloque queda a medias, no se sube.
2. **Lo que no se ve va primero.** Un rework de arte se puede rehacer sin romper nada; una decisión de arquitectura mal tomada obliga a rehacer todo lo que se apoyó encima.

El orden de los bloques no es arbitrario: cada uno depende del anterior. El shader de temperatura va antes que las paletas porque los colores hay que elegirlos mirándolos a través de él. Las paletas van antes que el arte de personajes porque es donde se aplican. Las UIs van últimas porque son lo que más duele rehacer y lo que más depende del color.

*La escritura no depende del motor: sigue en paralelo en la wiki (café, parque, puente), así cuando el motor se congele el contenido ya está esperando.*

---

## 0.2 — Cimientos invisibles

No cambia nada en pantalla. Es la limpieza que conviene hacer ahora que el proyecto está vacío.

### 0.2.0 — Separar flags de historia y estado de sesión

Hoy `lock_move` vive entre los flags. Si eso se guarda, un autosave que caiga en un momento bloqueado deja una partida en la que no se puede caminar nunca más, y con autosave único eso es una partida muerta. No es el único: hay flags que son estado de una cutscene y no historia (por ejemplo `lu_spam_3`).

La idea: lo que es **historia** se guarda (decisiones, conversaciones ya tenidas, estado del día). Lo que es **estado momentáneo** no se guarda nunca y arranca siempre limpio (bloqueos de movimiento, menús abiertos, cutscenes, transiciones).

Primer paso: auditar todas las banderas del código y clasificarlas (ver [[Auditoría de banderas]]).

**Lo importante no es solo qué se guarda, sino cómo se reconstruye la escena al cargar.** Al cargar no hay ninguna cutscene corriendo, no está el tinte, no suena el motor. El guardado tiene que decir *en qué momento estás y qué decidiste*; la escena la reconstruye el momento.

Eso ya existe a medias: las cutscenes `debug_<momento>` arman el estado de un momento desde cero. **Cargar una partida y saltar con el debug tienen que ser el mismo camino.** Un solo sistema, probado dos veces.

*Terminado cuando:* se puede guardar en cualquier punto de autosave, cerrar el juego, cargar, y la escena se reconstruye igual que si se hubiera llegado jugando.

### 0.2.1 — Guardado a prueba de futuro

Durante meses se van a agregar flags nuevos. El guardado tiene que aguantar eso sin romperse y sin obligar a tocarlo cada vez: agregar un flag nuevo debería ser gratis, y una partida vieja debería cargar aunque no conozca los flags que aparecieron después.

- Las banderas ya lo aguantan: si una no existe, `flag_get` devuelve el valor por defecto.
- Lo que hoy se rompería son **los chats**: si se agrega un contacto nuevo, una partida vieja no lo tiene. Solución: al cargar, arrancar de `chats_init()` y pisar con lo guardado, contacto por contacto.
- El archivo ya tiene `version: 1`: sirve para migraciones si algún día cambia la estructura.

*Terminado cuando:* se agrega un flag nuevo y un contacto nuevo sin tocar nada del guardado, y un save viejo sigue cargando.

### 0.2.2 — Testeo real del guardado

Hace falta poder ver el estado de los flags para testearlos: una página en el menú de debug que los muestre en vivo.

El test que vale es con el **ejecutable cerrado**: guardar, cerrar el juego, abrir, cargar. Guardar y cargar sin cerrar da falso positivo. Probar también guardando en una transición, con el celular abierto y a mitad de un diálogo.

### 0.2.3 — (opcional) Dueño del input

Lo que hay hoy resuelve *si* el jugador se puede mover, pero no *quién recibe la tecla*. No molesta mientras los sistemas se excluyan entre sí; cuando llegue el primer minijuego con controles propios va a hacer falta una sola respuesta a esa pregunta. Se puede dejar para ese momento.

Síntoma de que se acerca: cada objeto tiene su propia línea de `global.paused || global.debug_menu || ...`. La solución limpia es una sola función que diga quién tiene el input (debug, pausa, UI, diálogo, cutscene, jugador), y que cada sistema pregunte si es él.

---

## 0.3 — Luz y color

### 0.3.0 — Shader de temperatura y hora

Aunque sea tosco. Tiene que existir **antes** del trabajo de paletas, porque un tinte global sobre una paleta acotada le come justamente lo que la hace funcionar, y los colores elegidos sin el tinte no son los colores que se ven.

**Armarlo como un sistema de post-proceso con efectos apilables, no como "el shader de la temperatura".** Es el mismo lugar donde después van a vivir los efectos del trance (separación de colores, ondas, respiración) y el grano. Así el viaje es sumar efectos a algo que ya anda.

Decisiones a tomar acá, no después:
- Cuánto del tinte recibe la UI. Si recibe cero, de noche la caja de diálogo queda blanca sobre un mundo azul y se despega como un sticker. Una fracción es mejor que nada.
  - *Ojo técnico:* el post-proceso procesa la imagen del mundo; las interfaces se dibujan después, aparte. Darle una fracción del tinte implica dibujar la UI en una superficie propia, o aplicarle el mismo shader con menos intensidad al dibujarla.
- La pantalla del celular es emisiva: que sea lo único que no cambia de temperatura durante el día es un detalle gratis y lindo.
- El tinte de `obj_screen` (la penumbra del micro) probablemente se mude acá.

### 0.3.1 — Room de prueba de color

Un escenario de prueba con personajes y fondo, y un control para barrer temperatura y hora. Rato de trabajo, se usa durante todo el proyecto.

### 0.3.2 — Paletas nuevas

- El color natural es el **punto medio**: interior, media tarde. Es la referencia para dibujar.
- Validar en los **extremos**, no en el medio: atardecer en el puente y noche con frío. Son los dos casos que duelen y encima son los narrativamente más importantes del tramo.
- Revalidar bajo el shader las paletas ya armadas (micro, terminal, abrazo, parque).

*Terminado cuando:* la paleta aguanta las tres condiciones sin empastarse ni perder lecturas.

---

## 0.4 — Personajes

### 0.4.0 — Nueva base

Cuerpos más cortos, cabeza con forma distinta, colores nuevos. Los pelos y ropas existentes se importan a la base nueva con retoques mínimos.

Cambiar proporciones toca más de lo que parece: máscaras de colisión, origins, el sprite del abrazo, los retratos, el centrado del abrazo, de dónde salen los corazones. 

### 0.4.1 — Plantilla de expresiones faciales

Un set fijo de expresiones sobre la base, para que agregar un personaje nuevo sea trabajo de ropa y pelo, no de cara.

### 0.4.2 — Outfits invernales

- Mel y Lu con abrigo como default.
- Los outfits actuales (ropa de calor) pasan a ser la ropa de interior en la casa de Lu.
- Anotar en la wiki en qué momento exacto se desabrigan: es un cambio que el juego tiene que disparar en algún lado, y además es una escena.

---

## 0.5 — Interfaces

### 0.5.0 — UI de diálogo

- Recorte de papel en lugar de la caja negra.
- Caja propia y distinta para los pensamientos. Esto tiene una consecuencia buena: si los pensamientos tienen forma reconocible, **su ausencia también se lee**, y las escenas donde Mel deja de pensar (la terminal, por ejemplo) comunican solas.
- Indicador de que la conversación sigue. Las hojas apiladas con tope de 3 comunican "quedan más de dos", no cuántas. Decidir antes si se quiere que el jugador sepa cuánto falta: mostrar el largo convierte la charla en barra de progreso.

### 0.5.1 — UI de celular desde cero

Replantear el concepto, no solo el aspecto. Cosas que ya quedaron claras y conviene resolver acá:
- Si el campo de escribir existe, que haga algo. La opción elegida escribiéndose sola letra por letra antes de enviarse es el lugar más barato del juego para mostrar a Mel dudando.
  - Acá entra la mecánica de **escribir y borrar**: Mel escribe algo honesto, lo mira, lo borra, y manda otra cosa.
- Si los colores de burbuja identifican a cada contacto, que escale: el chat de los padres con paleta propia es información gratis.
- Contraste: el amarillo claro sobre crema del mockup se lee mal.
- Notificaciones con jerarquía, para que el correo basura no entrene al jugador a ignorar avisos.

### 0.5.2 — Billetera y plata finita

El cambio más grande del bloque, y **no es un cambio de UI**. Antes de ponerse hay que decidir en [[Diseño de gameplay]]:
- Con cuánta plata arranca y quién se la dio.
- En qué se gasta.
- Quién paga en el café.
- Si el jugador puede quedarse sin plata para algo necesario, y si el juego protege contra eso.
- Si al desaparecer el inventario Mel deja de cargar objetos, y dónde vive el celular.

Propuesta a evaluar: en vez de una cantidad de plata, **billetes**. "Tenés dos", y cada cosa cuesta uno. No hay precios que mostrar ni que envejezcan, y se lee de un vistazo. Si Mel solo carga billetera y celular, es una simplificación sana para esta historia.

---

## 0.6 — Mundo habitado

### 0.6.0 — Sistema de NPCs

Diálogos cortos y repetibles para la gente del pueblo, que varíen según el momento del día y según lo que ya pasó. Es un sistema distinto del de conversaciones guionadas y es lo que le da al tramo el "algo para hacer" que faltaba.

Puede usar el mismo formato que `scr_objects` (`base` + una clave por momento).

### 0.6.1 — Audio

- Que el audio general baje en pantallas seleccionadas (celular, billetera, pausa). Es la diferencia entre "abrí un menú" y "Mel se metió en el teléfono y dejó de escuchar lo que pasa alrededor". Se apoya en los volúmenes por categoría que ya existen.
- Revisar si el ambiente responde a la hora del día.

---

## 0.7 — EXPERIMENTO: Cámara y profundidad

### 0.7.0 — Parallax

Atado al depth. Detalle de producción al dibujar: las capas de **primer plano** se desplazan más rápido que la cámara, así que necesitan más ancho que la room — no alcanza con dibujar al tamaño del escenario.

### 0.7.1 — Blur de primer plano

Entra solo si pasa la prueba. La sospecha: el borde duro es lo que sostiene el estilo vectorial, y desenfocarlo puede leerse como filtro antes que como profundidad de campo. Probarlo junto con la textura de papel, porque la combinación —textura nítida sobre dibujo borroso— es exactamente la que hay que ver antes de decidir.

---

## 0.8 — La textura de papel

Reemplaza la idea anterior de textura global fija a pantalla. **Ojo: cambia lo que ya está hecho** — hoy `obj_paper` es un solo papel pegado a la pantalla para todo. **Prototipar primero en un proyecto vacío aparte**, no en Mel: un experimento a medio salir metido en el repo es lo que después nadie sabe si está en uso.

### 0.8.0 — El efecto

La idea es la de Chowder: el elemento se mueve y la textura no, así que el pelo de Mel va cambiando de papel según por dónde pasa.

- Nada de identificar los elementos por color exacto: el arte es vectorial, los bordes tienen píxeles intermedios por el antialiasing, y esos quedarían sin texturar. Además dos cosas del mismo color se texturarían igual. La textura tiene que seguir la silueta del dibujo, no su color.
- *Implicancia técnica:* es un shader que se aplica al dibujar cada cosa, así que **todo tiene que dibujarse a través de una misma función**.
- El anclaje conviene que sea **regulable**, no fijo. Pegada a la pantalla, la textura se mueve sobre el escenario y queda quieta sobre Mel (porque la cámara la sigue). Pegada al mundo, pasa lo contrario: se mueve sobre Mel y queda quieta sobre el escenario. En el medio se mueve sobre todo. No vale la pena decidirlo en abstracto, se regula mirando.
- Ojo con que la textura no "hormiguee" mientras la cámara se mueve. Es el síntoma a vigilar. Suele venir de que la cámara está en posiciones con decimales: redondear la cámara lo resuelve casi siempre.

### 0.8.1 — Asignación y prueba

- Qué elemento lleva qué papel (servilleta, rayado, cuadriculado, afiche).
- Decidir si los fondos también llevan textura. Si solo la llevan los personajes, van a parecer pegados encima.
- **Probar caminando, no con una captura.** Quieto se ve hermoso siempre; el efecto real solo se juzga con la cámara en movimiento.
- Tileado, si hace falta: el rayado y el cuadriculado empalman bien si el alto del tile es múltiplo exacto del espaciado de las líneas. El grano (servilleta, afiche) es el más fácil de todos.

---

## 0.9 — Integración y limpieza

- Pasada de consistencia con todo junto: un mismo escenario de prueba, de día y de noche, con personajes, UI, textura y parallax funcionando a la vez.
- Borrar lo que haya quedado sin uso.
- Actualizar la wiki con lo que cambió.

---

## prealpha 1.0

Motor congelado. A partir de acá, contenido.

---

## Fuera de esta tanda

- **Hora del día en tiempo real.** Cambio grande y riesgoso: pelea con que el jugador se tome su tiempo para husmear el celular y con que Lu guíe el día (si el reloj corre de verdad, Lu espera parada mientras el jugador lee). La pregunta a responder antes es qué se quiere que el reloj le haga al jugador: si es presión, el tiempo real la da; si es que el día pase y la luz cambie, el tiempo por acciones alcanza y además deja controlar al minuto cuándo cae el atardecer en el puente.
- **Minijuegos.** Cada uno es su propia cosa y se hace cuando el contenido lo pida, no antes. El del vóley va a tocar los controles del player, así que probablemente arrastre el punto 0.2.3.
- **Lu caminando junto a Mel** entre lugares (seguir al jugador, o guiarlo).
- **El menú de temas** de las charlas (elegir de qué hablar, con lo no preguntado quedando sin saberse).
- **Las fotos**: sacarlas en cada lugar y elegir cuáles mandarles a los papás.
- **Efectos del trance**, sobre el post-proceso de 0.3.0.

---

## Decisiones pendientes que el plan necesita

- Qué banderas son historia y cuáles estado de sesión (ver [[Auditoría de banderas]]).
- Cuánto tinte recibe la UI, y cómo se le aplica.
- Si los fondos llevan textura o solo los elementos seleccionados.
- Si el blur entra.
- Si se muestra o no cuánto falta de una conversación.
- Toda la economía de la billetera (ver 0.5.2), incluida la propuesta de billetes.

---

## Nota de versionado

Un bloque, una rama; se mergea cuando el juego corre. El rework de base de personajes y el de paletas tocan todo el arte a la vez, así que conviene que vivan en su propia rama hasta estar completos.
