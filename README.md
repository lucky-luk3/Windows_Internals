# Windows_Internals

## Procesos
El proceso es un objeto de administración que provee la información necesaria para ejecutar un programa.  
Un proceso no corre ni ejecuta código, eso lo hacen los hilos.  
El proceso no conoce ni necesita las direcciones físicas de la memoria, únicamente conoce las direcciones virtuales y es el gestor de memoria del sistema operativo el que hace esa traducción.  

Un proceso contiene:  
* Un espacio de memoria privado.
* Una cantidad de memoria física privada asociada (working set)
* El fichero ejecutable, un enlace al fichero en disco que contiene la función main.
* Una tabla del indicador de los manejadores de objetos del kernel asociados a él.  
* Un token de seguridad que contiene los descriptores de seguridad del proceso.    
* Prioridad, esa prioridad será la asignada a los hilos del proceso.  

Un proceso termina cuando:
* Todos sus hilos terminan.
* Uno de los hilos llama a ExitProcess(Win32), esto ocurre a veces como rutina de cierre del hilo principal, de tal manera que cerrando ese hilo se cierra todo el proceso. Esta es la manera correcta de terminar un proceso ya que da tiempo al proceso y a las dlls a terminar por si mismas e incluso escribir en logs, liberar memoria o guardar información del fichero.  
* Uno de los hilos llama a TerminateProcess(Win32), es la manera no recomendada de terminar un proceso. Esta llamada puede hacerse desde el exterior de los hilos del proceso. 
Algunas páginas de la memoria física pueden ser compartidas, por ejemplo, cuando se ejecutan dos instancias de un mismo programa, la página en la memoria física que almacena el ejecutable o dlls compartidas, puede ser compartida.  
La columna del Administrador de Tareas nos muestra la memoria ocupada por un proceso, pero únicamente aquella que el proceso tiene marcada como memoria privada. Para ver la memoria completa que ocupa un proceso, necesitaremos activar la columna Tamaño de asignación. 

### Procesos protegidos
En Windows Vista, fueron incluidos los procesos protegidos, esto implica que ciertos procesos en el sistema, no serán accesibles por el resto del sistema.
Esto se implementó para poder reproducir contenido multimedia DRM con la certeza de que otro proceso no podría acceder al espacio de memoria del reproductor y volcar el contenido.
Estos procesos únicamente pueden carcar unas librerias firmadas firmadas con un certificado especial.
#### Procesos Protegidos Light
Fueron introducidos en Windows 8.1, ofrecian varios niveles de protección en los procesos.  
Incialmente fueron pensados para el software antivirus y la mayoria de los procesos del sistema fueron protegidos con esta medida.  
Esto permite que un proceso con privilegios de administardor no tenga porque tener permisos de acceso a un proceso protegido.  
Esta protección es unicamnete en User Mode, así que un administrador podría ejecutar un driver en Kernel Mode y tener acceso a todo.  

![Imagen de niveles de seguriad en procesos protegidos](./images/protected_processes_levels.png?raw=true "PPL Levels")  
Los valores válidos para marcar a los procesos como protegidos en EPE son:  
![Imagen de valores de seguriad en procesos protegidos](./images/protected_processes_values.png?raw=true "PPL Levels")
Podemos ver estos valores en ProcessExplorer, con la columna Protection. ProcessExplorer no nos mostrará como protegidos los procesos del kernel y no nos mostrará las librerias cargadas por los procesos protegisdos ya que no tiene permisos para acceder a esa información.  
 

### Creación de un proceso
* Apertura del ejecutable
* Creación de un objeto en el kernel para administrar esa imagen. Por ejemplo KPROCESS. (Executive process).
* Creación del hilo principal.
* Creación de un objeto en el kernel para administrar hilos. Por ejemplo KTHREAD o ETHREAD (Executive Thread).
* El kernel notifica a CSRSS que un nuevo proceso y un nuevo hilo ha sido creado por LPC.
* CSRSS crea sus manejadores y estructuras para poder administrar los procesos y los hilos.  
* Se continua con la creación del proceso y la ejecución de los hilos. 
   * Se cargan las DLLs necesarias y se inicializan.
   * Se llama a la función DllMain a través de DLL_PROCESS_ATTACH 
