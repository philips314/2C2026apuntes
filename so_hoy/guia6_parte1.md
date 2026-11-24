# Parte 1 - Introducción a Sistemas de Archivos


### Ejercicio 1 / ¿Cuántos accesos al disco hasta llegar al último bloque N en FAT?

Tenemos una computadora cuyo disco se accede sin memoria cache y un sistema de archivos FAT. Además, la FAT no queda en la memoria (recordar que lo normal es que la FAT se cargue en memoria) ¿Cuántos accesos al disco son necesarios para llegar hasta el último bloque de un archivo de N bloques?

En este contexto, cada vez que necesitamos saber el "siguiente bloque", se debe realizar una lectura directa al disco para consultar la FAT. Imagino que tenemos N bloques secuenciales $B_1, ..., B_N$ que corresponden al archivo que queremos leer.

```
SECTOR DE ARRANQUE / FAT / DIRECTORIO ROOT / DATOS
```

Asumiendo que la entrada de directorio del archivo ya fue leída, obtenemos la primer dirección $B_1$ en la entrada de directorio. Notar que como la FAT reside en el disco y no hay memoria caché, cada consulta a la tabla requiere un acceso a disco.
- Para saber dónde está $B_2$ tenemos que leer el bloque $B_1$ en la tabla FAT -> 1 acceso al disco.
- Para saber dondé está $B_3$ tenemos que leer el bloque $B_2$ en la tabla FAT -> 2 accesos al disco ...
- Para saber dondé está $B_N$ tenemos que leer el bloque $B_{N-1}$ en la tabla FAT -> $N-1$ accesos al disco.
- Una vez obtenida la dirección física del bloque $B_N$ (sabemos que es la última porque su entrada en FAT tiene un EoF), se realiza la lectura de su contenido en disco -> $N$ accesos a disco.  

Para llegar y acceder al contenido del último bloque, se requieren $N$ accesos al disco. Si además se requiriera haber leído todos los bloques de datos intermedios durante su trayecto, se sumarían $N-1$ accesos más a datos, danto un total de $2N - 1$ accesos.

### Ejercicio 2 (*) / Tabla FAT

Tenemos un disco de 128GB y bloques de 8KB. Supongamos un sistema de archivos similar a FAT, donde la tabla se ubica desde la posición 0, y cada entrada de la tabla es de 24B.

#### a. Tamaño de la tabla.

Capacidad del disco: $128GB = 2^7 * 2^{30} = 2^{37}$ bytes.  
Tamaño de bloques: $8KB = 2^{3}*2^{10} = 2^{13}$ bytes.  
Tamaño de cada entrada: $24$ bytes.
$$
\text{Cantidad de bloques: } \frac{2^{37}}{2^{13}} = 2^{24} \text{ bloques}.
$$

Como cada entrada es de $24$ bytes, tenemos que la tabla ocupa $2^{24}*24$ bytes. Como 1MB es $2^{20}$ bytes, nos queda: $2^4 * 24 = 384$ MB. 

#### b. ¿Cuántos archivos de 10MB es posible almacenar?

El disco tiene $128$ GB de capacidad. Es decir, $128*2^{10}$ MB. La FAT ocupa $384$ MB, por lo tanto, nos queda una capacidad de $130688$ MB. Es decir, podemos almacenar unos $128*2^{10} - 384 = 13068$ archivos de $10$ MB.

#### c. Indicar tamaño de archivo

Se sabe que un archivo comienza en el bloque 20. Dada la siguiente FAT, indicar el tamaño de dicho archivo:

| Bloque | 0 | 1 | 2 | 3 | 4 | 5 | 6 | ... | 20 | 21 | 22 | ... |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Siguiente** | EOF | 2 | 23 | 4 | 5 | 0 | 7 | ... | 21 | 22 | 3 | ... |

Empezamos con el bloque 20 -> 21 -> 22 -> 3 -> 4 -> 5 -> 0 -> EoF; son 7 bloques de $2^{13}$ bytes. Por lo que podriamos asumir que el archivo pesa $7*2^{13}$ bytes. Es decir, $7*2^{3} = 56$ KB.

### Ejercicio 3 (*) / Calculando bloques y demás en EXT2

Tengo un sistema UNIX, similar a ext2, que tiene bloques de tamaño 4KB y el direccionamiento a bloques de disco (LBA) es de 8B. A su vez, cada inodo cuenta con 5 entradas directas, 2 indirectas simples y una con indirección doble.

