# Semáforos en FreeRTOS

En FreeRTOS, los semáforos constituyen otro de los mecanismos de comunicación y sincronización entre tareas. Funcionan, esencialmente, mediante primitivas de tipo `give` (en parte de la bibliografía empleada en el curso equivale a `signal`), que los liberan, y `take` (en parte de la bibliografía empleada en el curso equivale a `get`), que los toman.

---

## ¿Cuáles son las diferencias entre semáforos binarios y semáforos contadores?

La principal diferencia entre ambos tipos es que el semáforo binario solamente puede tener dos valores, 0 (taken) o 1 (free), mientras que el semáforo contador alberga un valor entre 0 y un máximo `n` definido a la hora de su creación, de forma tal que cada `give` incrementa dicho contador y cada `take` lo decrementa.

Así, el semáforo binario es ideal para sincronización o comunicación básica de un evento singular, mientras que el semáforo contador será más adecuado cuando se quiere llevar la cuenta de múltiples eventos o recursos disponibles.

---

## ¿Cómo crear y usar semáforos binarios y semáforos contadores?

Un semáforo binario se crea mediante la primitiva `xSemaphoreCreateBinary()`, por ejemplo:

```
SemaphoreHandle_t h_bin_sem;

h_bin_sem = xSemaphoreCreateBinary();
```

Para crear un semáforo contador, en su lugar se usa la primitiva `xSemaphoreCreateCounting(<valor máximo>, <valor inicial>)`, por ejemplo:

```
SemaphoreHandle_t h_cnt_sem;

h_cnt_sem = xSemaphoreCreateCounting(10, 0) // Semáforo de 0 a 10 inicializado en 0.
```

En ambos casos, para liberar un semáforo se emplea la primitiva `xSemaphoreGive(<handle del semáforo>)`, y para (intentar) tomarlo la primitiva `xSemaphoreTake(<handle del semáforo>, <timeout>)`.

A la hora de tomar un semáforo, se necesita un timeout al igual que en el caso de las colas. Esto permite que la tarea que intenta realizar dicha acción se quede bloqueada por un tiempo determinado esperando al semáforo (por siempre si el timeout es `portMAX_DELAY`).

---

## Interacción entre `task_btn` y `task_led` mediante semáforos binarios.

En la actividad anterior, la interacción entre las tareas se hizo mediante una cola. En esta ocasión, vamos a lograrlo mediante dos semáforos binarios. Para ello, se hicieron una serie de modificaciones al código (principalmente en `app.c`, `app.h`, `task_btn.c` y `task_led.c`):

- Primero, se crean dos semáforos junto con la creación de tareas en `app.c`, verificándose el proceso con un `configASSERT` y agregándolos al registro (trace) de FreeRTOS. Como ambos son del tipo binario, no se necesita pasar ningún parámetro en su creación. Los nombres de estos semáforos son `h_btn_led_blink_bin_sem` y `h_btn_led_off_bin_sem`.

- En `task_btn.c` se reemplazaron las llamadas a `put_event_task_led` por primitivas del tipo `xSemaphoreGive`. Cuando el botón se pulsa y la orden es hacer parpadear al led, se libera el semáforo `h_btn_led_blink_bin_sem`; cuando el botón se suelta, el semáforo que se libera es el otro, `h_btn_led_off_bin_sem`.

- En `task_led.c` se reemplazaron las lecturas de la estructura de datos compartida por primitivas del tipo `xSemaphoreTake`. Cuando el led está apagado, el intento de tomar el semáforo de parpadeo es del tipo bloqueante (es decir, con un timeout de `portMAX_DELAY`). Cuando está parpadeando, no es posible que sea bloqueante dado que el led necesita estar actualizando su estado para parpadear a la vez que intenta tomar el semáforo de apagado.

- De este modo, la aplicación continúa funcionando igual que antes, y se logra que los archivos `task_led_interface.h` y `task_led_interface.c` ya no sean necesarios (aún así, se decidió conservarlos en el proyecto).