* Se ejecuta la función main del punto de entrada del binario (main/WinMain)2
El sistema operativo puede asignar más memoria a los procesos de la RAM que tiene, usando el fichero de paginación.  
Es el gestor de la memoria el que va asignando memoria física a la memoria virtual cuando necesita ejecutar información de esas páginas. 

#### C++
```c++
BOOL CreateProcessA(
  LPCSTR                lpApplicationName,
  LPSTR                 lpCommandLine,
  LPSECURITY_ATTRIBUTES lpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL                  bInheritHandles,
  DWORD                 dwCreationFlags,
  LPVOID                lpEnvironment,
  LPCSTR                lpCurrentDirectory,
  LPSTARTUPINFOA        lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation
);
```
* lpApplicationName: nombre del binario en system32, cuidado por los binarios necesarios en wow64.
* lpCommandLine: nombre del proceso a lanzar, deja que el sistema interprete que binario ejecutar.
* lpProcessAttributes: atributos de seguridad del proceso.
* lpThreadAttributes: atributos de seguridad de los hilos.
* bInheritHandles: decidir si el proceso hijo debe heredar la tabla de manejadores del padre.
* dwCreationFlags: parámetros en la creación del proceso, por ejemplo CreateSuspended para process hollowing. 
    * https://docs.microsoft.com/en-gb/windows/win32/procthread/process-creation-flags
* lpEnvironment: pasarle variables de entorno al proceso en la creación. Con nullptr hereda las del creador.
* lpCurrentDirectory: crea la variable CurrentDirectory util por ejemplo para llamar a librerías teniendo en cuenta esa ruta. Con nullptr hereda.
* lpStartupInfo: estructura STARTUPINFO ¿?
* lpProcessInformation: estructura PROCESS_INFORMATION que indica la información que entregará la función de vuelta.
    * dwprocessId: indicador único para el proceso creado.
    * dwTreaId: indicador único para el hilo creado.
    * hProcess: manejador para poder interactuar con el proceso creado.
    * hThreat: manejador para poder interactuar con el hilo creado.
    
```c++
#include <iostream>
#include <windows.h>
using namespace std;

int main(int argc, TCHAR arg[])
{
    PROCESS_INFORMATION pi;
    STARTUPINFO si = { sizeof(si) };
    TCHAR name[] = TEXT("notepad");
    BOOLEAN success = CreateProcess(nullptr, name, nullptr, nullptr, FALSE, 0, nullptr, nullptr, &si, &pi);

    if(success){
        cout << "PID: " << pi.dwProcessId << endl;
        cout << "HProcess: " << pi.hProcess << endl;
        WaitForSingleObject(pi.hProcess, INFINITE);
        DWORD code;
        GetExitCodeProcess(pi.hProcess, &code);
        cout << "Notepad has exited. Exit code=" << code << endl;       
    }
    else {
        cout << GetLastError() << endl;
    }
    return 0;
}
```

## Hilos
Los hilos son los encargados de ejecutar el código.  
Un hilo contiene:  
* El estado de los registros de la CPU.
* Sesión o modo de acceso asignado (user mode o kernel mode)
* Una pila con las variables, métodos... en el espacio del kernel y otra en el espacio del usuario. ¿?
* Un área de almacenamiento privado, llamado Thread Local Storage (TLS).
* Token de seguridad. (Opcional)
* Cola de mensajes y ventanas que el hilo ha creado. (Opcional)
* Prioridad. Número de 0-31. Prioridad de ejecución cuando sea programado. 31 es la mayor prioridad.
* Estado: Running, ready, waiting.  

