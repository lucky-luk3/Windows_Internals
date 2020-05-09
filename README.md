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
Estos procesos únicamente pueden cargar unas librerías firmadas con un certificado especial.
#### Procesos Protegidos Light
Fueron introducidos en Windows 8.1, ofrecían varios niveles de protección en los procesos.  
Inicialmente fueron pensados para el software antivirus y la mayoría de los procesos del sistema fueron protegidos con esta medida.  
Esto permite que un proceso con privilegios de administrador no tenga por qué tener permisos de acceso a un proceso protegido.  
Esta protección es únicamente en User Mode, así que un administrador podría ejecutar un driver en Kernel Mode y tener acceso a todo.  

![Imagen de niveles de seguridad en procesos protegidos](./images/protected_processes_levels.png?raw=true "PPL Levels")  
Los valores válidos para marcar a los procesos como protegidos en EPE son:  
![Imagen de valores de seguriad en procesos protegidos](./images/protected_processes_values.png?raw=true "PPL Levels")
Podemos ver estos valores en ProcessExplorer, con la columna Protection. ProcessExplorer no nos mostrará como protegidos los procesos del kernel y no nos mostrará las librerías cargadas por los procesos protegidos ya que no tiene permisos para acceder a esa información.  
 

### Creación de un proceso
* Apertura del ejecutable
* Creación de un objeto en el kernel para administrar esa imagen. Por ejemplo, KPROCESS. (Executive process).
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
En la parte del Kernel, el hilo contiene dos estructuras:
* ETHREAD: principal estructura del hilo, maneja la mayoría de la información del hilo.
* KTHREAD: es la primera estructura del ETHREAD. 

En la parte del stack del hilo, podemos ver las llamadas desde el user mode al kernel mode.  

Un hilo termina cuando:
* El hilo devuelve un valor con return.
* Función ExitThread(Win32) forma correcta de hacerlo, se sale de las librerías que tenga implementadas...
* TerminateThread es la forma de forzar la finalización del hilo, sin que escriba en logs o termine las librerías inicializadas.  

En Windows 10 y Server 2016 se añadió la opción de asignarle un nombre a los hilos. Este nombre no puede ser utilizado para acceder al
hilo, es únicamente para debugging.
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

En un proceso con Normal Priority Class, con la función SetthreadPriority puedes asignar +2, +1, -1, -2. Se pueden saturar los valores asignando 1 o 15. Estos son las llamadas prioridades normales. 
En un proceso con Above Normal Priority class la prioridad base es 10 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con High Priority class la prioridad base es 13 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con idle Priority class la prioridad base es 4 y se pueden realizar los mismos cambios que con el anterior.  
En un proceso con Realtime Priority class la prioridad base es 24 y se pueden realizar los mismos cambios que con el anterior pero la saturación será a los valores 16 y 31.  
Los hilos que están trabajando en primer plano, Windows los aumenta cada poco tiempo con +2.  
![Imagen de thread priorities](./images/thread-priorities.png?raw=true "priorities")

