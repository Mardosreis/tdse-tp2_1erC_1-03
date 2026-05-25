# Resultados de Depuración - Task Sensor (3 Botones)

Luego de conectar físicamente los 3 botones a la placa NUCLEO-F103RB y ejecutar el programa, se pausó la depuración tras varias iteraciones de la función `app_update()`. 

Los valores de rendimiento almacenados en el índice 0 del arreglo (correspondiente a `task_sensor` en `task_dta_list[0]`) son los siguientes:

* **NOE (Number of Executions):** 4016 (adimensional, cantidad de veces que se ejecutó la tarea).
* **LET (Last Execution Time):** 9 µs (microsegundos).
* **BCET (Best-Case Execution Time):** 9 µs (microsegundos).
* **WCET (Worst-Case Execution Time):** 11 µs (microsegundos).

**Análisis de los resultados:**
Al comparar estos tiempos con el modelo anterior de un solo sensor, se observa un incremento lógico en el tiempo de ejecución (de ~4 µs a un rango de 9-11 µs). Esto confirma el correcto funcionamiento de la escalabilidad del código: la función `task_sensor_update()` ahora ejecuta su bucle `for` iterando 3 veces por cada ciclo principal para procesar las máquinas de estado independientes (anti-rebote) de `BTN_B`, `BTN_C` y `BTN_D`. El código mantiene su característica de ser no-bloqueante y con un tiempo de ejecución determinístico muy bajo.
