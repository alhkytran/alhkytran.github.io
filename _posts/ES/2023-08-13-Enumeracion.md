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

La enumeración consiste en obtener la mayor parte de información posible de un activo, desde filiales asociadas a la compañía, dominios e IPs que engloban la compañía y sus filiales, hasta lista de usuarios y correos electrónicos.

<br>
---

## Tipos de Enumeración
Para una forma mas sencilla de mostrar técnicas de obtención de información, dividiremos la información por *tipos* de información.
1. ### Filiales
En esta parte intentaremos obtener el nombre de empresas filiales asociadas a la compañía principal, ya que, por lo general, existe una relación de confianza entre los dominios de las filiales y la empresa *madre*. Esta acción nos dará una mayor superficie de "ataque", ampliando el scope.
Para esto utilizaremos google dorks, introduciendo, por ejemplo:

	`filiales NOMBRE DE EMPRESA`

	![Image not found](/assets/img/filiales.png)

	De esta forma obtendrémos una lista de empresas asociadas a la compañia indicada, y se podrán añadir los activos de dichas filiales al scope.




2. ### Dominios e IPs
Esta parte consiste en crear una superficie de activos a las que atacar, conociendo las IPs y dominios asociadas a la empresa principal y sus filiales obtenidas en el paso anterior.

	1. ### Herramientas descubrimiento de dominios e IPs
		1. ### Dominios
		Para enumerar existen muchos tipo de herramientas que nos ayudaran en la obtencion de dominios, es importante tener encuenta la diferencia entre busqueda activa y pasiva, ya que si la compñaia posee una buena monitorización de los DNS, al realizar una enumearción de dominios o IPs de forma activa, podrían detectar que estan siendo enumerados.

			- ### Amass
			**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`
	
				Es aplicación tiene diferentes opciones, bastante potente, y que puede hacer que de problemas al realizar consultas a google, ya que es uno de los buscadores que puede utilizar para obtener nombres de dominio.
				```help
				Subcommands: 

					amass intel - Discover targets for enumerations
					amass enum  - Perform enumerations and network mapping
					amass viz   - Visualize enumeration results
					amass track - Track differences between enumerations
					amass db    - Manipulate the Amass graph database
					amass dns   - Resolve DNS names at high performance
				```

				La opción para la enumeración de subdominios es  **enum**
				Una de las opciones que se puede utilizar es la forma pasiva, esta consulta las fuentes publica, para obtener información de subdominios


				`amass enum -passive -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_dom_pas.png)
			
				Otra opción muy utilizada es la opción de **brute** que realiza un ataque de fuerza bruta al sub dominio, con -wordlist puedes utilizar un diccionario para

				`amass enum -passive -brute  -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_pas_brut.png)

				Las opciones activas se puede utilizar con **-active** y puede hacerse con la opción brute o sin ella como la opción **passive**
				
				`amass enum -active  -brute  -d alhkytran.xyz`
                                
				![Image Not Found!](/assets/img/amass_act_brut.png)

				`amass enum  -active -d alhkytran.xyz`
                               
				![Image Not Found!](/assets/img/amass_dom_act.png)

				La opcion db creara una base de datos con los resultados.

			- ### Massdns
			**Install**

				[https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8](https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8)

				Esta aplicación saca los registros DNS de un dominio, para ello hay que facilitarle un "resolver", tiene su propia lista de resolvers, servidores DNS

				`massdns $HOME/test/dominio -r /$HOME/Tools/massdns/lists/resolvers.txt -o S -w $HOME/test/salida`
                                ![Image Not Found!](/assets/img/massdns.png)

				Es recomendable utilizar unas listas propias ya que las que están por defecto pueden dar falsos positivos, es recomendable utilizar la herramienta  dnsvalidator para verificar que dns son óptimos y cuales no. Se recomienda tirar este script mas de 30 veces para confirmar que el dns es bueno para todos
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
		
				Esta herramienta de enumeración de subdominios es muy rápida en comparación con otros, como amass. La ejecución de este comando se realiza de la siguiente forma
			
				`subfinder -d alhkytran.xyz -silent`

				![Image Not Found!](/assets/img/subfinder1.png)

				La flag silent nos evitara introducir basura en la salida, como el promp y demas, en caso de que luego queramos gestionar los datos que devuelve sera mas sencillo.
				También se le puede añadir la flag -all, esta flag hará que la herramienta utilice todos los sources para obtener la información
			
				`subfinder -d hackerone.com -silent -all`

				![Image Not Found!](/assets/img/subfinder2.png)

		2. ### IPs
			Obtener el rango de IP nos sirve para identificar posibles activos dentro del rango sin dominio asociado, esto podría permitirnos acceder a posibles activos olvidados o en desuso de la empresa o de las filiales de la misma. Actualmente se utiliza el ASN para conocer los rangos de IPs asociados a una compañía, podemos utilizar las siguientes herramientas para realizar esto.

			- ### ASNmap
				**Install**

				`go install github.com/projectdiscovery/asnmap/cmd/asnmap@latest`

				Esta herramienta realiza búsquedas en el ASN, para obtener información relacionada con el dominio, obteniendo el rango de ips que aparecen en estas listas asociadas a dichos dominios.

				Para el uso de la herramienta existen varias posibilidades de utilización de esta herramienta, una es facilitarle el dominio, de esta forma te ofrecerá el rango de IPs asociado a dicho dominio

				`asnmap -d alhkytran.xyz -silent` o `echo "alhkytran.xyz | asnmap -silent"`

				![Image Not Found!](/assets/img/asnmap1.png)

				También se le puede pasar el nombre de la organización
				
				`echo "Telefonica" | asnmap  -silent`

				![Image Not Found!](/assets/img/asnmap2.png)

			- ### ASNlookup
				Este recurso es una web que realizar la búsqueda a través de los ASN y nos facilita la información
				[https://asnlookup.com/](https://asnlookup.com/)

				Esta web te permite obtener información sobre una organización a través de los ASN
				![Image Not Found!](/assets/img/asnlook1.png)

				Si se busca por el nombre de una empresa nos ofrece un resultado con los diferentes ASN asociados
				![Image Not Found!](/assets/img/asnlook2.png)

				Si se selecciona uno de estos, se pueden ver las ips de ese ASN
				![Image Not Found!](/assets/img/asnlook3.png)

			- ### Amass
				**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`

				Además de lo visto en la sección de dominios, también es posible obtener rangos de IP con amaas, para ello sumaremos <b>Intel -o</b>

				Esta opción se puede utilizar para obtener rangos de ip de una organización, IPs válidas mediante el ASN.

				`amass intel -org "google"`
				
				![Image Not Found!](/assets/img/amass_asn_go.png)
				
				Otra forma de enumerar con asn y amass es:
				
				`amass intel -active -asn `

		3. ### Favicon
			- ### Favfreak
				**Install**

				`https://github.com/devanshbatham/FavFreak`

				Con esta herramienta se puede obtener del hash del favicon y utilizarlo para obtener webs que estén utilizando el mismo favicon que el indicado, por lo que los dominios e/o IPs que se obtengan serán parte de los activos de la empresa a auditar, ya que usan el mismo logo corporativo.
				Para utilizar esta herramienta debemos facilitarle la web o las webs desde donde adquirir el fichero y podrá realizar una búsqueda en Shodan otorgándote los nuevos dominios. *Para el el dork que utiliza necesita cuenta de pago de shodan*
			
				`cat dominios.txt | python ./favfreak.py --shodan`
				![Image Not Found!](/assets/img/favfreak.png)

				Una vez obtenido las querys de shodan seria buscarlas y ver el resultado de las mismas.

				![Image Not Found!](/assets/img/shodanfav.png)

			- ### Manual
				Una vez mas agradecimiento a *-- Mario Bartolome --* el cual me explico una forma de obtener las ip, incluso conocer la que están detrás de Akamai, mediante el Favicon de la empresa.

				Lo primero es descargar el favicon de la web y obtener el hash del mismo, esto último podemos realizar lo a través del siguiente script
				```python3
				import mmh3
				import requests
				import codecs
				headers={"User-Agent":"Mozilla/5.0 (X11; Linux i686; rv:108.0) Gecko/20100101 Firefox/108.0"}
				img=requests.get("url/favicon.ico",headers=headers)
				bas64=codecs.encode(img.content,"base64")
				aux=mmh3.hash(bas64)
				print(aux)
				```
				Esto devolverá un numero que deberemos introducir en shodan con el siguiente comando para obtener información sobre el dominio. Para esto necesitas cuenta en shodan.
				
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
			
			Además de las dorks existen herramientas de pago que realizan esta tarea por nosotros, buscando con cuentas de pago por las webs. Además existe unas herramientas que buscan en la Deep Web datos leak de la empresa que le indiquemos, como hace *"pwndb"*, el cual actualmente no funciona ya que este tipo de servicio lo van cerrando, pero no esta demás buscar por si te pudieran ofrecer información.

		- ### CrossLinked
			Esta herramienta realiza consulta a los buscadores para obtener un listado de los usuarios en liknedin pertenecientes a la empresa que se le indique, y puedes generar los correos con el formato que se lo indiques, por ellos siempre es recomendable encontrar un ejemplo de dirección de correo antes de lanzar el script.
			**Instalación**
		
			`pip3 install crosslinked`

			Un ejemplo de su ejecución seria, generar un listado de *NomEmpresa* con la primera letra del nombre punto apellido, el ejemplo descrito se obtendría con el siguiente comando:
			
			`crosslinked -f {f}.{last}@NomEmpresa.com NomEmpresa`

			![Image Not Found](/assets/img/crosslinked.png)

