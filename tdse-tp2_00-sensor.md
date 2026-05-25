Análisis y Explicación del Código Fuente
A continuación, se detalla el funcionamiento de los archivos adjuntos que conforman la lógica de lectura de sensores (anti-rebote) y la comunicación entre tareas mediante una cola de eventos.

1. Funcionamiento de los archivos adjuntos
task_sensor_attribute.h: Este archivo de cabecera define las estructuras de datos y los tipos enumerados necesarios para la tarea encargada de leer el sensor (un botón). Define los eventos de hardware (EV_BTN_UP, EV_BTN_DOWN), los estados de la máquina de estados anti-rebote (ST_BTN_UP, ST_BTN_FALLING, ST_BTN_DOWN, ST_BTN_RISING), y las estructuras de configuración (task_sensor_cfg_t) y de datos dinámicos (task_sensor_dta_t).

task_system_attribute.h: Define los atributos para la tarea de sistema ("Task System"). Incluye los eventos lógicos del sistema (EV_SYS_IDLE, EV_SYS_ACTIVE), sus estados y su estructura de datos dinámica (task_system_dta_t).

task_sensor.c: Contiene la implementación de la tarea del sensor. Inicializa un arreglo de configuración para un botón (BTN_A) y ejecuta una máquina de estados periódica que lee el estado del pin GPIO, elimina los rebotes mecánicos y, cuando se detecta un pulso válido, envía un evento de sistema (EV_SYS_ACTIVE o EV_SYS_IDLE) a la tarea principal.

task_system_interface.c: Implementa una interfaz de comunicación entre tareas utilizando una estructura de datos de tipo cola circular (FIFO). Esta cola permite que la tarea del sensor (task_sensor) almacene los eventos válidos para que la tarea del sistema (task_system) los procese de manera asíncrona.

2. Evolución de las variables en task_sensor
Durante el inicio (task_sensor_init())
Al ejecutar la función de inicialización, las variables se establecen de la siguiente manera:

index: Se utiliza como variable de iteración en un bucle for para recorrer todos los sensores configurados (en este caso, solo 1, por lo que itera de 0 a SENSOR_DTA_QTY - 1).

task_sensor_dta_list[index].tick: No se inicializa de forma explícita en esta función de arranque inicial, por lo que su valor permanece indefinido (o 0 si se asume por ser una variable global bss) hasta que entra en funcionamiento la máquina de estados. Su unidad de medida son ticks del sistema (donde 1 tick = 1 milisegundo, según el texto definido en p_task_sensor__).

task_sensor_dta_list[index].state: Se inicializa en el estado de reposo ST_BTN_UP.

task_sensor_dta_list[index].event: Se inicializa en el evento por defecto EV_BTN_UP.

Durante la ejecución cíclica (task_sensor_update())
En cada llamada al bucle principal:

index: Vuelve a iterar sobre los sensores configurados, delegando el procesamiento a task_sensor_statechart(index).

task_sensor_dta_list[index].event: En cada iteración, se actualiza inmediatamente leyendo el nivel lógico del pin (HAL_GPIO_ReadPin). Si el botón está presionado, cambia a EV_BTN_DOWN; de lo contrario, cambia a EV_BTN_UP.

task_sensor_dta_list[index].state y tick: Su evolución depende exclusivamente de los cambios en event manejados por la máquina de estados. Por ejemplo, al detectar un EV_BTN_DOWN en estado ST_BTN_UP, el state cambia a ST_BTN_FALLING y tick se carga con el valor máximo de retardo configurado (DEL_BTN_MAX o 50 milisegundos). En sucesivas llamadas a update, tick irá decrementándose en 1 por cada milisegundo hasta llegar a 0, confirmando o cancelando la transición al nuevo estado estable.

3. Comportamiento de void task_sensor_statechart(uint32_t index)
Esta función es el núcleo lógico del sensor y funciona como una Máquina de Estados Finitos (FSM) diseñada para implementar un algoritmo anti-rebote (debounce) por software:

Lectura Inicial: Comienza comparando el estado actual del pin físico (gpio_port y pin) con la configuración de presionado (pressed). Asigna un evento base interno: EV_BTN_DOWN o EV_BTN_UP.

Transitorios (Rebotes): Si la máquina se encuentra en un estado estable (ST_BTN_UP o ST_BTN_DOWN) y el evento base detecta un cambio, la máquina pasa a los estados de transición (ST_BTN_FALLING o ST_BTN_RISING, respectivamente) y se inicializa la variable temporizadora tick.

Filtrado temporal: Durante los estados de transición (FALLING/RISING), si el botón mantiene el estado durante el tiempo estipulado (mientras tick > 0 se decrementa cíclicamente), se asume que el contacto eléctrico es firme. Si en medio del conteo el pin regresa al valor original, la transición se aborta.

Emisión de eventos: Cuando tick llega a 0 y el cambio es válido, la FSM transiciona al nuevo estado estable (ST_BTN_DOWN o ST_BTN_UP) y se invoca la función put_event_task_system() para encolar una señal de sistema (signal_down o signal_up, configuradas como EV_SYS_ACTIVE o EV_SYS_IDLE).

4. Evolución de las variables de la cola event_task_system_queue
La estructura event_task_system_queue funciona como una cola circular estándar en C, inicializada indirectamente (probablemente antes del inicio del bucle) mediante init_event_task_system().

Estado inicial
head (índice de escritura): Comienza en 0.

tail (índice de lectura): Comienza en 0.

count (cantidad de elementos): Comienza en 0.

queue[i] (el arreglo): Todas sus posiciones (de 0 a 15) se rellenan con el valor EMPTY (255).

Evolución en las sucesivas ejecuciones
Cuando la máquina de estados en task_sensor_update() confirma un pulsado o soltado válido, invoca a put_event_task_system(event).

Al hacerlo, count se incrementa en 1.

Se escribe el evento emitido en el arreglo en la posición actual de head: queue[head] = event.

head se incrementa en 1. Si head alcanza el tamaño máximo de la cola (QUEUE_LENGTH), vuelve inmediatamente a 0, creando el comportamiento de anillo circular.

Eventualmente, otra tarea (la de sistema) invocará get_event_task_system() para leer estos eventos.

En ese instante, count se decrementa en 1.

Se lee el evento ubicado en la posición actual de tail, y esa posición en queue[tail] se sobrescribe con EMPTY.

tail se incrementa en 1. Al igual que head, si alcanza QUEUE_LENGTH, reinicia su valor a 0.
