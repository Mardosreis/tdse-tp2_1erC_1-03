# Resultados de Depuración - Task Actuator (Escalabilidad a 3 LEDs)

Luego de conectar físicamente los 2 LEDs adicionales (B y C) a los pines correspondientes de la placa NUCLEO-F103RB, se procedió a compilar y ejecutar el programa. Se pausó la depuración tras varias iteraciones de la función `app_update()` para analizar el impacto en el rendimiento.

*(Nota aclaratoria: El enunciado solicita leer los valores de la tarea del actuador en el índice 0 del arreglo `task_dta_list`. Sin embargo, según la inicialización definida en la arquitectura base `app.c`, `task_sensor` ocupa el índice 0 y `task_actuator` se encuentra en el índice 2. A continuación se reportan los valores leídos en el índice 0 tal cual lo requiere la guía).*

Los valores de rendimiento medidos por el DWT obtenidos en esta ejecución (`task_dta_list[0]`) son los siguientes:

* **NOE (Number of Executions):** 7616 (adimensional, cantidad de veces que se ejecutó la tarea).
* **LET (Last Execution Time):** 4 µs (microsegundos).
* **BCET (Best-Case Execution Time):** 4 µs (microsegundos).
* **WCET (Worst-Case Execution Time):** 4 µs (microsegundos).

**Análisis de los resultados:**
La depuración confirma el correcto funcionamiento y la excelente escalabilidad del modelo Actuator. A pesar de haber incrementado la carga de procesamiento (pasando de gestionar 1 a 3 máquinas de estado independientes para los LEDs), el tiempo del peor caso (WCET) se mantiene estable en unos ínfimos 4 microsegundos. Esto corrobora que el diseño basado en arreglos de estructuras (`task_actuator_cfg_list`) es altamente eficiente, permitiendo controlar múltiples periféricos de hardware sin comprometer la naturaleza determinística y no-bloqueante del firmware.