4. ### Servicios
	1. ### Herramientas descubrimiento de servicios
		- ### Shodan
			Es un motor de búsqueda en el que tiene los puertos abiertos de los servicios ofrecidos en internet de prácticamente la totalidad de internet. Esto puede consultarse realizando búsquedas concretas para localizar activos concretos, de esta forma evitas realizar escaneos sobre un activo con IPs propias ya que están en la base de datos de shodan.

			![Image Not Found](/assets/img/shodan1.png)

			Puede verse el puerto y el resultado de una petición "curl -I", se pueden realizar consultas concretas por tipo de versión

			![Image Not Found](/assets/img/shodan2.png)

		- ### Fofa
			https://en.fofa.info/

			Fofa es un servicio similar a Shodan, por lo que pueden consultarse de forma simultanea para contrastar información.

			![Image Not Found](/assets/img/fofa.png)

			La información facilitada es similar a la que ofrece shodan. dándonos lo puertos abiertos y el resultado de una petición "curl -I"

		- ### Nmap
			Nmap es la típica herramienta utilizada para conocer los puertos abiertos de un servidor.
			Con opciones como
			```
				-sV da información obtenida a través del banner
				-p- realiza el escaneo sobre todos los puertos
				--top-ports 2000 realiza el escaneo sobre lo 2000 puertos mas habituales, se puede modificar el número
			```

			Exite con -sT -sS -sU o -sn seleccionar el tipo de escaneo, el envío de paquetes
			```
				-sT mandara paquetes por TCP completando la comunicación
				-sS realiza el escaneo en modo stealth, para no ser detectado, actualmente se detecta por cualquier IDS, realiza una conexión a medias,.
				-sU manda paquetes UDP, para servicios com o DNS que utilizan este protocolo.
				-sn algunos dispositivos tienen el protocolo ICMP desactivado, por lo que no contestan a ese protocolo, con -sn se obvia el ping para continuar con el escaneo
			```

			![Image Not Found](/assets/img/nmap.png)

			En el ejemplo podemos observar que tiene el puerto 80 u el 8082 abiertos, filtered indica que puede tener el puerto abierto pero algún firewall esta bloqueando la complexion.
			Al hacerlo con el -sV vemos que el puerto 8082 esta dando el servicio ssh, normalmente ubicado en el puerto 22.