### Planificación de hilos
Cuando el administrador de hilos tiene que elegir que hilo ocupará el procesador, únicamente tiene en cuenta el nivel de prioridad del proceso. En el caso de que varios hilos con la misma prioridad estén en estado de "ready", el primer hilo en llegar a la cola será ejecutado durante un periodo de tiempo, si después de ese periodo no ha terminado, pasará a la cola otra vez y entrará el siguiente hilo de la misma prioridad.  
Si un hilo se está ejecutando y entra en la cola un hilo de Prioridad Tiempo Real, el hilo que se estaba ejecutando se para, se le manda a la cola y se ejecuta el de alta prioridad.  
Si el equipo es multiprocesador el algoritmo cambia.  
![Imagen de thread scheduling multi-cpu](./images/scheduling-mulicpu.png?raw=true "scheduling")
#### Quantum
Es la medida de tiempo que el procesador tiene en cuenta para cambiar de hilo, aunque no haya terminado la ejecución.  
El planificador tiene ticks cada 10 msec o 15 en el caso de los multiprocesadores.  
El tiempo de quantum en clientes de 2 ticks y en los servidores de 12 ticks.  
Se puede ver utilizando la herramienta clockres.exe.  
Se puede modificar cambiando la clave de registro: *HKLM\SYSTEM\CCS\Control\PriorityControl:Win32PrioritySepaaration*  
Es de 30 msec en clientes y 180 msec en servidores.  
El hilo que generó la ventana que está en primer plano, recibe un aumento del quantum al triple.
### Estados de los hilos
* Inicial (0, init): cuando el hilo es creado.
* listo (1/7, ready): hay un estado intermedio entre el 1 y el 7 que es deferred, no explicado. Esta es la cola de procesos que quieren ser ejecutados.
    * Antes de Windows 8, cada procesador tenia su propia cola de procesos en estado ready.
    * Después de Windows 8, hay una cola única de procesos en estado listo para mejorar la eficiencia.
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
    * El contador del semáforo es mayor de cero  
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
Son usados para evitar que varios hilos modifiquen un fichero a la vez, por ejemplo. Cuando el hilo termina la ejecución de código en esa sección, se libera el mutex (ReleaseMutex) y si otro hilo está esperando por él, puede continuar la ejecución.  
Cuando se crea un mutex es necesario asignarle un nombre, cuando el proceso crea el manejador para el mutex, cada procesador crea su propio manejador y si no le asignamos un nombre, no habrá relación entre ellos y habrá colisiones.  

#### Semáforos (Semaphore)
Cuando el semáforo es iniciado(CreateSemaphore), se le asigna un numero de contador. Cuando un hilo quiere ejecutar una sección de código controlada por un semáforo, lanzará una petición al semáforo(WaitForSingleObject) y este, en el caso de tener su contador a más de 0, reducirá su contador en 1 y permitirá al hilo la ejecución de código. Si por el contrario, el semáforo está a 0 cuando el hilo pregunta, el hilo tendrá que esperar.  
Hay una mayor flexibilidad con los semáforos que con los mutex ya que, en el caso de los semáforos, puede liberar el semáforo (ReleaseSemaphore) un hilo diferente que el que lo ocupó.  

#### Eventos (Event/Notification)
Elemento binario.  
Cuando creas un evento o una notificación, el resto de objetos del sistema pueden quedar a la espera de un cambio de estado del evento y reaccionar en el caso de que el estado cambie. Por ejemplo, cuando se ejecuta el pagado de la máquina, todos los hilos están pendientes de ese evento y en el momento que se activa, proceden a cerrarse.  

### Afinidad de procesador
La afinidad de procesador proporciona la posibilidad de elegir que procesador queremos que ejecute los hilos
de un proceso concreto. Esta función es a nivel de proceso y los hilos no pueden escapar de este control.  
En Windows 10 se incluyó la posibilidad de que los hilos puedan usar un procesador determinado.  
SetProcessDefaultCpuSets asigna un procesador a todos los hilos.  
SetThreadSelectedCpuSets asigna un procesador a ciertos hilos.  
Esta funcionalidad es utilizada por el modo de juego añadido en Windows 10 RS2, esto permite que los procesos del 
sistema no afecten a la eficiencia de los hilos del juego.

### Reparto justo de CPU
En un equipo con varias sesiones activas, puede ocurrir que una sesión genere hilos con prioridad alta
que provoquen que el procesador esté únicamente ocupado con ellos, no permitiendo ocupar la CPU
a los procesos de otras sesiones.  
Esto no aplica a los procesos del sistema como smss.  
Para gestionar esto se creó DFSS (Dynamic Fair Share Scheduling), que puede ser activado.  
Cuando se crea un grupo de reparto, para una sesión concreta por ejemplo, en esta se marca una política en la que se asignara 
un porcentaje de uso de procesador o una prioridad para los procesos e hilos que contiene.  
También se crea un contador que controla el tiempo de ejecución de CPU de los hilos para su grupo.
Se habilita añadiendo el valor "EnableCpuQuota=1" en las claves:
* HKLM\System\Current Control Set\Control\Session Manager\Quota System
* HKLM\Software\Policies\Microsoft\Windows\Session Manager\Quota System