#### a. ¿Tamaño máximo de archivo que soporta?

Cantidad de punteros por bloque: $\frac{4KB}{8B} = \frac{2^{2}*2^{10}}{2^{3}} = 2^{9} = 512$ punteros.  
Cantidad de bloques de datos direccionables:  
- Bloques directos: $5$ bloques.  
- Bloques por indirección simple: $2*(2^{9}) = 2^{10} = 1024$ bloques.  
- Bloques por indirección doble: $1*(2^{9})^{2} = 2^{18} = 262144$ bloques.  

Cantidad total de bloques direccionables: $5 + 2^{10} + 2^{18} = 263173$ bloques.  

Tamaño máximo: $1024\text{MB} + 4\text{MB} + 20\text{KB} = 1028\text{MB} + 20\text{KB}$
- Directos: $5*4\text{KB} = 20\text{KB}$
- Indirección simple: $1024*4\text{KB}= 4\text{MB}$
- Indirección doble: $262144*4\text{KB} = 2^{18}+2^{12} = 2^{30} = 1\text{GB}$


#### b. ¿Porcentaje de disco desperdiciado?

Considerando solo el espacio utilizado en los bloques de datos. El 50% del espacio en disco está ocupado por archivos de 2KB, el 25% por archivos de 4KB y el 25% restante por archivos de 8 KB.

Como el tamaño de cada bloque es de 4KB, cada archivo de 2KB desperdicia 2KB. Con solo estos archivos estamos desperdiciando el 25% del disco (puesto que 50% del disco está ocupado con este tipo de archivos). Los archivos de 4KB y de 8KB no desperdician espacio porque entran exactamente en uno o dos bloques. Así que el sistema estaría desperdiciando el 25% del disco en fragmentación interna.

#### c. ¿A cuántos bloques debo acceder para procesar un archivo de 5MB?

Como cada bloque es de 4KB, necesitariamos leer $\frac{5\text{MB}}{4\text{KB}} = \frac{5*2^{20}}{2^{12}} = 5*2^8 = 1280$ bloques de datos.

- Entradas directas; cubren 5 bloques, me quedan 1275 bloques y no necesito bloques de indirección.
- Entradas indirectas simples; cubren 512 bloques y tengo dos, me quedan 251 bloques y necesito acceder a los dos bloques de indirección simple en disco.
- Entradas indirectas dobles; cargo la primer entrada, es decir, una que apunta a una indirecta simple. Esa entrada sola cubre 512 bloques, así que con cargar una ya me es suficiente.

Bloques de datos: $5 + 1024 + 251 = 1280$.  
Bloques de indirección: $2\text{ simples } + 1 \text{ doble } + 1 \text{ simple dentro de doble } = 4 \text{ bloques }$.  

Por lo tanto, se necesian los 1280 bloques de datos y 4 bloques de indirección. Es decir, 1284 bloques en total.

### Ejercicio 4 (*) / Calculando bloques y demás en EXT2 basado en inodos

Suponer un sistema basado en inodos (como ext2) y bloques de 4KB.

#### a. ¿Cuántos bloques para leer un archivo de 40KB?

En un inodo tengo 12 entradas simples, 1 de direccionamiento simple, 1 doble y 1 triple. Si ya tenemos el acceso al inodo asociado al archivo:    
- Como cada bloque ocupa 4KB, con las 12 entradas simples puedo leer $12*4\text{KB} = 48\text{KB}$, o sea que como mínimo me basta con leer 10 bloques de datos (no hace falta leer un bloque de indirección).

#### b. ¿Cuántos bloques para leer un archivo de 80KB?

Si ya tenemos acceso al inodo asociado al archivo; para leer 80KB va a haber que leer, primero, las 12 entradas simples del inodo. Luego, para leer los 32KB restantes, hay que usar la indirección simple para poder leer los restantes.  

Leemos el bloque de indirección simple y, luego, leemos como mínimo las primeras 8 entradas. Por lo tanto, necesitamos leer 21 bloques como mínimo.

### Ejercicio 5 / Comparando FAT con EXT2 en accesos al disco

Considerando un sistema ext2 (12 entradas directas, 3 indirectas, 1 doble indirecta y 1 triple indirecta) y un sistema FAT. ¿Cuántos bloques de disco se deben acceder para leer los siguientes bloques de un archivo?

Además, asumir que sé conoce el número del bloque inicial en FAT y el número de inodo en ext2.  