5. ### Data leak
	1. ### Herramientas descubrimiento de leaks
		En ocasiones, por errores o despistes, se publica información sensible de una aplicación o de la propia empresa en repositorios de acceso público, pudiendo encontrar desde direcciones de correo validas, hasta usuarios y contraseñas de domino.
		- ### .git
			Una forma habitual es encontrar una web o aplicación publica en la que se haya dejado la carpeta .git, en esta carpeta puede encontrarse información que puede ser utilizada. En ocasiones al acceder a estos recursos devuelve un 403, esto no quiere decir que no pueda accederse a la información con herramientas como git-dumper

			`pip install git-dumper`

			Para lanzar lo deberemos indicar la url y la carpeta, esto descargará el contenido de la carpeta, que deberemos revisar en busca de información sensible que pueda ser de utilizada.

			`git-dumper https://revistas.urldetest.es/ .git`

			![Image Not Found](/assets/img/git-dump.png)
			
			Por ejemplo con `grep -rE "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b"` buscaremos de forma recursiva direcciones de correo dentro de los archivos extraídos

			![Image Not Found](/assets/img/git-dump2.png)

		- ### Metadatos	
			Los metadatos son informacion adjunta a los ficheros, ya sean png, jpeg, pdf, doc... En estos datos se puede ver ubicaciones donde se ha obtenido una fotografia, software utilizado para generar el fichero o usuarios validos que creo el fichero. Existen herramientas, como foca o metagoofil que revisan los ficheros asociados a un dominio y extran dichos metadatos que puedes consultar.
			*Metagoofil*() es una herramienta creada en python2 que descarga los ficheros que se el indique en extension y extrae todos los metadatos de estos ficheros. Para su ejecución,

			`python metagoofil.py  -d DOMINIO -t xdoc,doc,pdf -l 250 -n 50 -o endesafiles -f results.html`

			![Image Not Found](/assets/img/metagoofil.png)

			El proyecto esta abandonado y actualmente realiza consultas a google, que al realizar varias de forma seguida, google banea y la aplicación deja de obtener ficheros. 
		
		- ### Noseyparker
                        [NoseyParker](https://github.com/praetorian-inc/noseyparker) esta herramienta es muy útil ya que realiza un escaneo del repositorio que le facilitemos y trata de obtener toda la información sensible que haya en su interior.
                        Puede realizarse un escaneo en busca de todos los repositorios de un usuario, para ello utilizaremos el comando de la siguiente forma:

                        `noseyparker github repos list --user alhkytran`


                        ![Image Not Foud](/assets/img/noseyparker1.png)

                        Si conocemos el repositorio podemos indicarle que lo escanee y que guarde la información en una base de datos

                        `noseyparker scan --datastore np.test --git-url https://github.com/praetorian-inc/noseyparker`

                        ![Image Not Foud](/assets/img/noseyparker2.png)

                        Se puede consultar el contenido de la base de datos creada

                        ![Image Not Foud](/assets/img/noseyparker3.png)

                        Para consultar la información que a obtenido la herramienta lanzaremos el comando

                        `noseyparker report --datastore np.test`

                        ![Image Not Foud](/assets/img/noseyparker4.png)

                        ![Image Not Foud](/assets/img/noseyparker5.png)


6. ### Extra
	Durante la creación del post se realizó al DefCon 2023 donde se presento la herramienta de reconocimiento [DorXNG](https://github.com/ResearchandDestroy/DorXNG) esta herramienta recopila información de diferentes fuentes, incluso a través de la deep web usando tor. [aqui](https://github.com/ResearchandDestroy/DorXNG/blob/main/query.lst) tienen un ejemplo de posibles queris que se pueden utilizar
como punto de partida. Por ejemplo con la petición

	`DorXNG.py -q "DB_USERNAME filetype:env" -d test.db`

	![Image Not Foud](/assets/img/dorxng1.png)

	Buscara en archivos con nombre *env* el contenido *DB_USERNAME* y lo almacenará en la base de datos *test.db*, al acceder a uno de los resultados puede verse lo siguiente:

	![Image Not Foud](/assets/img/dorxng2.png)

	Otra query que se puede realizar es `DorXNG.py -q 'intitle:"index of" "pass.txt"' -d test.db` para que busque archivos llamados pass.txr y se pueden encontrar cosas interesantes.

	![Image Not Foud](/assets/img/dorxng3.png)

	Viéndose cosas como esta

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