## Manejadores (handles)
Es el kernel el único que puede obtener un puntero a un objeto y manipularlo. Desde la sesión de un usuario, para poder acceder a un objeto, es necesario un manejador que hace las veces de intermediario. Cuando obtienes un manejador, la dirección de acceso a él es añadida a tu tabla de manejadores y cuando dejas de necesitarlo y llamas a la función de closehandle, lo único que ocurre es el borrado de esa entrada, tú no sabes si el manejador es borrado o no, si otro proceso tiene abierto un manejador contra el mismo objeto se mantendrá activo, pero si el número de manejadores abiertos para un objeto se vuelve 0, el objeto se borra a sí mismo.  
Desde ProcessExplorer podemos ver los manejadores de un proceso y podemos activar la vista de ObjectAddress, esta dirección es la dirección del objeto en la sesión de kernel.  
La columna access indican las flags sobre qué podemos hacer con ese manejador en concreto.  
Se crearon los CPU Sets para poder asignar diferentes procesadores para los hilos que, por ejemplo, crea el sistema de manera independiente sobre el proceso que ha 
creado el desarrollador, como svchost.  
 

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


## Seguridad Basada en Virtualización (Virtualization Based Security)(Win10 Enterprise-WS2016)
Está característica no está disponible en todas las versiones de Win10.  
Se basa en la creación de dos niveles de virtualización en el sistema, uno confiable y otro no.  
El VTL0 (Virtual Trust Level) es el que tiene privilegios "normales".  
El VTL1 es el nivel de confianza para el sistema.  
Ambos niveles disponen de un Modo Kernel y un Modo Usuario. ¿Sesión 0 y 1?.  
Lo que diferencia el VTL 1 es que es capaz de acceder a todo lo que hay en VTL 0 pero no al revés.  
Por debajo de ambos, tenemos al Hyper-V que es quién gestiona todos los permisos.  
Esto permite que incluso un driver en VTL 0 no pueda acceder al kernel de VTL 1.  
Para que esta tecnología pueda funcionar, se añadió la tecnología de Traducción de Direcciones de Segundo Nivel (SLAT), esto permite que el kernel en VTL 1 pueda crear direcciones intermedias que hacen que desde VTL 0 sea imposible acceder a direcciones en VTL 1.  
Otra tecnología que se implementó es I/O MMU (I/O Memory Management Unit), esto le permite al Hipervisor o al kernel de VTL 1, ocultar cosas en el espacio I/O.  

![Imagen de arquitectura Wow64](./images/VBS.PNG?raw=true "VBS")

### Credential Guard
Una de las características que dependen de VBS es Credential Guard. Esta característica permite almacenar las credenciales en el proceso "Lsaiso.exe" en vez del típico (Lsass.exe).  
Lsaiso.exe está en VTL 1, de esta manera, incluso un proceso con privilegios de system en VTL 0, no podrá acceder al espacio de memoria de Lsaiso.exe.  

### Device Guard
Esta característica, permite únicamente ejecutar aplicaciones confiables en el sistema.  
Este control se hace a nivel de Hypervisor de tal manera que bypassearlo es muy complejo.  

### Secure Driver Framework (SDF)
Es un framework que ha autorizado Microsoft para el desarrollo de driver en VTL 1.  

