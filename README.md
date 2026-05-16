# FIUBA - Electrónica - Sistemas Operativos Embebidos
## Trabajo Práctico N° 2 - Comunicación de Tareas de FreeRTOS
### 2026-1erC - 1-03

### Responsables de la entrega:
| Padrón | Apellidos, Nombres | Fecha | Deadline |
| :----- | :--------------------- | :------: | :-------: |
| 110901 | Chechko, Víctor Nicolás | - | Semana 06 |
| 109308 | Marconi Casares, Lourdes | - | Semana 06 |
| 109324 | Solari Parravicini, Facundo | - | Semana 06 |

---

- En este trabajo, exploramos dos de los métodos de comunicación entre tareas que ofrece FreeRTOS, las colas y los semáforos, aplicándolos a la interacción simple entre un botón y un led que se analizó en el trabajo anterior.

- Al final, también se explora la posibilidad de gestionar el botón mediante una interrupción, empleando para ello un semáforo binario como mecanismo de comunicación entre la rutina de servicio de interrupción y `task_btn` (es decir, la tarea del botón).

---