# Windows_Internals
Windows es un sistema orientado a objetos, todo en el sistema operativo es un objeto.  
Los objetos están en el espacio de memoria y solamente son accesibles mediante el kernel. Es el kernel el que puede 
## Procesos
El proceso es un objeto de administración que provee la información necesaria para ejecutar un programa.  
Un proceso no corre ni ejecuta código, eso lo hacen los hilos.  
El proceso no conoce ni necesita las direcciones físicas de la memoria, únicamente conoce las direcciones virtuales y es el gestor de memoria del sistema operativo el que hace esa traducción.  
Un proceso contiene:  
* Un espacio de memoria privado.
* El fichero ejecutable, un enlace al fichero en disco que contiene la función main.
* Una tabla del indicador de los manejadores de objetos del kernel asociados a él.  

El sistema operativo puede asignar más memoria a los procesos de la RAM que tiene, usando el fichero de paginación.  
Es el gestor de la memoria el que va asignando memoria física a la memoria virtual cuando necesita ejecutar información de esas páginas.  
Algunas páginas de la memoria física pueden ser compartidas, por ejemplo cuando se ejecutan dos instancias de un mismo programa, la página en la memoria física que almacena el ejecutable o dlls compartidas, puede ser compartida.  
La columna del Administrador de Tareas nos muestra la memoria ocupada por un proceso, pero únicamente aquella que el proceso tiene marcada como memoria privada. Para ver la memoria completa que ocupa un proceso, necesitaremos activar la columna Tamaño de asignación.  


## Hilos
Los hilos son los encargados de ejecutar el código.  
Un hilo contiene:  
* El estado de los registros de la CPU.
* Sesión o modo de acceso asignado (user mode o kernel mode)
* Una pila con las variables, métodos... en el espacio del kernel y otra en el espacio del usuario. ¿?
* Un área de almacenamiento privado, llamado Thread Local Storage (TLS).
* Token de seguridad. (opcional)
* Cola de mensajes y ventanas que el hilo ha creado. (opcional)
* Prioridad. Número de 0-31. Prioridad de ejecución cuando sea programado. 31 es la mayor prioridad.
* Estado: Running, ready, waiting.  

En la parte del stack del hilo, podemos ver las llamadas desde el user mode al kernel mode.

## Manejadores (handles)
Es el kernel el único que puede obtener un puntero a un objeto y manipularlo. Desde la sesión de un usuario, para poder acceder a un objeto, es necesario un manejador que hace las veces de intermediario. Cuando obtienes un manejador, la dirección de acceso a él es añadida a tu tabla de manejadores y cuando dejas de necesitarlo y llamas a la función de closehandle, lo único que ocurre es el borrado de esa entrada, tu no sabes si el manejador es borrado o no, si otro proceso tiene abierto un manejador contra el mismo objeto se mantendrá activo pero si el número de manejadores abiertos para un objeto se vuelve 0, el objeto se borra a si mismo.  
Desde ProcessExplorer podemos ver los manejadores de un proceso y podemos activar la vista de ObjectAddress, esta dirección es la dirección del objeto en la sesión de kernel.  
La columna access indican las flags sobre qué podemos hacer con ese manejador en concreto.

## Arquitectura
![Imagen de arquitectura](./images/user-kernel_mode.PNG?raw=true "Ficheros")  
Si por ejemplo, queremos crear un fichero, nuestra aplicación utilizará la función de Win32 CreateFile, esa función pertenece a la Subsystem DLL  kernel32.dll. Kernel32.dll llamará a ntdll.dll, ya que esa dll es quien puede interactuar con el kernel.  
Ntdll llamará a la función NtCreateFile que no está documentada, al igual que el resto de funciones de ntdll.  
Executive es un grupo de componentes del sistema como el gestor de memoria, el gestor de I/O ..., es el encargado de realizar las comprobaciones de seguridad.  
Executive realiza las llamadas al propio kernel o a los drivers del sistema.  
El kernel es el encargado de manejar los hilos y los manejadores.  
En este caso, llamará al ntfs.sys, el driver encargado del control del sistema de ficheros.  
ntfs.sys interactuará con el Hardware Abstraction Layer (HAL), que es la capa más baja del sistema operativo y es el encargado de realizar la interacción con el hardware del equipo.  
Win32k.sys es el encargado de la gestión de la GUI de windows. Las funciones como CreateWindows no pasan por el Executive.  
Los Enviroment Subsystem pueden ser win32, unix... en el caso de win 32, la dll del sistema es csrss.exe.  
Los servicios son procesos que arrancan cuando se carga el subsistema y corren bajo local system, network service  local service.  
Dentro de procesos del sistema está por ejemplo csrss.exe, SCM (service control manager), smss.exe, si estos procesos se corrompen producen un error fatal en el sistema.