## Procesos UWP (Universal Windows Platform)
Son las llamadas aplicaciones metro, strore, moder...  
Estas aplicaciones incluyen en un manifest la declaración de características necesarias para su funcionamiento, como permisos de cámara o cosas así. El usuario puede aceptarlas al instalar la aplicación.  
La aplicación, únicamente puede tener acceso a aquello que ha declarado.    
Estas aplicaciones son ejecutadas con los permisos de AppContainer.  
En Process Explorer, los procesos que tienen Package Name, son procesos UWP.  
No está documentado por Microsoft cómo crear este tipo de procesos, si utilizamos el command line del Process Explorer, nos saldrá un error.  
Una de las maneras de lanzar estos procesos es mediante la aplicación MetroManager.  


## Minimal Processes
Estos procesos no cargan NtDll u otra librería de subsistema.  
No disponen de un espacio de memoria de usuario.  
No tienen PEB o TEBs.  
No es posible crear estos procesos desde el user mode.  
El espacio de memoria es creado y usado únicamente por hilos del kernel. 

## Pico Processes
Son procesos mínimos soportan drivers.  
Únicamente contienen las direcciones de memoria del driver que controla estos procesos (Pico provider).  
Cualquier llamad al sistema, es recibida por este driver no por el manejador oportuno.  
Esto permite por ejemplo que el Windows Subsystem for Linux, tenga su propio Pico Provider.  
Estos procesos son creados en el arranque.

![Imagen de arquitectura Wow64](./images/summary_process_types.png?raw=true "summary_process_types")

## Procesos importantes
### Secure Kernel (Win 10 Ent)
Representa al kernel en VTL 1.
Es análogo a System en VTL 0.
No dispone de todas las herramientas típicas del kernel, esas están en System, como por ejemplo la gestión de la memoria, gestión de hilos...


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
Windows permite la ejecución de binarios de 32bits en una arquitectura de 64.  
Para ello, tiene un librería de 32 bits de NtDll y otras dlls de conversión.  
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
En la versión del kernel NT 4, fue movido al kernel para mejorar eficiencia.  
En Windows 10, este driver se ha segregado para aumentar la seguridad y para aumentar la eficacia, creando tres versiones del mismo.
* Win32kMin.sys
* Win32KBase.sys
* Win32KFull.sys

# Trabajos.  Jobs
Son objetos del kernel para manejar uno o más procesos.  
Provee auditoria sobre esos procesos como tiempo de ejecución, memoria consumida... 
Los procesos hijos creados dentro de un trabajo, automáticamente se asocian a ese trabajo, a menos que se marque la bandera CREATE_BREAKWAY_FROM_JOB en la creación y el trabajo lo permita.  
Limites que se pueden ajustar en un trabajo:
* Número máximo de procesos activos
* Consumo de CPU
    * Por trabajo y tiempo
    * Por proceso y tiempo
    * Windows 10 introdujo un ratio en el control de la CPU
* Consumo de memoria
* Consumo de ancho de banda de red
* Control de manejadores, por ejemplo, no permitir a esos procesos que se comuniquen con procesos fuera del trabajo, acceso al portapapeles...  

## Trabajos anidados
En Windows 8 se introdujo el concepto de jerarquía de trabajos, siendo posible la anidación de trabajos.  
Los trabajos hijo pueden ser más restrictivos que el trabajo padre pero no menos.  

### API
CreateJobObject, OpenjobObject: crea o abre un objeto de trabajo por su nombre.
AssignProcessToJobObject: añade un proceso a un trabajo.  
SetInformationJobObject: marcar información sobre un trabajo concreto.  
QueryInformationJobObject: consultar información sobre el objeto trabajo.  
TerminateJobObject: termina todos los procesos asociados a un objeto trabajo.

# Silos
Los silos en Windows, es la manera que encontró Microsoft de implementar contenedores en Windows.  
Los silos son un tipo de trabajo que ofrece un aislamiento superior aun entre los procesos que están en su interior, 
ofreciendo por ejemplo:
* Un espacio de nombres privado.
* Un sistema de ficheros virtualizado propio, realizando redirecciones.
* Ids de procesos privados dentro del silo.
* Virtualización del registro.  

