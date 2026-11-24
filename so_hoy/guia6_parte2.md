# Parte 2 - Ejercicios de desarrollo

### Ejercicio 7 (*) / ¿FAT o Inodos?

Decidir si es mejor utilizar FAT o Inodos dependiendo de las necesidades en cada item.  

#### a. Creación de enlaces simbólicos

En un sistema de archivos basado en FAT no se pueden crear enlaces simbólicos (ausencia de atributo nativo para marcar a un archivo como enlace simbólico). En cambio, en uno basado en inodos sí. Conviene usar inodos.

#### b. Cantidad de sectores para estructuras auxiliares acotada

Es importante que la cantidad de sectores utilizados para guardar estructuras auxiliares sea acotada, independientemente del tamaño del disco.  

En un sistema basado en FAT, el tamaño de la tabla FAT está determinado por la cantidad total de bloques del disco. Por lo que dependiendo del tamaño del disco, el tamaño de la tabla puede crecer. En cambio, un file-system basado en inodos no depende del tamaño del disco y tiene un tamaño acotado por el tamaño de los bloques de datos y direccionamiento. Conviene utilizar inodos.

#### c. Tamaño máximo de archivo limitado solo por el tamaño del disco

Con inodos, la longitud de un archivo está limitada por la cantidad de bloques direccionables totales y la longitud de cada uno. En cambio, en FAT, el tamaño máximo está acotado solamente por el tamaño del disco. Conviene utilizar FAT.

#### d. Cantidad de memoria principal ocupada por filesystem lineal a la cantidad de archivos abiertos

Es importante que la cantidad de memoria principal ocupada por estructuras del filesystem en un instante dado sea (a lo sumo) lineal en la cantidad de archivos abiertos en ese momento.

En un sistema basado en FAT, incluso aunque no haya archivos abiertos, es necesario tener toda la tabla FAT cargada en memoria. En cambio, en inodos, solo es necesario cargar en memoria los inodos de los archivos que están en uso.  

Por lo tanto, la cantidad de memoria ocupada en FAT es constante, mientras que en inodos es lineal con respecto a la cantidad de archivos abiertos. Conviene utilizar inodos.


### Ejercicio 8 / FAT + Hash COMPLETAR

### Ejercicio 9 / COMPLETAR

### Ejercicio 10 (*) /  FAT

Implementar la función `datos = cargar_archivo(directorios[])` para obtener el contenido de un archivo a partir de su ruta absoluta en un disco formateado con FAT.  
- `directorios[]` es una lista ordenada con los nombres de los directorios de la ruta, incluyendo el nombre del archivo al final y excluyendo el directorio raíz.
    - \Documentos\Users\foto.png -> ['Documentos', 'Users', 'foto.png']

Se cuentan con las siguientes funciones auxiliares:      

```C
// devuelve entrada de la FAT de la posición block_address
FAT_entry(block_address);
// lee del disco todos los bloques indicados por parámetro, en orden.
raw_data = read_blocks(block_address1,block_address2,...);
// devuelve una lista de struct_entrada_directorio, donde cada elemento representa los subdirectorios del directorio pasado en raw_data.
parse_directory_entries(raw_data);
// devuelve los datos de la tabla de directorios de root
raw_data = root_table();

```

#### a. Estructura struct_entrada_directorio

Hay que enumerar tres campos que debe tener esa estructura.

```C
typedef struct {
    char name[256];
    int size;  
    int initial_block_address;
    int attr; // indica si es directorio o archivo
} entrada_directorio_t;
```

#### b. Implementar cargar_archivo

```C
char* cargar_archivo(char** directorios, int cant_directorios){
    int *raw_data = root_table(); // datos crudos de '/' 

    // itero por cada subdirectorio y, al final, el archivo
    for(int i = 0; i < cant_directorios; i++){ 
        // parseo todas las entradas del directorio actual subdirectorios
        entrada_directorio_t *entradas = parse_directory_entries(raw_data);

        int j = 0;
        while(strcmp(entradas[j].name, directorios[i]) != 0 ){ 
            j++;  // asumimos que existe
        }

        entrada_directorio_t entrada_buscada = entradas[j];

        // ahora tengo la entrada buscada de la iteración de los directorios
        int block_address = entrada_buscada.initial_block_address;
        int cant_bloques = contar_cant_bloques(block_address);
        int *bloques = malloc(cant_bloques * sizeof(int));

        for(int k = 0; k < cant_bloques; k++){
            bloques[k] = block_address;
            block_address = FAT_entry(block_address);
        }

        raw_data = read_blocks(bloques, cant_bloques);
        free(bloques);
    }
    return raw_data;
}

int contar_cant_bloques(int initial_block_address){
    int current = initial_block_address;
    int res = 0;
    while(current != EOF){
        res++;
        current = FAT_entry(current_block_address);
    }
    return res;
}

```

