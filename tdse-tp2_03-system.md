# Resultados de Depuración - Task System

Luego de la integración del diagrama de estados y varias ejecuciones de la función `app_update()`, se pausó la depuración para analizar el rendimiento de la tarea del sistema.

*(Nota: De acuerdo con la inicialización del arreglo `task_cfg_list` en el archivo `app.c`, la tarea `task_system` se encuentra en el índice 1, no en el 0 como sugiere la guía).*

Los valores de rendimiento medidos por el DWT (Data Watchpoint and Trace) para `task_system` (`task_dta_list[1]`) son los siguientes:

* **NOE (Number of Executions):** 21742 (adimensional, cantidad de veces que se ejecutó la máquina de estados del sistema).
* **LET (Last Execution Time):** 3 µs (microsegundos).
* **BCET (Best-Case Execution Time):** 3 µs (microsegundos).
* **WCET (Worst-Case Execution Time):** 3 µs (microsegundos).

**Análisis de los resultados:**
La depuración confirma el correcto funcionamiento del modelo System. El procesador tarda consistentemente solo 3 microsegundos en acceder a los datos, evaluar si existen eventos en la cola FIFO, procesar el bloque `switch-case` de la máquina de estados de la barrera de estacionamiento y salir. La coincidencia entre el mejor caso (BCET) y el peor caso (WCET) demuestra que el código de la capa de control central es altamente determinístico y 100% no-bloqueante.
