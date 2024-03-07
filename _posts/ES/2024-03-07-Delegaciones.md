---
layout: post2
language: es
title:  "Kerberos: Delegaciones"
categories:  es Kerberos Delegaciones
tags: [ "Red Team", "AD", "Delegaciones"]
---

## Índice
---
1. [Introduccion](#kerberos:-delegaciones)
2. [Unconstrained delegation](#unconstrained-delegation)
	1. [Como detectar la caracteristica.](#como-detectar-la-caracteristica)
	2. [Como abusar de Unconstrained Delegation](#como-abusar-de-unconstrained-delegation)
3. [Constrained delegation](#constrained-delegation)
4. [Resource-based constrained delegations (RBCD)](#resource-based-constrained-delegation)
	1. [RBCD sin posibilidad de usar *ServicePrincipalName* (SPN)](#rbcd-sin-spn)
5. [Posibles remediaciones](#posibles-remediaciones)

---

# Kerberos: Delegaciones

1. ### Introducción
Las delegaciones es una caracterisitca que tienen los entornos de directorio activo de windows, su implementacion tiene la funcion de que un servicio pueda realizar acciones en nombre de un usuario que se halla autenticado en dicho servicio. Por ejemplo, si un servidor web con autenticación, *Kerberos*, tiene una funcionalidad de subida de fichero, la cual se realiza en un servidor de ficheros, para poder hacer que el servicio web pueda realizar acciones en nombre del usuario autenticado, en el servidor de ficheros, utiliza las delegaciones de windows.    
 
<p align="center">
   <img src="/assets/img/delegacion.png">
</p>

 
Una mala configuración de esta caracteristica puede hacer que un ususario pueda realizar acciones en nombre de otros usuarios sin el consentimiento de estos mismos. Existen tres tipos de Abusos de dicha cualidad. **Unconstrained delegation**, **Constrained delegation**, **Resource- base constrained delegation(RBCD)**
<br>

---

2. ### Unconstrained delegation
En este caso la delegación permite que el usuairo pueda impersonar a cualquier usuario que haya logieado en el servicio, esto es por que se almacena una copia del TGT al realizar la autenticación, esto permite que puedan generarse TGS en su nombre para ser utilizados en otros servios, por lo que si se consigue realizar conexito, y exite una copia del TGT de un administrador de dominio en el equipo podría utilizarse para realizar accciones como *DCsync* en nombre de dicho administrador de dominio.

Esto es debido a el servicio que pose la caracteristica *SeEnableDelegation* activa, que se consigue activando la opcion *Confiar en este equipo para la delegación a cualquier sercicio (solo Kerberos)*

<p align="center">
   <img src="/assets/img/SeEnableDelegation.png">
</p>


Esto modifica el *userAccountControl* y añade la opción *TrustedForDelegation* que permite aprovecharnos de esta delegación.

<p align="center">
   <img src="/assets/img/TrustedForDelegation.png">
</p>



  1. #### Como detectar la caracteristica
Para detectar la vulnerabilidad mediante ldap search necesitamos, y debemos filtrar por maquinas y por el valor *524288* en el *userAccountControl*. Un ejemplo de peticion sería.
`ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))"`

A lo que aparecerá una lista con las maquinas que tienen activadas esta funcionalidad

<p align="center">
   <img src="/assets/img/Uncosldapsearch.png">
</p>

  2. #### Como abusar de Unconstrained Delegation


3. ### Constrained delegation
4. ### Resource-based constrained delegation 
  1. ### RBCD sin SPN

5. ### Posibles remediaciones

---

<br>
## References
- *https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf*a
- *https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/domain-compromise-via-unrestricted-kerberos-delegation*
- *https://www.tarlogic.com/blog/kerberos-iii-how-does-delegation-work/*
 


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
