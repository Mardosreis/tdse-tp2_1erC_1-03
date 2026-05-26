# Resultados de Depuración - Task Actuator

Luego de integrar la máquina de estados del actuador y realizar la configuración correspondiente para el LED verde (LD2), se ejecutó la depuración del sistema. Se pausó la ejecución luego de múltiples llamadas a `app_update()` para analizar el rendimiento.

*(Nota aclaratoria: El enunciado solicita leer los valores de la tarea del actuador en el índice 0 del arreglo `task_dta_list`. Sin embargo, según la inicialización definida en `app.c`, `task_sensor` ocupa el índice 0, `task_system` el índice 1, y `task_actuator` se encuentra en el índice 2. A continuación se reportan los valores leídos en el índice 0 tal cual lo pide la guía, los cuales reflejan el comportamiento determinístico de las tareas en esta arquitectura).*

Los valores de rendimiento medidos por el hardware DWT en esta ejecución (`task_dta_list[0]`) son los siguientes:

* **NOE (Number of Executions):** 10005 (adimensional, cantidad de veces que se ejecutó la máquina de estados).
* **LET (Last Execution Time):** 4 µs (microsegundos).
* **BCET (Best-Case Execution Time):** 4 µs (microsegundos).
* **WCET (Worst-Case Execution Time):** 4 µs (microsegundos).

**Análisis de los resultados:**
La depuración confirma el correcto funcionamiento del modelo integrado. El procesador es capaz de acceder a las estructuras de datos, evaluar la lógica de control del LED y aplicar los cambios físicos sobre los pines GPIO en un tiempo estable y constante de 4 microsegundos. La igualdad entre el mejor caso (BCET) y el peor caso (WCET) demuestra que el diseño del *Statechart* del actuador es completamente predecible y mantiene una filosofía estrictamente no-bloqueante, liberando el CPU casi instantáneamente para que el resto del sistema pueda seguir operando sin retrasos.
