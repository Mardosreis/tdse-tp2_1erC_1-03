Análisis del Funcionamiento de los Archivos
1. startup_stm32f103rbtx.s (y system_stm32f1xx.c)
El archivo de startup (escrito en ensamblador) contiene el Reset_Handler, que es el punto de entrada del microcontrolador al recibir energía. Su función principal es inicializar la memoria (copiando variables a la RAM) y llamar a la función SystemInit(), la cual se encuentra en el archivo system_stm32f1xx.c. Esta función establece la configuración base de la memoria Flash y asigna la dirección de la tabla de vectores de interrupción. Una vez configurado lo básico, el código ensamblador realiza el salto a la función main() escrita en C.

2. main.c
Este es el núcleo de tu aplicación. El flujo de ejecución es lineal al principio y luego entra en un bucle:

Inicialización Base: Llama a HAL_Init() para resetear periféricos y configurar el temporizador base del sistema.

Configuración del Reloj: Llama a SystemClock_Config(). Según el código adjunto, configura el oscilador interno (HSI) y utiliza el multiplicador PLL para elevar la frecuencia del sistema a 64 MHz.

Inicialización de Periféricos: Llama a MX_GPIO_Init() (configura el botón de usuario B1 como interrupción y el LED LD2 como salida) y a MX_USART2_UART_Init() (configura la comunicación serie a 115200 baudios).

Lógica de Aplicación: Delega el comportamiento específico llamando a app_init() antes del bucle y a app_update() dentro del bucle infinito while(1). Aquí es donde residirá la lógica de tus diagramas de estado.

3. stm32f1xx_it.c
Este archivo gestiona las Rutinas de Servicio de Interrupción (ISR). Contiene las funciones que el hardware ejecuta automáticamente cuando ocurre un evento específico, pausando temporalmente el main.c:

SysTick_Handler(void): Se ejecuta periódicamente (generalmente cada 1 ms). Llama a HAL_IncTick(), que es responsable de incrementar la variable global del sistema que cuenta los milisegundos (uwTick).

EXTI15_10_IRQHandler(void): Se ejecuta cuando se detecta un cambio de estado en los pines 10 al 15. En tu placa, el botón azul de usuario (B1) está conectado al pin PC13. Esta interrupción llama a HAL_GPIO_EXTI_IRQHandler(B1_Pin) para procesar el evento del botón.

Evolución temporal de SystemCoreClock y SysTick
La variable global SystemCoreClock almacena la frecuencia principal del procesador (en Hertz). Por otro lado, el hardware del SysTick es un temporizador que cuenta de forma regresiva y, en el entorno HAL, incrementa una variable por software llamada uwTick cada vez que llega a cero.

A continuación se detalla su evolución cronológica:

Paso 1: Arranque en Reset_Handler y SystemInit()

El microcontrolador arranca utilizando su oscilador interno de alta velocidad (HSI), que por defecto funciona a 8 MHz.

SystemCoreClock: Se inicializa con el valor 8000000 (8 MHz) al declararse en system_stm32f1xx.c.

SysTick: Está completamente apagado. No cuenta ni genera interrupciones.

Paso 2: Entrada a main() y ejecución de HAL_Init()

La función HAL_Init() configura el hardware del SysTick para generar una interrupción exactamente cada 1 milisegundo, basándose en el reloj actual de 8 MHz.

SystemCoreClock: Se mantiene en 8000000.

SysTick: Se enciende y comienza a contar. La variable por software uwTick se inicializa en 0 y comienza a incrementar +1 cada milisegundo.

Paso 3: Ejecución de SystemClock_Config()

Esta función reconfigura radicalmente el árbol de relojes. Según tu código, toma el HSI (8 MHz), lo divide por 2 (4 MHz) y usa el PLL para multiplicarlo por 16 (RCC_PLL_MUL16). El resultado es una frecuencia de núcleo de 64 MHz. Al finalizar la configuración, la función actualiza el valor de la variable global de reloj y reconfigura el temporizador base.

SystemCoreClock: Pasa abruptamente de 8000000 a 64000000 (64 MHz).

SysTick: Su registro de recarga (reload value) en el hardware se modifica automáticamente para adaptarse a los nuevos 64 MHz, garantizando que la interrupción siga ocurriendo exactamente cada 1 milisegundo. La variable uwTick no se reinicia; sigue sumando desde el valor que tenía.

Paso 4: Ingreso al loop principal while(1)

El sistema ha terminado de configurarse e ingresa al ciclo de ejecución continua donde se ejecuta tu app_update().

SystemCoreClock: Se mantiene constante en 64000000 indefinidamente.

SysTick: Sigue funcionando en segundo plano. La variable uwTick continuará incrementándose en +1 cada milisegundo durante todo el tiempo que la placa permanezca encendida.

Resumen de Estados

Etapa de Ejecución,Valor de SystemCoreClock,Estado del Timer SysTick,Variable uwTick (Milisegundos)
Inicio (Reset_Handler),8.000.000,Apagado,No inicializada / 0
Fin de HAL_Init(),8.000.000,Encendido (Interrumpe a 8 MHz),Inicia en 0 y comienza a sumar
Fin de SystemClock_Config(),64.000.000,Encendido (Reajustado a 64 MHz),Continúa sumando sin perder la cuenta
Dentro del while(1),64.000.000,Encendido de forma permanente,Sigue incrementando indefinidamente