En la parte del stack del hilo, podemos ver las llamadas desde el user mode al kernel mode.  
Un hilo termina cuando:
* El hilo devuelve un valor con return.
* Función ExitThread(Win32) forma correcta de hacerlo, se sale de las librerías que tenga implementadas...
* TerminateThread es la forma de forzar la finalización del hilo, sin que escriba en logs o termine las librerías inicializadas.  
### Creación de hilos
Se utilizará la función CreateThread(Win32).  
```c++
HANDLE CreateThread(
  LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  SIZE_T                  dwStackSize,
  LPTHREAD_START_ROUTINE  lpStartAddress,
  __drv_aliasesMem LPVOID lpParameter,
  DWORD                   dwCreationFlags,
  LPDWORD                 lpThreadId
);
```
* lpThreadAttributes: estructura de seguridad con el descriptor de seguridad del hilo, si es null no hereda del proceso.
* dwStackSize: tamaño de la pila del hilo, si se marca a 0 cogerá el tamaño por defecto.
* lpStartAddress: puntero a la función que tiene que ejecutar el hilo.
* lpParameter: puntero a los parámetros que se le pasan a la función.
* dwCreationFlags: flags de creación:
    * 0 : se ejecuta inmediatamente.
    * CREATE_SUSPENDED: crear el hilo en estado suspendido.
* lpThreadId: puntero a la variable que almacenará el identificador del thread una vez creado.
```c++
for(int i = 0; i < NUM_THREADS; i++) {
		data[i].first = first + i * delta;
		data[i].last = i == NUM_THREADS - 1 ? last : first + (i + 1) * delta - 1;
		hThread[i] = CreateThread(nullptr, 0, CalcPrimes, &data[i], 0, &id);
	}
  ```
### Prioridades de Hilos
La prioridad de los hilos es un valor que va desde el 1 al 31, siendo el 31 el más elevado.  
La prioridad 0 está reservada para la página 0 del hilo.  
La prioridad base es la prioridad asignada al proceso y es la asignada por defecto a los hilos del proceso. Por defecto 8.  
La prioridad se puede cambiar con:
* SetPriorityClass: cambia la prioridad base del proceso. Win32.  
* SetThreadPriority: cambia el puntero a la prioridad. Win32.  
* KeSetPriorityThread: cambia el valor de la prioridad a un valor absoluto. Kernel.  

En un proceso con Normal Priority Class, con la funcion SetthreadPriority puedes asignar +2, +1, -1, -2. Se pueden saturar los valores asignando 1 o 15. Estos son las llamadas prioridades normales. 
En un proceso con Above Normal Priority class la prioridad base es 10 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con High Priority class la prioridad base es 13 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con idle Priority class la prioridad base es 4 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con Realtime Priority class la prioridad base es 24 y se pueden realizar los mismos cambios que con el anterior pero la saturación será a los valores 16 y 31.  
Los hilos que están trabajando en primer plano, windows los aumenta cada poco tiempo con +2.  
![Imagen de thread priorities](./images/thread-priorities.png?raw=true "priorities")

Cuando el administrador de hilos tiene que elegir que hilo ocupará el procesador, únicamente tiene en cuenta el nivel de prioridad del proceso. En el caso de que varios hilos con la misma prioridad estén en estado de "ready", el primer hilo en llegar a la cola será ejecutado durante un periodo de tiempo, si después de ese periodo no ha terminado, pasará a la cola otra vez y entrará el siguiente hilo de la misma prioridad.  
Si un hilo se está ejecutando y entra en la cola un hilo de Prioridad Tiempo Real, el hilo que se estaba ejecutando se para, se le manda a la cola y se ejecuta el de alta prioridad.  
Si el equipo es multi procesador el algoritmo cambia.  
![Imagen de thread scheduling multi-cpu](./images/scheduling-mulicpu.png?raw=true "scheduling")
#### Quantum
Es la medida de tiempo que el procesador tiene en cuenta para cambiar de hilo, aunque no haya terminado la ejecución.  
El planificador tiene ticks cada 10 msec o 15 en el caso de los multiprocesadores. El tiempo de quantum en clientes de 2 ticks y en los servidores de 12 ticks.  
Se puede ver utilizando la herramienta clockres.exe.  
Se puede modificar cambiando la clave de registro: *HKLM\SYSTEM\CCS\Control\PriorityControl:Win32PrioritySepaaration*  
Es de 30 msec en clientes y 180 msec en servidores.  
El hilo que generó la ventana que está en primer plano, recibe un aumento del quantum al triple.
### Estados de los hilos
* Inicial (0, init): cuando el hilo es creado.
* listo (1/7, ready): hay un estado intermedio entre el 1 y el 7 que es deferred, no explicado.
* Espera (3, standby): es cuando el hilo va a ser el siguiente en ser ejecutado. Aunque lo normal es que el siguiente paso sea Running, puede pasar que otro hilo con mayor prioridad entre en la cola y de espera pase a listo.
* corriendo (2, running): cuando el procesador está ejecutando el código del hilo.
* termiando (4, terminate): cuando un hilo termina su ejecución pasa a este estado.
* esperando (5, waiting): cuando el hilo está esperando por algo, como una entrada por parte del usuario, pasa a este estado. Este hilo pasará a listo cuando ocurra lo que está esperando.
* Transición (6, transition): cuando un hilo está durante mucho tiempo en espera, el kernel lo pasa a este estado para ahorrar espacio en memoria.  
![Imagen de threads states](./images/threads-states.png?raw=true "states")