### Ejercicio 11 (*) /  Recorrido completo en EXT2

#### a. 

Se debe implementar una función `find_file_less_size(char *dir, int min_bytes, char *arch_nombre)`, que tome el path de un directorio, una cantidad de bytes y un nombre, y devuelva una lista con todos los archivos que tengan un tamaño menor a `min_bytes`, cuyo nombre sea `arch_nombre` a partir de `dir` (inclusive). La busqueda deberá también considerar los subdirectorios del directorio dado (excepto `.` y `..`).  


```C
#define BLOCK_SIZE 
#define EXT2_FT_DIR
#define EXT2_FT_REG_FILE

int* find_file_less_size(char *dir, int min_bytes, char *arch_nombre){
    std::vector<int> lista;
    find_file_less_size_aux(dir, min_bytes, arch_nombre, lista);
    return vector_to_array(lista);
}

void find_file_less_size_aux(char *dir, int min_bytes, char *arch_nombre, std::vector<int> &lista){

    // obtengo inodo del path
    struct Ext2FSInode* inodo = inode_for_path(dir);

    // itero en bloques de datos
    unsigned int cant_bloques = inodo->size / BLOCK_SIZE;
    for(int i = 0; i < cant_bloques; i++){

        // obtengo dirección del bloque y lo cargo en buffer
        unsigned int block_addr = get_block_address(inodo, i);
        char buffer[BLOCK_SIZE];
        read_block(block_addr, buffer);

        // itero dir_entries del bloque
        int offset = 0;
        while(offset < BLOCK_SIZE){

            // casteo dir_entry
            Ext2FSDirEntry* entry = (Ext2FSDirEntry*)(buffer + offset);
            if(entry->record_length == 0) break; // fin del bloque

            // 1. saltear "." y ".."
            if(strcmp(entry->name, ".") != 0 && strcmp(entry->name, "..") != 0){
                // 2. veo si es directorio o archivo

                // subdirectorio -> recursión
                if(entry->file_type == EXT2_FT_DIR){
                    char subdir_path[1024];
                    if(strcmp(dir, "/") == 0){ // si está en root
                        sprintf(subdir_path, "/%s", entry->name);
                    } else { // si ya pasé por root
                        sprintf(subdir_path, "%s/%s", dir, entry->name);
                    }
                    find_file_less_size_aux(subdir_path, min_bytes, arch_nombre, lista);
                }

                // archivo -> ¿cumple condiciones?
                if(entry->file_type == EXT2_FT_REG_FILE){
                    
                    if(strcmp(entry->name, arch_nombre) == 0){
                        // si el nombre cumple, cargo inodo para ver tamaño
                        struct Ext2FSInode* inodo_arch = load_inode(entry->inode);
                        if(inodo_arch->size < min_bytes){
                            lista.push_back(entry->inode);
                        }
                    }

                }
            }
            // itero por tamaño indicado en dir_entry
            offset += entry->record_length;
        }
    }
}
```


#### b.

Se debe diseñar la lista que retornará la función para que contenga: Información de cada tipo de archivo encontrado, su última vecha de modificación, su tamaño y propietario. Además, contar con:

```cpp
// se cuenta con el tamaño del bloque en BLOCK_SIZE
struct Ext2FSDirEntry {
    unsigned int inode;
    unsigned short record_length;
    unsigned char name_length;
    unsigned char file_type;
    char name[];
};

struct Ext2FSInode {
    unsigned short mode; // info sobre el tipo de archivo y los permisos
    unsigned short uid; // id de usuario
    unsigned int size; // en bytes
    unsigned int atime;
    unsigned int ctime;
    unsigned int mtime; // fecha ultima modificacion
    unsigned int dtime;
    unsigned short gid; // id de grupo
    unsigned short links_count; // cantidad de enlaces al archivo
    unsigned int blocks;
    unsigned int flags;
    unsigned int os_dependant_1;
    unsigned int block[15];
    unsigned int generation;
    unsigned int file_acl;
    unsigned int directory_acl;
    unsigned int faddr;
    unsigned int os_dependant_2[3];
};

// dado el id de un usuario, devuelve el nombre del usuario.
char * nombre_propietario(unsigned short id_usuario);
// dado un path, devuelve su inodo
struct Ext2FSInode * Ext2FS::inode_for_path(const char * path);
// dada una direccion de bloque y un buffer, carga el bloque indicado en el buffer
void Ext2FS::read_block(unsigned int block_address, unsigned char * buffer);
// dados un inodo y un numero de bloque, recorre el inodo buscando la direccion del bloque de datos indicado
unsigned int Ext2FS::get_block_address(struct Ext2FSInode * inode, unsigned int block_number);
// dado un numero de inodo, busca el inodo en el grupo y lo devuelve
struct Ext2FSInode * Ext2FS::load_inode(unsigned int inode_number);
```

