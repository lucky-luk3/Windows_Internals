# Windows_Internals
## Procesos
El proceso es un objeto de administración que provee la información enecesaria para ejecutar un programa.  
Un proceso no corre ni ejecuta código, eso lo hacen los hilos.  
El proceso no conoce ni necesita las direcciones físicas de la memoria, únicamente conoce las direcciones virtuales y es el gestor de memoria del sistema operativo el que hace esa traducción.  
Un proceso contiene:  
* Un espacio de memoria privado.
* El fichero ejecutable, un enlace al fichero en disco que contiene la funion main.
* Una tabla del indicador de los manejadores de objetos del kernel asociados a el.  

El sistema operativo puede asignar más memoria a los procesos de la RAM que tiene, usando el fichero de paginación.  
Es el gestor de la memoria el que va asignando memoria física a la memoria virtual cuando necesita ejecutar información de esas páginas.  
Algunas páginas de la memoria física pueden ser compartidas, por ejemplo cuando se ejecutan dos instancias de un mismo programa, la página en la memoria física que almacena el ejecutable o dlls compartidas, puede ser compartida.  
La columna del Administrador de Tareas nos muestra la memoria ocupada por un proceso, pero únicamente aquella que el proceso tiene marcada como memoria privada. Para ver la memoria completa que ocupa un proceso, necesitaremos activar la columna Tamaño de asignación.  


## Hilos
Los hilos son los encargados de ejecutar el código.  
Un hilo contiene:  
* El estado de los registros de la CPU.
* Sesion o modo de acceso asignado (user mode o kernel mode)
* Una pila con las variables, metodos... en el espcio del kernel y otra en el espacio del usuario. ¿?
* Un area de almacenamiento privado, llamado Thread Local Storage (TLS).
* Token de seguridad. (opcional)
* Cola de mensajes y ventanas que el hilo ha creado. (opcional)
* Prioridad. Número de 0-31. Prioridad de ejecución cuando sea programdao. 31 es la mayor prioridad.
* Estado: Running, ready, waiting.  

En la parte del stack del hilo, podemos ver las llamadas desde el user mode al kernel mode.
