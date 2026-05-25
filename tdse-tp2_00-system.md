# Análisis y Explicación del Código Fuente del Sistema y Actuador

A continuación se detalla el funcionamiento de los archivos proporcionados, los cuales conforman la capa de la aplicación (sistema central) y su interfaz de comunicación con los actuadores (ej. LEDs).

## 1. Funcionamiento de los archivos adjuntos

* **`task_system_attribute.h`**: Define las estructuras de datos requeridas para la tarea principal del sistema. Declara los eventos lógicos (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`), los estados del sistema (`ST_SYS_IDLE`, `ST_SYS_ACTIVE`) y la estructura dinámica `task_system_dta_t` que almacena el estado, evento, un temporizador (tick) y una bandera (flag).
* **`task_actuator_attribute.h`**: Contiene las definiciones para la tarea del actuador. Define eventos (`EV_LED_IDLE`, `EV_LED_ACTIVE`), estados (`ST_LED_IDLE`, `ST_LED_ACTIVE`), identificadores (como `ID_LED_A`) y las estructuras de configuración (`task_actuator_cfg_t`) y datos dinámicos (`task_actuator_dta_t`).
* **`task_system.c`**: Implementa la lógica central de la aplicación. Posee una máquina de estados que consume los eventos generados por los sensores (mediante una cola) y, dependiendo de su estado actual, procesa estos eventos para enviar comandos de acción hacia la tarea del actuador.
* **`task_system_interface.c`**: Define la implementación de la cola circular (FIFO) mediante la cual otras tareas (como el sensor) le envían eventos a la tarea del sistema (`task_system`) de forma asíncrona.
* **`task_actuator_interface.c`**: Provee la función `put_event_task_actuator`, la cual es una interfaz directa (sin cola) para que el sistema le notifique nuevos eventos al actuador modificando directamente sus variables de estado.

---

## 2. Evolución de las variables en `task_system`

### Durante el inicio (`task_system_init()`)
* **`index`**: Se utiliza en un bucle `for` para inicializar el arreglo de datos del sistema. Como `SYSTEM_DTA_QTY` es igual a `MODE_QTY` (que vale 1 por tener solo el modo `NORMAL`), iterará únicamente con el valor `0`.
* **`task_system_dta_list[index].tick`**: No se inicializa explícitamente en esta función, quedando con valor indefinido o cero. Su unidad de medida son **ticks del sistema** (donde 1 tick equivale a 1 milisegundo, según se indica en `p_task_system__`).
* **`task_system_dta_list[index].state`**: Se inicializa en `ST_SYS_IDLE`.
* **`task_system_dta_list[index].event`**: Se inicializa en `EV_SYS_IDLE`.
* **`task_system_dta_list[index].flag`**: Se inicializa en `false`.

### Durante la ejecución cíclica (`task_system_update()`)
En el bucle principal, si el modo global es `NORMAL`, se invoca la máquina de estados. *(Nota: En `task_system_update`, no se usa un iterador `index`, sino que la máquina de estados accede directamente al índice `NORMAL` / 0)*.
* **`task_system_dta_list[NORMAL].flag`**: Cambia a `true` únicamente si la función `any_event_task_system()` detecta que hay un evento esperando en la cola. Si ocurre una transición de estado exitosa, la misma máquina de estados la vuelve a poner en `false`.
* **`task_system_dta_list[NORMAL].event`**: Si había un evento en la cola, toma el valor extraído mediante `get_event_task_system()` (`EV_SYS_ACTIVE` o `EV_SYS_IDLE`).
* **`task_system_dta_list[NORMAL].state`**: Transiciona a `ST_SYS_ACTIVE` si estaba en `IDLE` y recibe un evento `EV_SYS_ACTIVE`. Transiciona de vuelta a `ST_SYS_IDLE` si estaba en `ACTIVE` y recibe un `EV_SYS_IDLE`.
* **`task_system_dta_list[NORMAL].tick`**: No se utiliza ni se modifica en la operación normal de esta máquina de estados, salvo que ocurra un error y caiga en el caso `default`, donde se reinicia a `DEL_SYS_MIN` (0).

---

## 3. Comportamiento de la máquina de estados del sistema

*(Nota: En el código provisto, la función equivalente a la solicitada se llama `void task_system_normal_statechart(void)` y no recibe el parámetro `index`)*.

Su comportamiento es el de un orquestador (controlador principal):
1.  **Lectura de eventos:** Consulta si hay eventos pendientes en la cola del sistema. Si los hay, levanta su variable `flag` y consume el evento.
2.  **Evaluación y Transición:** Utiliza un bloque `switch` evaluando el estado actual (`ST_SYS_IDLE` o `ST_SYS_ACTIVE`). Si se levantó el `flag` y el evento entrante se corresponde con la transición esperada, realiza el cambio de estado.
3.  **Disparo de acciones:** Durante la transición, apaga su propio `flag` (marcando el evento como procesado) e invoca a `put_event_task_actuator()` para enviarle el comando correspondiente (`EV_LED_ACTIVE` o `EV_LED_IDLE`) al identificador físico del actuador (`ID_LED_A`).

---

## 4. Evolución de las variables de la cola `event_task_system_queue`

### Durante el inicio (`init_event_task_system()`)
* **`i`**: Es una variable local utilizada en un bucle `for` que itera desde 0 hasta 15 (ya que `QUEUE_LENGTH` es 16).
* **`event_task_system_queue.head`** (índice de escritura): Se inicializa en `0`.
* **`event_task_system_queue.tail`** (índice de lectura): Se inicializa en `0`.
* **`event_task_system_queue.count`**: Se inicializa en `0`.
* **`event_task_system_queue.queue[i]`**: Se rellena en sus 16 posiciones con el valor `EMPTY` (255).

### Durante la ejecución cíclica (`task_system_update()`)
Cuando la máquina de estados llama a `get_event_task_system()`:
* **`event_task_system_queue.count`**: Se decrementa en 1.
* **`event_task_system_queue.queue[tail]`**: Se lee el evento en la posición apuntada por `tail` y, a continuación, esa celda se sobrescribe con el valor `EMPTY` (255).
* **`event_task_system_queue.tail`**: Se incrementa en 1, apuntando a la próxima celda de lectura. Si alcanza el tamaño máximo de la cola (16), se reinicia a 0 (comportamiento circular).
*(Por otro lado, `head` solo cambia cuando el sensor inserta eventos usando `put_event_task_system()`)*.

---

## 5. Evolución de variables en la interfaz del Actuador (`task_actuator_interface.c`)

### Durante el inicio (`task_system_init()`)
La función de inicialización del sistema no realiza llamadas a los actuadores, por lo que las variables `identifier`, `event` y `flag` del actuador no son modificadas en esta etapa por este módulo.

### Durante la ejecución cíclica (`task_system_update()`)
Cuando la máquina de estados del sistema (`task_system_normal_statechart()`) ejecuta un cambio de estado válido, invoca la función `put_event_task_actuator(event, identifier)`. En ese momento:
* **`identifier`**: Toma el valor que el sistema le pasa como argumento, que en este código está *hardcodeado* como `ID_LED_A`.
* **`task_actuator_dta_list[identifier].event`**: Recibe y almacena directamente el evento comandado por el sistema (`EV_LED_ACTIVE` o `EV_LED_IDLE`), sobrescribiendo cualquier evento anterior.
* **`task_actuator_dta_list[identifier].flag`**: Cambia a `true`. Esto funciona como un semáforo para que, cuando la tarea del actuador se ejecute en el bucle principal, sepa que tiene una orden nueva que procesar.
