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
	3. [Recomendaciones de mitigacion](#recomendaciones-para-mitigar-el-problema)
2. [Constrained delegation](#constrained-delegation)
3. [Resource-based constrained delegations (RBCD)](#resource-based-constrained-delegation)
	1. [RBCD sin posibilidad de usar *ServicePrincipalName* (SPN)](#rbcd-sin-spn)
4. [Posibles remediaciones](#posibles-remediaciones)

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

En este caso la delegación permite que el usuairo pueda impersonar a cualquier usuario que haya logueado en el servicio, esto es por que se almacena una copia del TGT al realizar la autenticación, esto permite que puedan generarse TGS en su nombre para ser utilizados en otros servios, por lo que si se consigue realizar conexito, y exite una copia del TGT de un administrador de dominio en el equipo podría utilizarse para realizar accciones como *DCsync* en nombre de dicho administrador de dominio.

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
`ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))"`

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

  3. ### Recomendaciones para mitigar el problema

Para evitar ser objetivo de este ataque es importante deshabilitar las delegaciones siempre que sea posible, en caso de que sea necesario seria recomendable revisar que usuarios poseen permisos para realizar estas delegaciones.

Otra medida es habilitar la opción `La cuenta es importante y no se puede delegar`para las cuentas que tienen privilegios elevados

<p align="center">
   <img src="/assets/img/importante.png">
</p>




2. ## Constrained delegation
3. ## Resource-based constrained delegation 
  1. ### RBCD sin SPN

4. ## Posibles remediaciones

---

<br>
## References
- *https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf*a
- *https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/domain-compromise-via-unrestricted-kerberos-delegation*
- *https://www.tarlogic.com/blog/kerberos-iii-how-does-delegation-work/*
 


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
