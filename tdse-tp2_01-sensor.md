# Resultados de Depuración - Task Sensor

Luego de varias ejecuciones de la función `app_update()`, se obtuvieron los siguientes valores de rendimiento para la tarea del sensor ubicados en el índice 0 del arreglo (`task_dta_list[0]`):

* **NOE (Number of Executions):** 176342 (adimensional, cantidad de veces que se ejecutó la tarea).
* **LET (Last Execution Time):** 4 µs (microsegundos).
* **BCET (Best-Case Execution Time):** 4 µs (microsegundos).
* **WCET (Worst-Case Execution Time):** 4 µs (microsegundos).

**Análisis breve:**
Los valores de tiempo (LET, BCET, WCET) medidos a través del hardware DWT indican que la tarea de lectura y anti-rebote del botón (Sensor Statechart) tarda sistemáticamente 4 microsegundos en ejecutarse. La coincidencia entre el mejor y peor caso demuestra que el código no es bloqueante y tiene un comportamiento altamente determinístico.