Los drivers se comparten entre todos los silos, de tal manera que la modificación de un driver afectaría a todos los silos.  
![Imagen de arquitectura Silos](./images/silos.png?raw=true "Silos") 

# Seguridad en Windows
Winlogon es el proceso encargado de gestionar el login de los usuarios, además es quien gestiona el Ctrl+Alt+Del. Para mostrar la ventana de login, 
winlogon usa el proceso LogonUI. El encargado de recibir la información de estos procesos es el Securoty Reference Monitor, que forma parte del kernel.  
Lsass es el proceso encargado de la gestión de las credenciales, dispone de varios servicios:
 * Active Directory, comunicación con Active Directory.
 * NetLogon, comunicaciones seguras sobre la red.
 * LSA Server, almacenar y gestionar la politica local de seguridad.
 * SAM Server, almacena infromación sobre los usuarios locales y las politicas de seguridad relacionadas con usuarios y grupos.
 * Msv1_0.dll, libreria para el login local o la comunicación con el controlador de dominio en versiones antiguas
 * Kerberos.dll, comunicación con el controlador de dominio.
 * Event Logder, encargado de la generación de logs.
LSAISO es la nueva caracteristica para el almacenamiento de las credenciales con seguridad basada en virtualización.  
![Imagen de security components](./images/security_components.png?raw=true "Security components") 

## Secuencia de login
1. Winlogon inicia LogonUI para conseguir la infromación del login usando un Credential Provider.
    1. Winlogon crea un esritorio para la ventana de login al que únicamente System tiene acceso.  
    2. Cuando se pulsa SAS (Secure Attention Sequence), winlogon lanza este escritorio.
    3. El manejador del teclado deshabilita todos los hooks cuando detecta SAS.
    4. El control sobre SAS es del proceso que lo creo, que siempre es winlogon.
2. Winlogon envia esa información a Lsass.
3. Lsass recibe la información.
    1. Si es un login local o una versión antigua de Windows, utiliza MSVC1_0.
    2. Se es un login contra un dominio, utiliza Kerberos.
4. Analiza la respuesta.
    1. Si no es satisfactorio, se le mostrará al usuario.
    2. Si es satisfactorio, Lsass coge los privilegios de la cuenta de la base de datos de seguridad local o del Active Directory.
5. Lsass crea un token para esa sesión que identifica el contexto de seguridad del usuario.
6. Winlogon asocia este token al primer proceso de la sesión, todos los procesos siguientes copiaran este token. Normalmente este proceso es userinit.exe que es el encargado de lanzar otros procesos como Explorer.exe.  
![Imagen de logon sequence](./images/logon_sequence.png?raw=true "Logon sequence") 

## SAM Database
Esta base de datos forma parte del registro de Windows. HKLM\SAM\SAM.  
Únicamente el usuario System puede acceder al contenido. Aunque podemos añadir permisos si somos administradores, no es recomendable.  
Podemos lanzar el editor del registro con PsExec de Sysinternals.  
````bash
C:\Tools\Sysinternals>PsExec.exe -s -i regedit
````
* -s, para lanzar el proceso con permisos de system
* -i, al ser un proceso de system, se ejecuta por defecto en la sesión 0, el -i es para que se ejecute en la sesión 1 de manera interactiva.  

Esto nos abrirá un editor del registro en el que se podrá ver el contenido de SAM.

## Credential Provider
En las versiones antiguas de Windwos (anterior a Vista) el proceso encargado del login y de mostarr la interfaz genraba multitud de errores que provocaban caidas del sistema.  
LogonUI.exe es el encargado de gestioanr la ventana del login, si un método de login genera una excepción, LogonUI termina y automaticamente crea otro proceso de LogonUI.  
Permite la creación de proveedores de servicios de login por terceros.  
LogonUI carga del registro los proveedores de login como:
* authui.dll, Microsoft provides Interactive
* Smart-cardcredentialprovider.dll, Smartcard
* FaceCredentialProvider.dll, Windows Hello




#Investigar
LPC comunicación  
COM object comunicación

