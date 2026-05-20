# Gestión de Interrupciones en FreeRTOS

## ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción (ISR)?

En FreeRTOS solo se pueden utilizar las funciones de la API que terminan con el sufijo `FromISR`. Esto se debe a que las funciones "normales" de la API (como `vTaskDelay` o `xQueueReceive`) pueden intentar poner a la tarea actual en estado **BLOCKED** (Bloqueada). Esto no tiene sentido dentro de una interrupción de hardware y, si se intenta, causará que el microcontrolador colapse (Hard Fault) o que el sistema se vuelva inestable. Las versiones `FromISR` están diseñadas específicamente para no bloquear jamás la ejecución y ejecutarse de forma rápida.

* **Ejemplos:** `xQueueSendFromISR()`, `xSemaphoreGiveFromISR()`, `vTaskNotifyGiveFromISR()`.

---

## ¿Qué métodos existen para delegar el procesamiento de interrupciones a una Tarea?

Los métodos principales son:

1. **Semáforos Binarios o Task Notifications (Notificaciones de Tarea):** Es el método más rápido. La ISR desbloquea a una tarea de alta prioridad enviando una notificación (`vTaskNotifyGiveFromISR()`). La tarea, que estaba esperando, se despierta instantáneamente, procesa el evento y vuelve a dormir.
2. **Colas (Queues):** Si la interrupción necesita enviar datos además del "aviso" (por ejemplo, caracteres recibidos por una UART), se usan colas.
3. **Llamada a función en diferido (`xTimerPendFunctionCallFromISR`):** Permite a una ISR "encolar" una función para que sea ejecutada un poco más tarde por la tarea demonio de los temporizadores de FreeRTOS (*Timer Daemon Task*).

---

## ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?

**A. De la ISR hacia una Tarea (Datos hacia afuera de la ISR):**
1. El hardware genera la interrupción (ej. llega un dato del sensor).
2. En la ISR se llama a `xQueueSendFromISR()` para copiar el dato a la cola.

**B. De una Tarea hacia la ISR (Datos hacia adentro de la ISR):**
1. La Tarea productora pone datos en la cola usando la función normal `xQueueSend()`.
2. Cuando el hardware está listo para enviar más datos, la ISR se dispara.
3. Dentro de la ISR, se llama a `xQueueReceiveFromISR()` para extraer un dato de la cola y enviarlo físicamente al periférico.

---

## ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

En STM32, FreeRTOS soporta un **modelo de anidamiento completo**. Esto significa que una interrupción de hardware puede ser "interrumpida" por otra interrupción de mayor prioridad.

Para evitar que el RTOS corrompa datos durante el anidamiento, FreeRTOS divide las prioridades de hardware usando una constante clave (definida en `FreeRTOSConfig.h`): **`configMAX_SYSCALL_INTERRUPT_PRIORITY`**.

Esto crea dos categorías de interrupciones:

1. **Interrupciones No Gestionadas por FreeRTOS (Prioridad Hardware MÁS ALTA que el límite):**
    * Pueden interrumpir cualquier cosa, incluso el núcleo interno del RTOS.
    * Tienen **cero latencia** añadida por el sistema operativo.
    * **Prohibición absoluta:** NUNCA pueden usar las funciones de la API de FreeRTOS.
2. **Interrupciones Gestionadas por FreeRTOS (Prioridad Hardware IGUAL o MÁS BAJA que el límite):**
    * Pueden anidarse entre ellas.
    * **Sí** tienen permitido utilizar llamadas a la API que terminan en `...FromISR`.
    * Pueden experimentar un ligero retraso de microsegundos porque FreeRTOS deshabilita temporalmente este grupo de interrupciones mientras ejecuta sus propias secciones críticas (como actualizar la lista de tareas listas).

---

## Implementación y comportamiento observado: 

Se modificó la gestión del botón pasando de un esquema de consulta continua a un mecanismo eficiente **guiado por interrupciones** mediante el callback `HAL_GPIO_EXTI_Callback`. Al presionarse o liberarse el botón `B1_Pin`, la rutina de servicio de interrupción (ISR) detecta el flanco y evalúa su estado actual; según corresponda, libera (`xSemaphoreGiveFromISR`) el semáforo binario `h_btn_led_off_bin_sem` o `h_btn_led_blink_bin_sem`. 

Este cambio optimiza el uso del procesador, ya que `task_btn` deja de consumir recursos en bucles de espera y pasa a estar en **estado bloqueado (eficiente)**, despertándose de forma inmediata solo cuando la ISR entrega el semáforo mediante `portYIELD_FROM_ISR` si una tarea de mayor prioridad lo requiere.

