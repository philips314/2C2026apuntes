# Parte 2 - Implementación de drivers

### Ejercicio 7 (*) / Drivers de unidad de discos ópticos

Posee 3 registros de lectura y 3 de escritura.   

Los de escritura son:
- `DOR_IO`; enciende (escribiendo 1) o apaga (escribiendo 0) el motor de la unidad.
- `ARM`; número de pista a seleccionar.
- `SEEK_SECTOR`; número de sector a seleccionar dentro de la pista.
Los de lectura son:
- `DOR_STATUS`; contiene el valor 0 si el motor está apagado (o apagándose), 1 si está encendido (que valga 1 no garantiza velocidad rotacional suficiente).
- `ARM_STATUS`; contiene el valor 0 si el brazo se está moviendo, 1 si ya se ubica en la pista indicada en ARM.
- `DATA_READY`; contiene el valor 1 cuando el dato ya fue enviado.

Funciones auxiliares disponibles:
- `int cantidad_sectores_por_pista()`; empieza desde el sector 0.
- `void escribir_datos(void *src)`; escribe los datos apuntados por src en el último sector seleccionado.
- `void sleep(int ms)`; espera durante ms milisegundos.

Consideraciones del motor y tiempos:  
- Encendido; antes de escribir, el motor debe estar encendido. Si no lo está, encenderlo y esperar al menos 50 ms para alcanzar la velocidad rotacional suficiente.
- Apagado; al finalizar una operación, apagar el motor para ahorrar energía. Demora como máximo 200 ms en apagarse (no se pueden iniciar nuevas operaciones antes de que termine).

#### a. 

Implementar `write(int sector, void *data)` del driver sin usar interrupciones. Esta tiene que escribir los datos apuntados por `data` en el sector en formato LBA indicado por `sector`.

```C
sem_t mutex;

int driver_init(){
    mutex = sem_init(1);
    return IO_OK;
}

void write(int sector, void *data){
    mutex.wait(); // exclusión mutua


}
```