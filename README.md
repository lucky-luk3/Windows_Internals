# Windows_Internals
Windows es un sistema orientado a objetos, todo en el sistema operativo es un objeto.  
Los objetos están en el espacio de memoria y solamente son accesibles mediante el kernel.  

## Procesos
El proceso es un objeto de administración que provee la información necesaria para ejecutar un programa.  
Un proceso no corre ni ejecuta código, eso lo hacen los hilos.  
El proceso no conoce ni necesita las direcciones físicas de la memoria, únicamente conoce las direcciones virtuales y es el gestor de memoria del sistema operativo el que hace esa traducción.  
Un proceso contiene:  
* Un espacio de memoria privado.
* Una cantidad de memoria física asociada 8working set)
* El fichero ejecutable, un enlace al fichero en disco que contiene la función main.
* Una tabla del indicador de los manejadores de objetos del kernel asociados a él.  
* Un token de seguridad que contiene los descriptores de seguridad del proceso.    
* Prioridad, esa prioridad será la asignada a los hilos del proceso.  
Un proceso termina cuando:
* Todos sus hilos terminan.
* Uno de los hilos llama a ExitProcess(Win32), esto ocurre avaces como rutina de cierre del hilo principal, de tal manera que cerrando ese hilo se cierra todo el proceso. Esta es la manera correcta de terminar un proceso ya que da tiempo al proceso y a las dlls a termianr por si mismas e incluso escribir en logs, liberar memoria o guardar infromación del fichero.  
* uno de los hilos llama a TerminateProcess(Win32), es la manera no recomendada de termianr un proceso. Esta llamada puede hacerse desde el exterior de los hilos del proceso. 
Algunas páginas de la memoria física pueden ser compartidas, por ejemplo cuando se ejecutan dos instancias de un mismo programa, la página en la memoria física que almacena el ejecutable o dlls compartidas, puede ser compartida.  
La columna del Administrador de Tareas nos muestra la memoria ocupada por un proceso, pero únicamente aquella que el proceso tiene marcada como memoria privada. Para ver la memoria completa que ocupa un proceso, necesitaremos activar la columna Tamaño de asignación. 
### Creación de un proceso
* Apertura del ejecutable
* Creación de un objeto en el kernel para administrar esa imagen. Por ejemplo KPROCESS. (Executive process).
* Creación del hilo principal.
* Creación de un objeto en el kernel para administrar hilos. por ejemeplo KTHREAD o ETHREAD (Executive Thread).
* El kernel notifica a CSRSS que un nuevo proceso y un nuevo hilo ha sido creado por LPC.
* CSRSS crea sus manajadores y estructuras para poder administarr los procesos y los hilos.  
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
* dwCreationFlags: parametros en la creación del proceso, por ejemplo CreateSuspended para process hollowing. 
    * https://docs.microsoft.com/en-gb/windows/win32/procthread/process-creation-flags
* lpEnvironment: pasarle variables de entorno al proceso en la creación. Con nullptr hereda las del creador.
* lpCurrentDirectory: crea la variable CurrentDirectory util por ejemplo para llamar a librerias teniendo enc uenta esa ruta. Con nullptr hereda.
* lpStartupInfo: estructura STARTUPINFO ¿?
* lpProcessInformation: estructura PROCESS_INFORMATION que indica la información que entregará la función de vuelta.
    * dwprocessId: indicador único para el proceso creado.
    * dwTreaId: indicador único para el hilo creado.
    * hProcess: manejador para poder interactuar con el proceso creado.
    * hThreat: manejador para poder interactuar con el hilo creado.
    
```c++
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

## Procesos importantes
### Proceso inactivo (idle process)
Siempre tiene el PID 0.  
Es el proceso que ocupa los hilos del procesador cuando no hay tareas que pueda ocupar.  
No es un proceso real ya que no tiene un ejecutable asociado, espacio asignado.  

### Proceso del sistema (system process)
Siempre tiene el PID 4.  
Representa el espacio y los recursos del kernel.  
Los hilos del proceso son creaos por el propio kernel o por los drivers que tiene asociados. Nunca corre ningun hilo en el espacio de usuario.  
Los drivers crean hilos con la función PsCreateSystemThread.

### Administrador de sesiones (Session manager smss.exe)
Es el primer proceso  que crea system.  
Es el encargado de crear las variables del sistema, lanza los procesos de los subsistemas (csrss.exe), lanzar copias de si mismo en las sesiones nuevas, ese proceso de si mismo creado, lanza winlogon y csrss en esa sesión y muere.  
Monitoriza los procesos de subsistemas y si alguno muere, genera un pantallazo azul.  

### Winlogon
Maneja los accesos y salidas de sesión de los usuarios.  
Si el proceso muere, el usuario pierde la sesión en la que está logado y todos los procesos de esa sesión mueren.    
Es el encargado de capturar la Secure Attention Sequence (SAS) Ctrl+Alt+Del.  
Es quien lanza el proceso LogonUI.exe para mostrar la ventana de login, puede ser reemplazada esa ventana.  
Envia la información capturada a LSASS, si la respuesta es satisfactoria, incial la sesión del ususario.

### Local session Authenticatio SubSystem LSASS
Es el encargado de llamar al paquete de autenticación adecuado.  
Cuando recibe unas credenciales válidas, genera un token que representa el perfil de seguridad del usuario y se lo envia a winlogon de respuesta.  

### Service Control Manager (SCM services.exe)
Es el respnsable de iniciar, parar e interactuar con los servicios.  
Puede iniciar servicios cuando arranca el sistema sin necesidad de una sesió interactiva abierta.  
Puede correr bajo usuarios especiales:
* LocalSystem
* NetworkService
* LocalService  
Tambien se pe

### Local Session manager (lsm.dll)
Importada en svchost.exe.  
gestiona las sesiones de terminal.

## Wow64 (Windows on Windows 64)
Windows permite la ejecución de binarios de 32bits en una arquiectura de 64.  
Para ello, tiene un libreria de 32 bits de NtDll y otras dlls de conversión.
![Imagen de arquitectura Wow64](./images/win32-64.PNG?raw=true "Ficheros")
Existen algunas limitaciones.  
Los procesos de 64 bits no pueden cargar una dll de 32 y viceversa.  
Algunas APIs no son soportadas por wow64.  

