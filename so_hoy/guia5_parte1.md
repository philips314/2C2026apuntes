# Parte 1 - Interfaz de E/S

### Ejercicio 1 (*) / ¿Describen el concepto de driver?

- Es una pieza de Software:  
    - Si, es un módulo de software del kernel que maneja un dispositivo a través de su controlador.
- Es una pieza de Hardware: 
    - No, el hardware es el perifé́rico fí́sico o su controlador electró́nico.
- Es parte del SO: 
    - Si, se ejecuta dentro del espacio de kernel formando parte del subsistema de E/S.
- Dado que el usuario puede cambiarlo, es una aplicación de usuario: 
    - No, aunque un usuario con permisos de administrador pueda cargar o descargar mó́dulos del kernel, el có́digo corre con privilegio de Kernel y no en espacio de usuario.
- Es un gestor de interrupciones: 
    - No, es más que eso. El driver contiene e instala rutinas de atenció́n de interrupciones (ISR), pero su funció́n no es solo gestionar interrupciones sino abstraer las llamadas `open`, `read`, `write` y `close` traducié́ndolas en operaciones de registros (Los drivers no son los que atienden las interrupciones, las interrupciones las atiende el kernel. Pero los drivers pueden proveer rutinas de atención de interrupciones).
- Tiene conocimiento del dispositivo que controla pero no del SO en el que corre: 
    - No, el driver tiene conocimiento de las dos cosas. 
- Tiene conocimiento del SO en el que corre y del tipo de dispositivo que controla, pero no de las particularidades del modelo especí́fico: 
    - No, el driver conoce precisamente los bits, la secuencia de comandos y los registros ú́nicos del modelo especí́fico de hardware que maneja.

### Ejercicio 2 / Driver: cronómetro

Registros de E/S:  
* `CHRONO_CURRENT_TIME` (Lectura): Tiempo acumulado. 
* `CHRONO_CTRL` (Escritura): Recibe `CHRONO_RESET` para reiniciar.

```C
int driver_read(int* data){
    int tiempo_actual = IN(CHRONO_CURRENT_TIME);
    if(copy_to_user(data, &tiempo_actual, sizeof(tiempo_actual)) != 0){
        return IO_ERROR;
    }
    return IO_OK;
}

int driver_write(int* data){
    OUT(CHRONO_CTRL, CHRONO_RESET);
    return IO_OK;
}
```

### Ejercicio 3 / Driver: tecla con Polling (Busy Waiting)

Registros de E/S:  
* `BTN_STATUS` con bits menos significativos de interes:
    - `BTN_STATUS0` vale 0 si la tecla no fue pulsada, 1 si lo fue.
    - `BTN_STATUS1` escribir 0 en este bit para limpiar la memoria de la tecla.

```C
sem_t mutex;

int driver_init(){
    sem_init(&mutex, 1);
    return IO_OK;
}

int driver_read(int* data){
    mutex.wait();
    while ((IN(BTN_STATUS) & 0b1) == 0) {/*busy-waiting activo*/};
    // la tecla se presionó
    int status_actual = IN(BTN_STATUS);
    status_actual &= ~2; // deja el segundo bit menos significativo en 0, el resto igual
    OUT(BTN_STATUS, status_actual);
    mutex.signal();

    int res = BTN_PRESSED;
    copy_to_user(data, &res, sizeof(res));
    return IO_OK;
}
```

### Ejercicio 4 (*) / Driver: crónometro con interrupciones 

```C
sem_t mutex;
sem_t sem_tecla;

void irq_handler(){
    // solo despierta al semáforo sin bloquearse nunca
    sem_tecla.signal();
}

int driver_init(){
    sem_init(&mutex, 1);
    sem_init(&sem_tecla, 0);

    if(request_irq(7, irq_handler) == IRQ_ERROR){
        return IO_ERROR;
    }

    // guardar constante en el registro de la tecla
    OUT(BTN_STATUS, BTN_INT);
    return IO_OK;
}

int driver_remove(){
    free_irq(7);
    return IO_OK;
}

int driver_read(int* data){
    mutex.wait(); // exclusión mutua, un solo proceso a la vez intenta leer
    sem_tecla.wait(); // espera bloqueada a que la ISR haga el signal

    // limpiar memoria del botón y rearmar la interrupción
    int status_actual = IN(BTN_STATUS);
    status_actual &= ~2;
    status_actual |= BTN_INT; // enciende los bits del BTN_INT
    OUT(BTN_STATUS, status_actual);

    mutex.signal();

    int res = BTN_PRESSED;
    copy_to_user(data, &res, sizeof(res));
    return IO_OK;
}
```

### Ejercicio 5 / Acciones que debe tomar el E/S

Cuando se efectúa un `open()`:  
- El administrador de E/S primero debe chequear que el dispositivo exista y verificar que el usuario tiene los permisos para acceder al dispositivo.
- Inicializar todas las estructuras para efectuar la comunicación de una aplicación de nivel usuario con el dispositivo.
- Actualizar las estructuras del sistema operativo que indican los dispositivos abiertos.

Cuando se efectúa un `write()`:  
- Primero hay que verificar que exista el dispositivo y que el usuario que trató de hacerlo tenga permisos de escritura.
- Luego debería hacer un `copy_from_user` sobre el dato que nos pasó el usuario, comunicarse con el dispositivo por medio de OUT y por último devolver la cantidad de bytes escritos para indicar que fue exitosa la escritura.

### Ejercicio 6 / Nivel de acceso para syscalls IN/OUT

Las syscalls IN y OUT deberian tener nivel de privilegio cero. Porque no querriamos que una aplicacion de usuario pueda directamente comunicarse con los dispositivos sin ser intermediado por el driver del kernel.