### Sincronización de hilos
Para el trabajo con hilos concurrentes, podemos usar parallel_for(C++) o Parallel.For(.NET)
El kernel genera un estado en los objetos llamados señalados (signaled) o no señalados (non-signaled) respecto al que, monitorizando los cambios de estado, podremos gestionar la sincronización.  
Se puede monitorizar los cambios mediante las funciones:
* Windows API
    * WainForSingleObject
    * WaitForMultipleObjects
* Kernel
    * KeWaitForSingleObject
    * KeWaitForMultipleObjects

Los tipos de objetos son: Procesos, hilos, eventos, mutex, semaforo, timer, fichero, ejecución I/O.  
Dependiendo del objeto, será señalado dependiendo de alguna condición.  
* **Process**  
    * El proceso termina  
* **Hilo**  
    * El hilo termina  
* **Mutex**  
    * El mutex queda libre  
* **Evento**  
    * La bandera del evento es lanzada  
* **Semaforo**  
    * El contador del semaforo es mayor de cero  
* **Fichero o puerto I/O**  
    * La operación se completa
* **Timer**  
    * El intervalo expira

#### Sección crítica (CRITICAL_SECTION)
Es un tipo de estructura de código utilizada para facilitar el control de concurrencia. Esta gestión se realiza en el modo de usuario y es más eficiente que el uso de Mutex.  
Un hilo puede entrar en la sección (EnterCriticalSection) y salir (LeaveCriticalSection) usando las APIs del usuario.  
En .NET se utilizan los lockers para esta funcionalidad.  

#### Exclusiones Mutuas (Mutex/Mutant)
Son algoritmos para evitar que más de un hilo entre en una sección de código crítica. EL proceso que está ocupando el mutex(WaitForSingleObject), es el dueño del mismo. El mutex se ejecuta en el modo del kernel.  
Son usados para evitar que varios hilos modifiquen un fichero a la vez por ejemplo. Cuando el hilo termina la ejecución de código en esa sección, se libera el mutex (ReleaseMutex) y si otro hilo está esperando por él, puede continuar la ejecución.  
Cuando se crea un mutex es necesario asignarle un nombre, cuando el proceso crea el manejador para el mutex, cada procesador crea su propio manejador y si no le asignamos un nombre, no habrá relación entre ellos y habrá colisiones.  

#### Semaforos (Semaphore)
Cuando el semaforo es iniciado(CreateSemaphore), se le asigna un numero de contador. Cuando un hilo quiere ejecutar una sección de código controlada por un semáforo, lanzará una petición al semaforo(WaitForSingleObject) y este, en el caso de tener su contador a más de 0, reducirá su contador en 1 y permitirá al hilo la ejecución de código. Si por el contrario, el semáforo está a 0 cuando el hilo pregunta, el hilo tendrá que esperar.  
Hay una mayor flexibilidad con los semáforos que con los mutex ya que, en el caso de los semáforos, puede liberar el semáforo (ReleaseSemaphore) un hilo diferente que el que lo ocupó.  

#### Eventos (Event/Notification)
Elemento binario.  
Cuando creas un evento o una notificación, el resto de objetos del sistema pueden quedar a la espera de un cambio de estado del evento y reaccionar en el caso de que el estado cambie. Por ejemplo, cuando se ejecuta el pagado de la máquina, todos los hilos están pendientes de ese evento y en el momento que se activa, proceden a cerrarse.  