**EXT2, asumiendo que los bloques son de 4KB y los punteros de 4B**
1. Cantidad de punteros por bloque: 
$$
 K = \frac{4\text{KB}}{4\text{B}} = \frac{2^{12}}{2^{2}} = 1024 \text{ punteros}
$$
2. Cantidad de bloques direccionables:
- Directos: $12$ bloques de datos. 
- Indirección simple: $3*K = 3074$
- Indirección doble: $1*K^{2} = 1048576$
- Indirección triple: $1*K^{3} = 1073741824$

**FAT**
1. Suponiendo que la FAT está en la ram, leer N bloques requiere N accesos.
#### a. 1, 2, 3, 5, 7, 11, 13, 17, 23.

Con FAT: Suponiendo que la tabla FAT está cargada en memoria; hay que acceder a los 9 bloques para leer al archivo completo. Notar que no hace falta acceder a la tabla FAT para saber el siguiente bloque, solo tengo que acceder a los bloques que me piden.  

Con EXT2: Puedo leer los bloques 1, ..., 11 a partir de las entradas directas del inodo correspondiente al archivo (asumiendo que el inodo ya está en ram). Para los bloques 13, 17 y 23 tengo que leer un bloque de indirección simple. Por lo que terminaría leyendo 9 bloques de datos más uno de indirección simple.

#### b. 1, 2, 3, 4, 5, 6, 10001.

Con FAT: Igual que el anterior, hay que leer 7 bloques de datos.  

Con EXT2: Puedo leer los primeros 6 bloques a partir de las entradas directas del inodo. Para el bloque número 10001 voy a tener que acceder al bloque de indirección doble y, luego, el primer bloque de indirección simple en él. Es decir, hay que leer 7 bloques de datos, uno de indirección doble y otro al que apunta la indirecta doble.

#### c. 13, 10000, 1000000.

Con FAT: Hay que leer 3 bloques de datos.  

Con EXT2: El bloque 13 lo podemos leer desde la indirección simple (porque del 1 al 12 están en los directos). El número 10000 lo consigo del indirecto doble. El último bloque lo consigo leyendo del de indirección doble también. Por lo tanto, hay que cargar dos bloques para el 13, tres para el 10000 (porque cargo el indirecto doble, el simple al que apunta y al bloque correspondiente 10000) y 3 para el último (porque ya cargué el de indirección doble, luego cargo el doble simple al que apunta y el bloque correspondiente al número 1000000).

#### d. 14, 15, ..., 50.

Con FAT: Hay que leer 37 bloques de datos.  

Con EXT2: Los bloques del 14 al 50 están en el de indirección simple. Por lo que hay que cargar el bloque de la indirección simple y después leer los 37 bloques. Es decir, 38 lecturas de bloques.

### Ejercicio 6 / Comparando FAT con EXT2 en rutas de archivos

Considerar la ruta a un archivo: `/home/aprobar.txt`

#### a. Sistema basado en FAT

¿Cuántos bloques se deben acceder para llegar a los bloques de datos de ese archivo?  

Empezamos por la entrada de directorio root, busco el 1er bloque de la tabla de directorios para hallar la entrada home (utilizando metadato nombre). Cuando la encuentro, buscamos en la FAT el 1er bloque de datos de home para hallar `aprobar.txt`. Es decir, necesitamos acceder a dos bloques.

#### b. Sistema basado en EXT2

Se quiere leer el archivo `/pepe.txt` que es un enlace simbólico al archivo mencionado arriba. El enlace no está cargado en memoria. ¿Cuántos bloques de disco se deben acceder para llegar a los bloques de datos del archivo referenciado?  

El inodo raíz (inodo 2) ya está en ram, cada directorio entra en un bloque y los bloques leídos se mantienen en caché.  
1. Leo el primer bloque de datos del directorio raíz, para encontrar la entrada `pepe.txt` y obtener su número de inodo -> 1 acceso a disco.
2. Cargo inodo en ram y busco en el primer bloque de datos del archivo `pepe.txt` para obtener la ruta hacía el archivo orginal `/home/aprobar.txt` -> 2 acceso al disco.
3. Resolvemos la ruta `/home/aprobar.txt`, el primer bloque de datos del directorio raíz ya fue cargado, así que leemos el primer bloque de datos de `home` para obtener el inodo del archivo `aprobar.txt` -> 3 acceso al disco.  
4. El inodo del archivo `aprobar.txt` se carga en ram. Con él, ya tenemos acceso a sus punteros directos e indirectos.