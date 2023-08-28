---
layout: post2
language: es
title:  "Red Team:Enumeracion"
categories:  es Enumeracion
tags: [ "Red Team", "OSINT"]
---

## Índice
---
1. [Filiales](#filiales)
2. [Dominios e IP](#dominios-e-ips)
	1. [Herramientas de descubrimiento de dominios e IPs](#herramientas-descubrimiento-de-dominios-e-ips)
3. [Usuarios](#usuarios)
	1. [Herramientas de descubrimiento de usuarios](#herramientas-descubrimiento-de-usuarios)
4. [Servicios](#servicios)
	1. [Herramientas de descubrimiento de servicios](#herramientas-descubrimiento-de-servicios)
5. [Data leak](#data-leak)
	1. [Herramientas de descubrimiento de data leak](#herramientas-descubrimiento-de-leaks)
6. [Extra](#extra)

---

# Enumeración

Esta es la primera fase por la que se comienza en un ejercicio de Red Team, su importancia es bastante notoria, ya que afectará directamente al resto de pasos a seguir durante el ejercicio.

La enumeración consiste en obtener la mayor parte de información posible de un activo, desde filiales asociadas a la compañia, dominios e ips que engloban la compañia y sus filiales, hasta lista de usuarios y correos electronicos.

<br>
---

## Tipos de Enumeración
Para una forma mas sencilla de mostrar técnicas de obtencion de información, dividiremos la información por *tipos* de información.
1. ### Filiales
En esta parte intentaremos obtener el nombre de empresas filiales asociadas a la compañia principal, ya que, por lo general, existe una relación de confianza entre los dominios de las filiales y la empresa *madre*. Esta acción nos dara una mayor superficie de "ataque", ampliando el scope.
Para esto utilizaremos google dorks, introduciendo, por ejemplo:

	`filiales NOMBRE DE EMPRESA`

	![Image not found](/assets/img/filiales.png)

	De esta forma obtendrémos una lista de empresas asociadas a la compañia indicada, y se podrán añadir los activos de dichas filiales al scope.




2. ### Dominios e IPs
Esta parte consite en crear una superficie de activos a las que atacar, conociendo las ips y dominios asociadas a la empresa principal y sus filiales obtenidas en el paso anterior.

	1. ### Herramientas descubrimiento de dominios e IPs
		1. ### Dominios
		Para enumerar existen muchos tipo de herramientas que nos ayudaran en la obtencion de dominios, es importante tener encuenta la diferencia entre busqueda activa y pasiva, ya que si la compñaia posee una buena monitorización de los DNS, al realizar una enumearción de dominios o IPs de forma activa, podrían detectar que estan siendo enumerados.

			- ### Amass
			**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`
	
				Es aplicacion tiene diferentes opciones, bastante potente, y que puede hacer que de problemas al realizar consultas a google, ya que es uno de los buscadores que puede utilizar para obtener nombres de dominio.
				```help
				Subcommands: 

					amass intel - Discover targets for enumerations
					amass enum  - Perform enumerations and network mapping
					amass viz   - Visualize enumeration results
					amass track - Track differences between enumerations
					amass db    - Manipulate the Amass graph database
					amass dns   - Resolve DNS names at high performance
				```

				La opcion para la enumeracion de subdominios es  **enum**
				Una de las opciones que se puede utulizar es la forma passiva, esta conulta las fuentes publica, para obtener informacion de subdominios

				`amass enum -passive -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_dom_pas.png)
			
				Otra opcion muy utilizada es la opcion de **brute** que realiza un ataque de fuerza bruta al sub dominio, con -wordlist puedes utilizar un diccionario para

				`amass enum -passive -brute  -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_pas_brut.png)

				Las opciones activas se puede utilizar con **-active** y puede hacerse con la opcion brute o sin ella como la opcion **passive**
				
				`amass enum -active  -brute  -d alhkytran.xyz`
                                
				![Image Not Found!](/assets/img/amass_act_brut.png)

				`amass enum  -active -d alhkytran.xyz`
                               
				![Image Not Found!](/assets/img/amass_dom_act.png)

				La opcion db creara una base de datos con los resultados.

			- ### Massdns
			**Install**

				[https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8](https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8)

				Esta aplicacion saca los registros DNS de un dominio, para ello hay que facilitarle un "resolver", tiene su propia lista de resolvers, servidores DNS

				`massdns $HOME/test/dominio -r /$HOME/Tools/massdns/lists/resolvers.txt -o S -w $HOME/test/salida`
                                ![Image Not Found!](/assets/img/massdns.png)

				Es recomendable utilizar unas listas propias ya que las que estan por defecto pueden dar falsos positivos, es recomedable utilizar la herramienta  dnsvalidator para verificar que dns son optimos y cuales no. Se recomienda tirar este script mas de 30 veces para confirmar que el dns es bueno para todos
				https://github.com/vortexau/dnsvalidator

				**Importante** *-- Agradecimientos a Mario Bartolome por compartir esto conmigo --*
				Es importante tener una lista de DNS validos para realizar consultas de fuerza bruta, para ello podemos utilizar *dnsvalidator*, adjunto un script en bash que realiza una limpia de servidores no validos de la lista de DNS.
				```bash
					#!/bin/bash
					i=1
					source /home/user/script/env/bin/activate
					dnsvalidator -tL $1 --silent | tee /tmp/limpiar$i
				for i in $(seq 2 30)
				do
					j=$(expr $i - 1)
					lineas=$(cat /tmp/limpiar$j | wc -l)
					if [ $lineas == "0" ]; 
					then
						k=$(expr $j - 1)
						cat /tmp/limpiar$k > ./dnslistvalidated
						exit	
					else
						dnsvalidator -tL /tmp/limpiar$j --silent | tee /tmp/limpiar$i 
						fi
				done
				cat /tmp/limpiar30 > ./dnslistvalidated
				rm /tmp/limpiar*
				```


			- ### Subfinder
			Instalar, necesita la version de go 1.18 minimo, si no será imposible
			
				`go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`
		
				Esta herramienta de enumeracion de subdominios es muy rapida en comparacion con otros, como amass. La ejecucion de este comando se realiza de la siguiente forma
			
				`subfinder -d alhkytran.xyz -silent`

				![Image Not Found!](/assets/img/subfinder1.png)

				La flag silent nos evitara introducir basura en la salida, como el promp y demas, en caso de que luego queramos gestionar los datos que devuelve sera mas sencillo.
				Tambien se le puede añadir la flad -all, esta flag hara que la herramienta utilice todos los sources para obtener la informacion
			
				`subfinder -d hackerone.com -silent -all`

				![Image Not Found!](/assets/img/subfinder2.png)

		2. ### IPs
			Obtener el rango de ip nos sirve para identificar posibles activos dentro del rango sin dominio asociado, esto podria permitirnos acceder a posibles activos olvidados o en desuso de la empresa o de las filiales de la misma. Actualmente se utiliza el ASN para conocer los rangos de ips asociados a una compañia, podemos utilizar las siguienes herramientas para realizar esto.

			- ### ASNmap
				**Install**

				`go install github.com/projectdiscovery/asnmap/cmd/asnmap@latest`

				Esta herramienta realiza busquedas en el ASN, para obtener infornacion relacionada con el dominio, obteniendo el rango de ips que aparecen en estas listas asociadas a dichos dominios.

				Para el uso de la herramienta existen varias posibilidades de utilizacion de esta herrmaienta, una es facilitarle el dominio, de esta forma te ofrecera el rango de ips asociado a dicho dominio

				`asnmap -d alhkytran.xyz -silent` o `echo "alhkytran.xyz | asnmap -silent"`

				![Image Not Found!](/assets/img/asnmap1.png)

				Tambien se le puede pasar el nombre de la organizacion
				
				`echo "Telefonica" | asnmap  -silent`

				![Image Not Found!](/assets/img/asnmap2.png)

			- ### ASNlookup
				Este recurso es una web que realizar la busqueda a través de los ASN y nos facilita la información
				[https://asnlookup.com/](https://asnlookup.com/)

				Esta web te permite obtener informacion sobre una organizacion atraves de los ASN
				![Image Not Found!](/assets/img/asnlook1.png)

				Si se busca por el nombre de una empresa nos ofrece un resultado con los diferentes ASN asociados
				![Image Not Found!](/assets/img/asnlook2.png)

				Si se selecciona uno de estos, se pueden ver las ips de ese ASN
				![Image Not Found!](/assets/img/asnlook3.png)

			- ### Amass
				**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`

				Además de lo visto en la seccion de dominios, tambien es posible obtener rangos de ip con amaas, para ello suaremos <b>Intel -org</b>

				Esta opcion se puede utilizar para obtener rangos de ip de una organización, ips validad mediante el ASN.

				`amass intel -org "google"`
				
				![Image Not Found!](/assets/img/amass_asn_go.png)
				
				Otra forma de enumerar con asn y amass es:
				
				`amass intel -active -asn `

		3. ### Favicon
			- ### Favfreak
				**Install**

				`https://github.com/devanshbatham/FavFreak`

				Con esta herramienta se puede obtener del hash del favicon y utilizarlo para obtener webs que esten utilizando el mismo favicon que el indicado, por lo que los dominios e/o IPs que se obtengan serán parte de los activos de la empresa a auditar, ya que usan el mimso logo corporativo.
				Para utilizar esta herramienta debemos facilitarle la web o las webs desde donde adquirir el fichero y podrá realizar una busqueda en Shodan otorgandote los nuevos deominios. *Para el el dork que utiliza necesita cuenta de pago de shodan*
			
				`cat dominios.txt | python ./favfreak.py --shodan`
				![Image Not Found!](/assets/img/favfreak.png)

				Una vez obtenido las querys de shodan seria buscarlas y ver el resultado de las mismas.

				![Image Not Found!](/assets/img/shodanfav.png)

			- ### Manual
				Una vez mas agradecimiento a *-- Mario Bartolome --* el cual me explico una forma de obtener las ip, incluso conocer la que estan detrás de Akamai, mediante el Favicon de la empresa.

				Lo primero es descargar el favicon de la web y obtener el hash del mismo, esto ultimo podemos realizarlo a través del siguiente script
				```python3
				import mmh3
				import requests
				import codecs
				headers={"User-Agent":"Mozilla/5.0 (X11; Linux i686; rv:108.0) Gecko/20100101 Firefox/108.0"}
				img=request.get("url/favicon.ico",headers=headers)
				bas64=codecs.encode(img.content,"base64")
				mhh3.hash(bas64)
				```
				Esto devolvera un numero que deberemos introducir en shodan con el siguiente comando para obtener informacion sobre el dominio. Para esto necesitas cuenta en shodan.
				
				![Image Not Found!](/assets/img/favicon1.png)

				`http.favicon.hash:2042744103`

				![Image Not Found!](/assets/img/favicon2.png)

3. ### Usuarios
Como parte de la enumeración hacer un listado de posibles usuarios, o credenciales que esten en alguna base de datos que se pueda consultar como [dehashed](https://www.dehashed.com/) ayduaria para hacer ataques como password spraying o intentar conseguir un acceso a traves de una VPN.

	1. ### Herramientas descubrimiento de usuarios
		- ### Leaks 
			Además de conseguir los datos de los leaks en bases de datos como la comentada anteriormente puede tratar de buscarse leaks accidentales por parte de desarrolladores, en ocasiones pueden verse credenciales en webs como pastebin o gihub, google...
			Para localizar estos leaks se utilizara google dorks, similares a estos
			```dorks
			site:pastebin.com "NomEmpresa.com"
			site:github.com "NomEmpresa.com"
			site:googleapis.com "NomEmpresa.com"
			site:drive.google.com "NomEmpresa.com"
			.
			.
			.
```
			![Image Not Foud](/assets/img/paste1.png)
			
			Además de las dorks existen herrmaientas de pago que realizan esta tarea por nosotors, bucando con cuentas de pago por las webs. Admeás existe unas herramientas que buscan en la Deep Web datos leak de la empresa que le indiquemos, como hace *"pwndb"*, el cual actualmente no funciona ya que este tipo de servicio lo van cerrando, pero no esta demas buscar por si te pudieran ofrecer informacion.

		- ### CrossLinked
			Esta herramienta realiza consulta a los buscadores para obtener un listado de los usuarios en liknedin pertenecientes a la empresa que se le indique, y puedes generar los correos con el formato que se lo indiques, por ellos siempre es recomendable encontrar un ejemplo de dirección de correo antes de lanzar el script.
			**Instalación**
		
			`pip3 install crosslinked`

			Un ejemplo de su ejecución seria, generar un listado de *NomEmpresa* con la primera letra del nombre punto apellido, el ejemplo descrito se obtendira con el siguiente comando:
			
			`crosslinked -f {f}.{last}@NomEmpresa.com NomEmpresa`

			![Image Not Found](/assets/img/crosslinked.png)

4. ### Servicios
	1. ### Herramientas descubrimiento de servicios
		- ### Shodan
			Es un motor de busqueda en el que tiene los puertos abiertos de los servicios ofrecidos en internet de parcticamente la totalidad de internet. Esto puede consultarse realizando busquedas concretas para localizar activos concretos, de esta forma evitas realizar escaneos sobre un activo con ips propias ya que estan en la base de datos de shodan.

			![Image Not Found](/assets/img/shodan1.png)

			Puede verse el puerto y el resultado de una peticion "curl -I", se pueden realizar consultas concretas por tipo de version

			![Image Not Found](/assets/img/shodan2.png)

		- ### Fofa
			https://en.fofa.info/

			Fofa es un servicio similar a Shodan, por lo que pueden consultarse de forma simultanea para contrastar informacion.

			![Image Not Found](/assets/img/fofa.png)

			La información facilitada es similar a la que ofrece shodan. dandonos lo puertos habiertos y el resultado de una peticion "curl -I"

		- ### Nmap
			Nmap es la tipica herramienta utilizada para concoer los puertos abiertos de un servidor.
			Con opciones como
			```
				-sV da informacion obtenida a través del banner
				-p- realiza el escaneo sobre todos los puertos
				--top-ports 2000 realiza el escaneo sobre lo 2000 puertos mas habituales, se puede modificar el numero
			```

			Exite con -sT -sS -sU o -sn seleccionar el tipo de escaneo, el envio de paquetes
			```
				-sT mandara paquetes por TCP completando la comunicacion
				-sS realiza el escaneo en modo stealth, para no ser detectado, actualmente se detecta por cualquier IDS, realiza una conexion a medias,.
				-sU manda paquetes UDP, para servicios com o DNS que utilizan este protocolo.
				-sn algunos dispositivos tienen el protocolo ICMP desactivado, por lo que no contestan a ese protocolo, con -sn se obvia el ping para continuar con el escaneo
			```

			![Image Not Found](/assets/img/nmap.png)

			En el ejemplo podemos observar que tiene el puerto 80 u el 8082 abiertos, filtered indica que puede tener el puerto abierto pero algun firewall esta bloqueando la conexion.
			Al hacerlo con el -sV vemos que el puerto 8082 esta dando el servicio ssh, normalmente ubicado en el puerto 22.

5. ### Data leak
	1. ### Herramientas descubrimiento de leaks
		En ocasiones, por errores o despistes, se publica infomación sensible de una aplicación o de la propia empresa en repositorios de acceso público, pudiendo encontrar desde direcciciones de correo validas, hasta usuarios y contraseñas de domino.
		- ### .git
			Una forma habitual es encontrar una web o aplicación publica en la que se haya dejado la carpeta .git, en esta carpeta puede encontrarse información que puede ser utilizada. En ocasiones al acceder a estos recursos devuelve un 403, esto no quiere decir que no pueda accederse a la informacion con herramientas como git-dumper

			`pip install git-dumper`

			Para lanzarlo deberemos indicar la url y la carpeta, esto descargará el contenido de la carpeta, que deberemos revisar en busca de información sensisble que pueda ser de utilizada.

			`git-dumper https://revistas.urldetest.es/ .git`

			![Image Not Found](/assets/img/git-dump.png)
			
			Por ejemplo con `grep -rE "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b"` buscaremos de forma recursiva direcciones de correo dentro de los archivos extraidos

			![Image Not Found](/assets/img/git-dump2.png)

		- ### pastebin
		- ### Metadatos	
			Los metadatos son informacion adjunta a los ficheros, ya sean png, jpeg, pdf, doc... En estos datos se puede ver ubicaciones donde se ha obtenido una fotografia, software utilizado para generar el fichero o usuarios validos que creo el fichero. Existen herramientas, como foca o metagoofil que revisan los ficheros asociados a un dominio y extran dichos metadatos que puedes consultar.
			*Metagoofil*() es una herramienta creada en python2 que descarga los ficehros que se el indique en extension y extrae todos los metadatos de estos ficheros. Para su ejecucción,

			`python metagoofil.py  -d DOMINIO -t xdoc,doc,pdf -l 250 -n 50 -o endesafiles -f results.html`

			![Image Not Found](/assets/img/metagoofil.png)

			El proyecto esta abandonado y actualmente realiza consultas a google, que al realizar varias de forma seguida, google banea y la aplicacion deja de obtener ficheros. 
		
		- ### Noseyparker
                        [NoseyParker](https://github.com/praetorian-inc/noseyparker) esta herramienta es muy útil ya que realiza un escaneo del repositorio que le facilitemos y trata de obtener toda la informacion sensible que haya en su interior.
                        Puede realizarse un escaneo en busca de todos los repositorios de un usuario, para ello utilizaremos el comando de la siguiente forma:

                        `noseyparker github repos list --user alhkytran`


                        ![Image Not Foud](/assets/img/noseyparker1.png)

                        Si conocemos el repositorio podemos indicarle que lo escanee y que guarde la informacion en una base de datos

                        `noseyparker scan --datastore np.test --git-url https://github.com/praetorian-inc/noseyparker`

                        ![Image Not Foud](/assets/img/noseyparker2.png)

                        Se puede consultar el contenido de la base de datos creada

                        ![Image Not Foud](/assets/img/noseyparker3.png)

                        Para consultar la información que a obtenido la herramienta lanzaremos el comando

                        `noseyparker report --datastore np.test`

                        ![Image Not Foud](/assets/img/noseyparker4.png)

                        ![Image Not Foud](/assets/img/noseyparker5.png)


6. ### Extra
	Durante la creación del post se realizó al DefCon 2023 donde se presento la herramienta de reconocimiento [DorXNG](https://github.com/ResearchandDestroy/DorXNG) esta herramienta recopila información de diferentes fuentes, incluso a traves de la deep web usando tor. [aqui](https://github.com/ResearchandDestroy/DorXNG/blob/main/query.lst) tienen un ejmeplo de posibles queris que se pueden utilizar
como punto de partida. Por ejemplo con la petición

	`DorXNG.py -q "DB_USERNAME filetype:env" -d test.db`

	![Image Not Foud](/assets/img/dorxng1.png)

	Buscara en archivos con nombre *env* el contenido *DB_USERNAME* y lo almacenará en la base de datos *test.db*, al acceder a uno de los resultados puede verse lo siguiente:

	![Image Not Foud](/assets/img/dorxng2.png)

	Otra query que se puede relizar es `DorXNG.py -q 'intitle:"index of" "pass.txt"' -d test.db` para que busque archivos llamados pass.txr y se pueden encontrar cosas interesantes.

	![Image Not Foud](/assets/img/dorxng3.png)

	Viendose cosas como esta

	![Image Not Foud](/assets/img/dorxng4.png)	![Image Not Foud](/assets/img/dorxng5.png)

	Se puede jugar con las querys como si fuera google dorks.

	Para consultgar bases de datos ya alimentadas, podremos hacerlo con
	
	`DorXNG.py -D "DB_USERNAME" -d test.db`

	![Image Not Foud](/assets/img/dorxng6.png)

---
<br>
## References
- https://blog.intigriti.com/2021/06/08/hacker-tools-amass-hunting-for-subdomains/
- https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8
- https://github.com/praetorian-inc/noseyparker
- https://github.com/ResearchandDestroy/DorXNG


## Agradecimientos
- [Mario Bartolome](https://github.com/MarioBartolome) - *Por la incansable ayuda e inspiración* 