Solución:  

```C
typedef struct ResultadoArchivo {
    char nombre[256];
    unsigned char tipo_archivo;
    unsigned int fecha_modificacion;
    unsigned int tamaño;
    unsigned short informacion;
    unsigned short user;
    unsigned int inode_number;
};


ResultadoArchivo* find_file_less_size(char *dir, int min_bytes, char *arch_nombre){
    std::vector<ResultadoArchivo*> lista;
    find_file_less_size_aux(dir, min_bytes, arch_nombre, lista);
    return vector_to_array(lista);
}

void find_file_less_size_aux(char *dir, int min_bytes, char *arch_nombre, std::vector<ResultadoArchivo*> &lista){

    // obtengo inodo del directorio
    struct Ext2FSInode *inodo = inode_for_path(dir);

    // itero en bloques de datos del directorio
    unsigned int cant_bloques = inodo->size / BLOCK_SIZE;
    for(int i = 0; i < cant_bloques; i++){

        // obtengo dirección del bloque y lo cargo en buffer
        unsigned int block_addr = get_block_address(inodo, i);
        char buffer[BLOCK_SIZE];
        read_block(block_addr, buffer);

        // itero dir_entries del bloque
        int offset = 0;
        while(offset < BLOCK_SIZE){

            // casteo dir_entry
            Ext2FSDirEntry* entry = (Ext2FSDirEntry*)(buffer + offset);
            if(entry->record_length == 0) break; // fin del bloque

            // 1. saltear "." y ".."
            if(strcmp(entry->name, ".") != 0 && strcmp(entry->name, "..") != 0){
                // 2. veo si es directorio o archivo

                // subdirectorio -> recursión
                if(entry->file_type == EXT2_FT_DIR){
                    char subdir_path[1024];
                    if(strcmp(dir, "/") == 0){ // si está en root
                        sprintf(subdir_path, "/%s", entry->name);
                    } else { // si ya pasé por root
                        sprintf(subdir_path, "%s/%s", dir, entry->name);
                    }
                    find_file_less_size_aux(subdir_path, min_bytes, arch_nombre, lista);
                }

                // archivo -> ¿cumple condiciones?
                if(entry->file_type == EXT2_FT_REG_FILE){
                    
                    if(strcmp(entry->name, arch_nombre) == 0){
                        // si el nombre cumple, cargo inodo para ver tamaño
                        struct Ext2FSInode* inodo_arch = load_inode(entry->inode);
                        if(inodo_arch->size < min_bytes){
                            struct ResultadoArchivo* archivo = malloc(sizeof(ResultadoArchivo));
                            strcpy(archivo->nombre, entry->name);
                            archivo->tipo_archivo = entry->file_type;
                            archivo->fecha_modificacion = inodo_arch->mtime;
                            archivo->tamaño = inodo_arch->size;
                            archivo->informacion = inodo_arch->mode;
                            archivo->user = inodo_arch->uid;  
                            archivo->inode_number = entry->inode;
                            lista.push_back(archivo);
                        }
                    }

                }
            }
            // itero por tamaño indicado en dir_entry
            offset += entry->record_length;
        }
    }
}
```

#### c. ¿Problemas si consideramos enlaces simbólicos?

Tenemos varios problemas con los enlaces simbólicos. Primero, en el código no se chequea si los archivos son enlaces simbólicos, por lo que habría que hacerlo. Además, en el caso de encontrar un inodo correspondiente a un enlace simbólico, habría que chequear si el link está roto o no (ya que podría pasar que la ruta a la que apunta no existe más o se renombró) y, en el caso de que no esté roto, habría que extraer el path leyendo el primer bloque de datos del inodo y luego buscar en ese path. Por último, podriamos tener problemas con la recursión, ya que podriamos acceder a un path que ya hayamos recorrido (rompiendo la estructura de árbol) o hasta llegando a tener ciclos infinitos.