## Manejadores (handles)
Es el kernel el único que puede obtener un puntero a un objeto y manipularlo. Desde la sesión de un usuario, para poder acceder a un objeto, es necesario un manejador que hace las veces de intermediario. Cuando obtienes un manejador, la dirección de acceso a él es añadida a tu tabla de manejadores y cuando dejas de necesitarlo y llamas a la función de closehandle, lo único que ocurre es el borrado de esa entrada, tú no sabes si el manejador es borrado o no, si otro proceso tiene abierto un manejador contra el mismo objeto se mantendrá activo, pero si el número de manejadores abiertos para un objeto se vuelve 0, el objeto se borra a sí mismo.  
Desde ProcessExplorer podemos ver los manejadores de un proceso y podemos activar la vista de ObjectAddress, esta dirección es la dirección del objeto en la sesión de kernel.  
La columna access indican las flags sobre qué podemos hacer con ese manejador en concreto.

## Trabajos (Jobs)
Esta estructura nos permite agrupar procesos relacionados entre sí.  
Por ejemplo, es útil para limitar el número máximo de memoria que pueden ocupar el grupo o para gestionar mejor la finalización de esos procesos.  
Cuando creamos el trabajo (CreateJobObject) nos generará un manejador que después usaremos para asignarle el grupo a un proceso creado (AssignProcessToJobObject). Podremos terminar todos los procesos de un trabajo (TerminateJobObject).



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

## Arquitectura en Windows 10
La arquietectura en Windows 10, ha tenido varios cambios.
![Imagen de arquitectura](./images/architecture-win10.PNG?raw=true "Arquitectura")

## Seguridad Basada en Virtualización (Virtualization Based Security)(Win10 Enterprise-WS2016)
Está caracteristica no está disponible en todas las versiones de Win10.
Se basa en la creación de dos niveles de virtualización en el sistema, uno confiable y otro no.
El VTL0 (Virtual Trust Level) es el que tiene privilegios "normales".
El VTL1 es el nivel de confianza para el sistema.
Ambos niveles disponene de un Modo Kernel y un Modo Usuario. ¿Sesión 0 y 1?. 
Lo que diferencia el VTL 1 es que es capaz de acceder a todo loq ue hay en VTL 0 pero no al revés.
Por debajo de ambos, tenemos al Hyper-V que es quién gestiona todos los permisos.
Esto permite que incluso un driver en VTL 0 no pueda acceder al kernel de VTL 1.
Para que esta tecnología pueda funcionar, se añadió la tecnología de Traducción de Direcciones de Segundo Nivel (SLAT), esto permite que el kernel en VTL 1 pueda crear direcciones intermedias que hacen que desde VTL 0 sea imposibible acceder a direcciones en VTL 1.
Otra tecnología que se implementó es I/O MMU (I/O Memory Management Unit), esto le permite al Hipervidor o al kernel de VTL 1, ocultar cosas en el espacio I/O. 

![Imagen de arquitectura Wow64](./images/VBS.PNG?raw=true "VBS")

### Credential Guard
Una de las caracteristicas que dependen de VBS es Credential Guard. Esta caracteristica permite almacenar las credenciales en el proceso "Lsaiso.exe" en vez del típico (Lsass.exe).
Lsaiso.exe está en VTL 1, de esta manera, incluso un proceso con privvilegios de system en VTL 0, no podrá acceder al espacio de memoria de Lsaiso.exe.

### Device Guard
Esta caracteristica, permite únicamente ejecutar aplicaciones confiables en el sistema.
Este control se hace a nivel de Hypervisor de tal manera que bypassearlo es muy complejo.

### Secure Driver Framework (SDF)
Es un framework que ha autorizado Microsoft para el desarrollo de drivel en VTL 1.

## Pico Processes
Este tipo de procesos, son procesos mínimos que únicamente contienen las direcciones de memoria del driver que controla estos procesos. (Pico provider)
Cualquier llamad al sistema, es recibida por este driver no por el manejador oportuno.
Esto permite por ejemplo que el Windows Subsystem for Linux, tenga su propio Pico Provider.

