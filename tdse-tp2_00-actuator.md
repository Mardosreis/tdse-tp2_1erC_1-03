# Análisis y Explicación del Código Fuente del Actuador

A continuación, se detalla el funcionamiento de los archivos proporcionados, los cuales son responsables de la capa de actuación del sistema (por ejemplo, encender o apagar un LED físico en la placa).

## 1. Funcionamiento de los archivos adjuntos

* **`task_actuator_attribute.h`**: Este archivo de cabecera define todos los atributos necesarios para la tarea del actuador. Establece los identificadores (como `ID_LED_A`), los estados lógicos (`ST_LED_IDLE`, `ST_LED_ACTIVE`), los eventos de estímulo (`EV_LED_IDLE`, `EV_LED_ACTIVE`) y las estructuras de datos, tanto de configuración física del hardware (`task_actuator_cfg_t`) como de las variables dinámicas de ejecución (`task_actuator_dta_t`).
* **`task_actuator.c`**: Contiene la implementación lógica de la tarea. Se encarga de inicializar el estado del actuador, apagar físicamente el hardware en el arranque y ejecutar continuamente una Máquina de Estados Finitos (FSM) que reacciona a los eventos externos para cambiar el estado físico de los pines del microcontrolador.
* **`task_actuator_interface.c`**: Proporciona el mecanismo de comunicación (API) mediante el cual otras tareas (como la tarea del sistema) pueden enviar órdenes o eventos al actuador de forma directa, modificando sus variables en memoria sin utilizar una cola de mensajes.

---

## 2. Evolución de variables internas de la tarea actuador

### Durante el inicio (`task_actuator_init()`)
Al arrancar el sistema:
* **`index`**: Se inicializa en 0 dentro de un bucle `for` y recorre la cantidad de actuadores configurados (`ACTUATOR_DTA_QTY`). Como solo hay configurado un LED, itera únicamente para el valor 0.
* **`task_actuator_dta_list[index].tick`**: Esta variable no se inicializa de forma explícita en el bloque de inicio de esta función. Su unidad de medida son **ticks del sistema** (donde 1 tick equivale a 1 milisegundo, según lo documentado en `p_task_actuator__`).
* **`task_actuator_dta_list[index].state`**: Se inicializa forzosamente en el estado de reposo `ST_LED_IDLE`.
* **`task_actuator_dta_list[index].event`**: Se inicializa con el evento por defecto `EV_LED_IDLE`.
* **`task_actuator_dta_list[index].flag`**: Se inicializa en `false`, indicando que no hay eventos nuevos por procesar. Además, la función apaga físicamente el pin del actuador en el hardware.

### Durante la ejecución cíclica (`task_actuator_update()`)
En cada pasada del bucle principal:
* **`index`**: Vuelve a iterar desde 0 hasta el final del arreglo de actuadores para ejecutar la máquina de estados de cada uno de ellos mediante `task_actuator_statechart(index)`.
* **`task_actuator_dta_list[index].tick`**: Permanece inalterada durante el funcionamiento normal de la máquina de estados. Únicamente tomaría el valor `DEL_LED_MIN` (0) si la máquina de estados cayera en el caso `default` (condición de error).
* **`task_actuator_dta_list[index].event`**: No se modifica desde dentro de esta tarea; mantiene el último evento que le haya sido inyectado desde el exterior.
* **`task_actuator_dta_list[index].flag`**: Si al iniciar el ciclo está en `true` y el evento asociado provoca una transición válida en la máquina de estados, la propia máquina lo vuelve a cambiar a `false` inmediatamente después de aceptarlo.
* **`task_actuator_dta_list[index].state`**: Cambia de `ST_LED_IDLE` a `ST_LED_ACTIVE` si llega un evento de activación, y de `ST_LED_ACTIVE` a `ST_LED_IDLE` si llega un evento de desactivación.

---

## 3. Comportamiento de la función `task_actuator_statechart(uint32_t index)`

Esta función implementa una máquina de estados sencilla (sin estados transitorios temporizados, a diferencia del sensor) que reacciona a los comandos del sistema:

1.  **Lectura de punteros:** Obtiene las referencias directas a la configuración (puertos, pines) y a los datos dinámicos del actuador específico indicado por el `index`.
2.  **Evaluación (`switch`)**:
    * Si el estado actual es `ST_LED_IDLE`, verifica si existe un evento nuevo no procesado (`flag == true`) y si ese evento es `EV_LED_ACTIVE`. De cumplirse ambas, limpia la bandera (`flag = false`), escribe el valor lógico de encendido (`led_on`) en el pin GPIO físico correspondiente y cambia su estado a `ST_LED_ACTIVE`.
    * Si el estado actual es `ST_LED_ACTIVE`, verifica si hay un evento nuevo (`flag == true`) del tipo `EV_LED_IDLE`. Si es así, limpia la bandera (`flag = false`), escribe el valor lógico de apagado (`led_off`) en el pin GPIO físico y vuelve al estado `ST_LED_IDLE`.
3.  **Seguridad (`default`)**: Si por algún error de memoria o corrupción la variable de estado tomara un valor no contemplado, restablece todas las variables a su estado seguro y neutral (`ST_LED_IDLE` y `flag` en falso).

---

## 4. Evolución de variables a través de la interfaz (`task_actuator_interface.c`)

### Durante el inicio y bucle natural (`task_actuator_init()` y `task_actuator_update()`)
La función `put_event_task_actuator` de la interfaz no es invocada por la propia tarea del actuador ni en su inicialización ni en su bucle de actualización, por lo que estas variables no evolucionan por sí solas en estos momentos.

### Cuando ocurre una llamada externa (Ej. desde la tarea del sistema)
Cuando la tarea lógica del sistema decide enviar un comando, llama a `put_event_task_actuator(event, identifier)`. En ese preciso instante:
* **`identifier`**: Es un parámetro local de la función que recibe el identificador del hardware que se desea comandar (por ejemplo, `ID_LED_A`, que corresponde al índice numérico del arreglo).
* **`task_actuator_dta_list[identifier].event`**: Se actualiza inmediatamente sobrescribiéndose con el nuevo evento recibido por parámetro (`EV_LED_ACTIVE` o `EV_LED_IDLE`).
* **`task_actuator_dta_list[identifier].flag`**: Se pone explícitamente en `true`. Esta es la señal de sincronización asíncrona que alerta a la máquina de estados (`statechart`), la próxima vez que se ejecute en el bucle principal, de que tiene una orden pendiente por leer y procesar físicamente.
