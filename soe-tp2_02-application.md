# Colas en FreeRTOS

En FreeRTOS, una cola es un recurso o estructura del sistema operativo que actúa como uno de los posibles mecanismos de comunicación segura entre tareas, ya que éstas pueden acceder a las mismas enviando o recibiendo datos. Las colas tienen una longitud máxima fija configurada durante su creación pero se llenan o vacían dinámicamente.

---

## ¿Cómo crear una Cola?

Para crear una cola se usa la primitiva `xQueueCreate(<cantidad de elementos>, <tamaño de cada elemento>)`, por ejemplo:

```
QueueHandle_t h_cola;

h_cola = xQueueCreate(10, sizeof(uint16_t)); // Permite guardar hasta 10 enteros de 16 bits.
```

---

## ¿Cómo eliminar una Cola?

Para eliminar una cola se usa la primitiva `vQueueDelete(<handle de la cola>)`, por ejemplo:

`vQueueDelete(h_cola);`

---

## ¿Cómo gestiona una Cola los datos que contiene?

En cuanto a la gestión de datos, las colas en FreeRTOS siguen el orden FIFO (First In, First Out), por lo que el primer dato introducido va a ser el primer dato en ser extraido. Entonces, es más similar a un registro de desplazamiento (shift register) que a una pila (stack).

---

## ¿Cómo enviar datos a una Cola?

Para enviar un dato a una cola se usa la primitiva `xQueueSend(<cola>, <dirección del valor>, <timeout>)`, por ejemplo:

```
BaseType_t ret;
uint16_t numero = 55;

ret = xQueueSend(h_cola, &numero, portMAX_DELAY);
```
---

## ¿Cómo recibir datos de una Cola?

Para recibir el dato de una cola se usa la primitiva `xQueueReceive(<cola>, <dirección de la variable donde se guarda el dato recibido>, <timeout>)`, por ejemplo:

```
BaseType_t ret;
uint16_t numero_recibido;

ret = xQueueReceive(h_cola, &numero_recibido, portMAX_DELAY);
```

---

## ¿Qué significa bloquearse en una Cola?

Bloquearse en una cola significa que una tarea se queda esperando o bien a que se libere una posición en una cola llena, si se quieren enviar datos, o bien a que se llene una posición en una cola vacía, si se quieren recibir datos. Mientras espera (siempre y cuando el timeout sea distinto de 0), la tarea se queda en estado `Blocked`, por lo que no ocupa procesamiento. Una vez que la cola se actualiza, el scheduler cambia el estado de la tarea bloqueada a Ready, y, si esta tenía mayor prioridad que la que se estaba ejecutando hasta el momento, se hace un cambio de contexto inmediato.

Si el timeout vence (o directamente se configura como 0), la tarea también vuelve al estado Ready, pero la primitiva que accede a la cola devuelve `errQUEUE_EMPTY`/`errQUEUE_FULL` (iguales a `pdFALSE`).

Usar un timeout de `portMAX_DELAY` es equivalente a "esperar por siempre", por lo que en realidad la tarea nunca se desbloquea por tiempo, sino solamente cuando cambia el estado de la cola o si otra tarea la desbloquea. Esto es preferible a un esquema tipo "busy waiting" con un bucle ya que, al bloquearse la tarea, no consume procesamiento.

---

## ¿Cómo bloquearse en varias Cola?

Para bloquearse en varias colas o esperar múltiples fuentes de eventos, hay que usar un Queue Set. Por ejemplo:

```
QueueSetHandle_t h_queue_set;

h_queue_set = xQueueCreateSet(10);

xQueueAddToSet(h_cola_1, h_queue_set);
xQueueAddToSet(h_cola_2, h_queue_set);
```

En la tarea en cuestión:

```
QueueSetMemberHandle_t estado;

estado = xQueueSelectFromSet(queue_set, portMAX_DELAY);
```

`estado` valdrá `pdTRUE` cuando alguna de las dos colas pasa a tener un elemento, y la tarea se desbloquea.

---

## ¿Cómo sobrescribir datos en una Cola?

Para sobreescribir datos en una cola, se usa la primitiva `xQueueOverwrite(<cola>, <dirección del nuevo valor>)`; solo funciona en colas de tamaño 1, por ejemplo:

```
QueueHandle_t h_cola;

h_cola = xQueueCreate(1, sizeof(uint16_t));

uint16_t valor_1 = 1;

xQueueSend(h_cola, &valor_1, portMAX_DELAY);

// ...

uint16_t valor_2 = 2;

xQueueOverwrite(h_cola, &valor_2);
```

---

## ¿Cómo vaciar una Cola?

Para vaciar una cola se usa la primitiva `xQueueReset(<cola>)`. Esto vacía todas las posiciones y reinicia los índices internos.

---

## ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Tener en cuenta las prioridades es clave a la hora de usar colas. Puede suceder que:

- La tarea que llena la cola tenga mayor prioridad que aquella que recibe datos. En este caso, la tarea va a continuar insertando datos hasta que se bloquee cuando la cola se llene. Recién entonces, se puede ejecutar la que recibe datos, pero apenas extraiga un dato la tarea productora se desbloquea, resultando en un proceso poco eficiente.

- La tarea que llena la cola tiene menor prioridad que aquella que recibe datos. Dependiendo del caso, esto puede ser preferible, ya que apenas haya un dato en la cola, la tarea de mayor prioridad lo recibe y luego vuelve a bloquearse cuando quiere volver a leer, cediendo el procesamiento a la tarea productora.

Lo que debe evitarse absolutamente es la inanición (starvation), es decir que una cola nunca pueda llenarse o nunca pueda leerse como debería por prioridades mal asignadas.

---

## Interacción entre `task_btn` y `task_led` mediante una Cola.

El objetivo es gestionar la comunicación entre ambas tareas mediante una Cola de FreeRTOS en lugar de una estructura compartida como se venía haciendo hasta el momento. Para ello, se hicieron una serie de modificaciones al código (principalmente en `app.c`, `app.h`, `task_btn.c` y `task_led.c`):

- Para empezar, se crea una cola de longitud 1 junto con la creación de tareas en `app.c`, verificándose el proceso con un `configASSERT` y agregándola al registro (trace) de FreeRTOS. El tipo de dato que se guardará en la cola es `task_led_ev_t`, y, como nombre identificador de su handle, se le puso `h_btn_led_queue`, siguiendo las recomendaciones y estilo de nombramiento de variables.

- En `task_btn.c` se reemplazaron las llamadas a `put_event_task_led` por primitivas del tipo `xQueueOverwrite`, almacenando ya sea `EV_LED_BLINK` o `EV_LED_OFF`. Al ser la cola de longitud 1, se prefirió utilizar la primitiva para sobreescribir el dato en lugar de aquella usada para insertar un dato.

- En `task_led.c` se reemplazaron las lecturas de la estructura de datos compartida por primitivas del tipo `xQueueReceive`. Cuando el led está apagado, la lectura de la cola puede ser bloqueante (es decir, un timeout de `portMAX_DELAY`). Cuando está parpadeando, no es posible que sea bloqueante dado que el led necesita estar actualizando su estado para parpadear a la vez que intenta leer la cola.

- De este modo, la aplicación continúa funcionando igual que antes, y se logra que los archivos `task_led_interface.h` y `task_led_interface.c` ya no sean necesarios (aún así, se decidió conservarlos en el proyecto).