## Procesos importantes
### Secure Kernel (Win 10 Ent)
Representa al kernel en VTL 1.
Es análogo a System en VTL 0.
No dispone de todas las herramientas típicas del kernel, esas están en System, como por ejemplo la gestion de la memoria, gestión de hilos...


### Proceso inactivo (idle process)
Siempre tiene el PID 0.  
Es el proceso que ocupa los hilos del procesador cuando no hay tareas que pueda ocupar.  
No es un proceso real ya que no tiene un ejecutable asociado, espacio asignado.  

### Proceso del sistema (system process)
Siempre tiene el PID 4.  
Representa el espacio y los recursos del kernel.  
Los hilos del proceso son creaos por el propio kernel o por los drivers que tiene asociados. Nunca corre ningún hilo en el espacio de usuario.  
Los drivers crean hilos con la función PsCreateSystemThread.

### Administrador de sesiones (Session manager smss.exe)
Es el primer proceso que crea system.  
Es el encargado de crear las variables del sistema, lanza los procesos de los subsistemas (csrss.exe), lanzar copias de sí mismo en las sesiones nuevas, ese proceso de sí mismo creado, lanza winlogon y csrss en esa sesión y muere.  
Monitoriza los procesos de subsistemas y si alguno muere, genera un pantallazo azul.  

### Winlogon
Maneja los accesos y salidas de sesión de los usuarios.  
Si el proceso muere, el usuario pierde la sesión en la que está logado y todos los procesos de esa sesión mueren.    
Es el encargado de capturar la Secure Attention Sequence (SAS) Ctrl+Alt+Del.  
Es quien lanza el proceso LogonUI.exe para mostrar la ventana de login, puede ser reemplazada esa ventana.  
Envía la información capturada a LSASS, si la respuesta es satisfactoria, inicia la sesión del usuario.

### Local session Authenticatio SubSystem LSASS
Es el encargado de llamar al paquete de autenticación adecuado.  
Cuando recibe unas credenciales válidas, genera un token que representa el perfil de seguridad del usuario y se lo envía a winlogon de respuesta.  

### Service Control Manager (SCM services.exe)
Es el responsable de iniciar, parar e interactuar con los servicios.  
Puede iniciar servicios cuando arranca el sistema sin necesidad de una sesión interactiva abierta.  
Puede correr bajo usuarios especiales:
* LocalSystem
* NetworkService
* LocalService  
Tambien se pe

### Local Session manager (lsm.dll)
Importada en svchost.exe.  
Gestiona las sesiones de terminal.

## Wow64 (Windows on Windows 64)
Windows permite la ejecución de binarios de 32bits en una arquiectura de 64.  
Para ello, tiene un libreria de 32 bits de NtDll y otras dlls de conversión.
![Imagen de arquitectura Wow64](./images/win32-64.PNG?raw=true "Ficheros")  
Existen algunas limitaciones.  
Los procesos de 64 bits no pueden cargar una dll de 32 y viceversa.  
Algunas APIs no son soportadas por wow64.  

## Memory Compression (Win 10 Ent.)
Es un proceso "mínimo".  
Contiene la memoria comprimida en el espacio de memoria del usuario.  
No es mostrado por el Task Manager para evitar que los usuarios crean que les está consumiendo mucha memoria ese proceso.

## Segregación de Win32K.sys
Este driver, es el encargado de la interfaz gráfica en Windows.  
En la vesión del kernel NT 4, fue movido al kernel para mejorar eficiencia.  
En Windows 10, este driver se ha segregado par aumentar la seguridad y para aumentar la eficacia, creando tres versiones del mismo.
* Win32kMin.sys
* Win32KBase.sys
* Win32KFull.sys

## Procesos UWP (Universal Windows Platform)
Son las llamadas aplicaciones metro, strore, moder...  
Estas aplicaciones incluyen en un manifest la declaración de caracteristicas necesarias para su funcionamiento, como permisos de cámara o cosas así. El usuario puede aceptarlas al instalar la aplicación.  
La aplicación, unicamente puede tener acceso a aquello que ha declarado.  
Estas aplicaciones son ejecutadas con los permisos de AppContainer