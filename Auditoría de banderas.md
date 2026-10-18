Auditoría para el punto 0.2.0 de [[Plan de desarrollo alpha 1.0]]: qué hay hoy en el estado del juego, qué es **historia** (se guarda) y qué es **sesión** (no se guarda nunca, arranca limpio).

Revisado sobre el commit `prealpha-indev 0.1.0` (14/10/2026).

---

## Banderas (`global.flags`)

| Bandera | Dónde se escribe | Tipo | Qué hacer |
| --- | --- | --- | --- |
| `lock_move` | cutscenes `micro_ida`, `micro_llegada`, `debug_micro_base` (paso `flag`) | **Sesión** | Sacarla de las banderas. Es exactamente el caso del plan: si un autosave cae entre el despertar y la llegada, la partida carga con Mel trabada para siempre. Pasa a ser una variable de sesión (en `obj_player` o en un `global.session`), y el momento la reconstruye al cargar. |
| `micro_vio_celular` | `scr_dialogue_actions`, acción `celular_abierto` | Historia | Se queda. Evita que el pensamiento se repita después de cargar. |
| `micro_vio_mensajes` | `scr_dialogue_actions`, acción `mensajes_abierto` | Historia | Se queda. Ídem. |
| `micro_resp_lu` (1 / 2) | `scr_dialogue_actions`, acciones `micro_resp_lu_*` | **Historia (decisión)** | Se queda. Es una elección del jugador. |
| `micro_resp_fam` (1 / 2) | `scr_dialogue_actions`, acciones `micro_resp_fam_*` | **Historia (decisión)** | Se queda. Ídem. |
| `papas_respondido` | — (solo se lee en el Step de `obj_player`) | Resto del depto | Borrar la línea `_must_answer` del Step de `obj_player`: mira `tarde_calma` y `papas_respondido`, dos cosas que ya no existen. |

**Convención propuesta:** las banderas son solo historia. Si algo tiene que durar lo que dura una escena, no es una bandera.

---

## Otro estado que hoy se guarda

| Qué | Tipo | Problema | Qué hacer |
| --- | --- | --- | --- |
| `chats[].typing` | **Sesión** | Si se guarda en `true`, el "..." queda escribiendo para siempre. | Ponerlo en `false` al guardar o al cargar. |
| `chats[].pending` | Historia, pero frágil | Las respuestas pendientes se guardan, pero el **guion de chat que las puso no**: si venían de un `reply` de un guion (`obj_chat_controller`), al contestar después de cargar, el resto del guion no sigue. Las del micro andan porque su efecto va en la `action` de la opción. | Regla: las respuestas pendientes siempre llevan su consecuencia en la `action`, nunca dependen de que el guion siga corriendo. |
| `chats[].avatar` | Recurso | Ya resuelto: se reasigna por id al cargar. | — |
| `room_name` | Historia | Ya resuelto. | — |

## Lo que no se guarda (y está bien que no)

Guiones de chat en curso (`obj_chat_controller.runners`), acciones programadas (`obj_scheduler`), cutscenes, UIs abiertas, notificaciones, efectos, tinte y fundido de pantalla, audio. Todo eso es sesión: lo tiene que reconstruir el momento al cargar.

---

## Hallazgos de paso

- **Hoy el juego no guarda en ningún punto.** El paso `save` existe en `obj_cutscene_controller`, pero ninguna cutscene lo usa desde que se borró el departamento. Un "Continuar" carga una partida vieja, de otra versión del juego. Hasta definir el primer punto de autosave del juego nuevo, conviene borrar el save al arrancar una partida nueva (ya lo hace "Reiniciar") o no ofrecer "Continuar".
- **`wait_flag`** está implementado pero no se usa en ninguna cutscene. Puede quedar.
- **`instance_create_layer(..., "Instances", ...)`** sigue en 10 lugares (`obj_game_controller`, `obj_ui_controller`, `obj_door`, `obj_menu_controller`, `scr_dialogues`, `scr_save`, `scr_cutscenes`). Funciona mientras cada room tenga una capa llamada exactamente `Instances`; si alguna no la tiene, explota. Reemplazar por `instance_create_depth(0, 0, 0, ...)`.

---

## Orden sugerido para cerrar 0.2.0

1. Mover `lock_move` a sesión y borrar el `_must_answer` del depto.
2. Limpiar `typing` al guardar.
3. Que cargar use el mismo camino que el debug: el momento guardado arranca con su `debug_<momento>` (o su entrada normal si no tiene).
4. Elegir el primer punto de autosave del juego nuevo (candidato natural: al llegar a la terminal, antes del blanco).
5. Probar con el ejecutable cerrado.
