---
layout: post2
language: en
title:  "Red Team:Enumeration"
categories:  en Enumeration
tags: [ "Red Team", "OSINT"]
---

## Index
---
1. [Subsidiaries](#subsidiaries)
2. [Domains and IP](#domains-and-ips)
	1. [Domain and IP discovery tools](#domain-and-ip-discovery-tools)
3. [Users](#users)
	1. [Users discovery tools](#users-discovery-tools)
4. [Services](#services)
	1. [Services discovery tools](#service-discovery-tools)
5. [Data leak](#data-leak)
	1. [Data leak descovery tools](#data-leak-discovery-tools)
6. [Extra](#extra)

---

# Enumeration

This is the first step in a Red team exercise, it's very importan because al the next steps depend for the enumeration results.

The enumeration try to get all the information as is posible about an assets, from the subsidiaries, domains and IPs from the subsidiaries and main company to the username and email address.

<br>
---

## Enumeration Types
For a easy way to show technique about get information, we split the information in *types* of information.
1. ### Subsidiaries
In this part we try to get the name of the subsidiares compnaies. It's common that the main domain and the subsidiaries domains have trust relationship beteween them. The result of this will give us a scope to "attack".
For this we will use google dorks, as an example:

	`subsidiaries COMPANY NAME`
	`filiales NOMBRE DE EMPRESA`

	![Image not found](/assets/img/filiales.png)

	The result fo this query gives us a list with the companies associated to the main, and we can add this assets to the scope.




2. ### Domains and IPs
In this part we try to create an assets area to attack, we have to get the domains and IPs associated to the main companies and its subsidiaries gotten in the previus step

	1. ### Domain and IP discovery tools
		1. ### Domains
		For discover domains are a lot of different tools to help us. It's important know the differences about active search and passive search, because if the company has a goo monitoring in the DNS queries, whe you try to enumeate the IPs and domains, they could detected that someone are listing them.

			- ### Amass
			**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`
	
				The application is very powerfull and has differents options. Sometimes could give you problems with google, because use this searcher to get a domains name.
				```help
				Subcommands: 

					amass intel - Discover targets for enumerations
					amass enum  - Perform enumerations and network mapping
					amass viz   - Visualize enumeration results
					amass track - Track differences between enumerations
					amass db    - Manipulate the Amass graph database
					amass dns   - Resolve DNS names at high performance
				```

				The subdomain enumeration option is **enum**
				One options is the passive search, this user the public resources to get the information about subdomains


				`amass enum -passive -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_dom_pas.png)
			
				Other  option is the **brute**, used to do a bruteforce attack to get subdomains, you need the flag `-wordlist` to use a dictionary.

				`amass enum -passive -brute -d alhkytran.xyz`

				![Image Not Found!](/assets/img/amass_pas_brut.png)

				The active options could be used with the flagn **-active** and you cand user **brute** or not as in the **passive** option.
				
				`amass enum -active  -brute  -d alhkytran.xyz`
                                
				![Image Not Found!](/assets/img/amass_act_brut.png)

				`amass enum  -active -d alhkytran.xyz`
                               
				![Image Not Found!](/assets/img/amass_dom_act.png)

				The option `db` create a database to check the results.

			- ### Massdns
			**Install**

				[https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8](https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8)

				This application is used to get the DNS registers from a domain. To do this the application need a "resolver", the tool has its own resolvers list, DNS servers

				`massdns $HOME/test/dominio -r /$HOME/Tools/massdns/lists/resolvers.txt -o S -w $HOME/test/salida`
                                ![Image Not Found!](/assets/img/massdns.png)

				It's recommended use your own list for resolvers, because the default one could give you false popsitves. To create a valid DNS list you can use `dnsvalidator` to check if the DNS is working well or not. Use dnsvalidator close 30 times to check well if the DNS it's valid or not.
				https://github.com/vortexau/dnsvalidator

				**Important** *-- Thanks to Mario Bartolome for share this with me --*
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
			To install this application we need, the go version 1.18 or higher.
			
				`go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`
		
				This tool to subdomain enumeration is too fast if you compare with the previus. The command usage is:
			
				`subfinder -d alhkytran.xyz -silent`

				![Image Not Found!](/assets/img/subfinder1.png)

				The flag `silent` clean the results of the application, removing promt. This do easier use the result after the application execution. 
				Also is possible use the flag -all, this do that the tool use all the resources to get the information. 
			
				`subfinder -d hackerone.com -silent -all`

				![Image Not Found!](/assets/img/subfinder2.png)

		2. ### IPs
			Get the company IP range is usefull to identify the scope and know assets without domain assigned. Somtimes you can get forgotten assets to check and find vulnerabilities for the main company or for his subsidiaries. 
			To get this  information you can use ASN data, some tools use this to get the information that we are looking for.

			- ### ASNmap
				**Install**

				`go install github.com/projectdiscovery/asnmap/cmd/asnmap@latest`

				This tool search in the ASN data to get information about a domain, getting the IP range associated to this domains.

				The use for this tool has different possibilities, the easier it's give the domain name, and give you the range IP for this domain.

				`asnmap -d alhkytran.xyz -silent` o `echo "alhkytran.xyz | asnmap -silent"`

				![Image Not Found!](/assets/img/asnmap1.png)

				Also, you can give the organiztion name.
				
				`echo "Telefonica" | asnmap  -silent`

				![Image Not Found!](/assets/img/asnmap2.png)

			- ### ASNlookup
				This is a free web resource that you can use it to search in the ASN data all the information that you want.
				[https://asnlookup.com/](https://asnlookup.com/)

				This web give you all the information about ASN data.
				![Image Not Found!](/assets/img/asnlook1.png)

				If you search the comapny name the respond give you different information getting from the ASN data.
				![Image Not Found!](/assets/img/asnlook2.png)

				If you click in the result you could see the range IP associatted.
				![Image Not Found!](/assets/img/asnlook3.png)

			- ### Amass
				**Install**

				`go install -v github.com/OWASP/Amass/v3/...@master`

				In addition to what we have seen in the domain section, it's possible get the IP range with amass, for this we add <b>Intel -o</b>

				You can use this option to get the IP range for a company with the ASN data.

				`amass intel -org "google"`
				
				![Image Not Found!](/assets/img/amass_asn_go.png)
				
				Other way to enumeration with asin and amass is:
				
				`amass intel -active -asn `

		3. ### Favicon
			- ### Favfreak
				**Install**

				`https://github.com/devanshbatham/FavFreak`

				With this tool is possible get the favicon hash and use this hash to search all the web, in shodan, with this favicon, you can assume that this web are part of the companybecause use the same logo icon.
				To use it, we have to add the web or webs where get the favicon iamge, then use the result in shodan.  *To use the dork you need the shodan pay account*
			
				`cat dominios.txt | python ./favfreak.py --shodan`
				![Image Not Found!](/assets/img/favfreak.png)

				The queries that you have to use in Shodan with the hash is this:

				`http.favicon.hash:$HASHFAVICON`
				![Image Not Found!](/assets/img/shodanfav.png)

			- ### Manual
				Again, thanks to *-- Mario Bartolome --*, he explained me the way to get the IP and know the real IP throught Akamain, with the Favicon.

				The first thing is get the favicon file from the website, or use this script to get the favicon image hash.
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
				With this we get the hash to use in the shodan query to get the information that we want.
				
				![Image Not Found!](/assets/img/favicon1.png)

				`http.favicon.hash:2042744103`

				![Image Not Found!](/assets/img/favicon2.png)

3. ### Users
As enumeration part do a possible user list or user and credentials in a public leak is a good point to use in other attacks as password srapying or try to access by VPN. You can consulting resources as [dehashed](https://www.dehashed.com/) for this. 

	1. ### Users discovery tools
		- ### Leaks 
			In addition to get the data leaks in a known databases, like dehashed, you can search in puclib web like pastebin or github, google...
			For get this leaks you can use google dorks, similar to these one:
			```dorks
			site:pastebin.com "CompanyName.com"
			site:github.com "CompanyName.com"
			site:googleapis.com "CompanyName.com"
			site:drive.google.com "CompanyName.com"
			.
			.
			.
```
			![Image Not Foud](/assets/img/paste1.png)
			
			Also you can user pay tools to do the google dork for us, searching in pastebim or github. Likewise exist tools to search in the Deep Web data leaks of the company that you ask, *"pwnd"* is an example of tool to do this. actually this tool doesn't work

		- ### CrossLinked
			You can use this tool to do a user list. Ask to linkedin for the company name and return a name and surname list with the formate that you specify. If you know the format that the company use to build the email address, you can get a list to use in a phshing attack, for example.
			**Install**
		
			`pip3 install crosslinked`

			An example to generate a list for *NomEmpresa*, we create a list wit the name Capital letter dot surname, this is the result:
			
			`crosslinked -f {f}.{last}@NomEmpresa.com NomEmpresa`

			![Image Not Found](/assets/img/crosslinked.png)

4. ### Services
	1. ### Service discovery tools
		- ### Shodan
			It os a search engine where you can use to get the open ports for a services exposed public on internet. You can do specifics queries to find assets. With this tool you avoid user your own Ips to scan the assets.

			![Image Not Found](/assets/img/shodan1.png)

			You can see the port with the result of the *curl -I*, you can do specific queries for the versions.

			![Image Not Found](/assets/img/shodan2.png)

		- ### Fofa
			https://en.fofa.info/

			Fofa is similar to Shodan, you can ask the same than shodan to compare the information.

			![Image Not Found](/assets/img/fofa.png)

			The information returns is similar than the Shodan's retunr, give us the open ports, with the reult of a *curl-I*

		- ### Nmap
			Nmap is the most common tool used to know which services offered for a server.
			Has option like:
			```
				-sV return the banner service information.
				-p- do the scan over all ports
				--top-ports 2000 do scan over the 2000 more common ports, you can select the number of ports to scan
			```

			You can add the flags -sT -sS -sU or -sn to select the kind of scan, and the protocols.
			```
				-sT Send TPC packet and complete the handshake communication.
				-sS Do a stealth scan, to try not be detected. It doen't complete the handshake communication.
				-sU Send UDP package, used to detect services used this protocol.
				-sn Some devices don't use ICMP protocol and have block reply this protocol, to avoid be blocked you can use -sn flag.
			```

			![Image Not Found](/assets/img/nmap.png)

			In this example, it seens that has the 80 and 8082 ports open, filtered means that could have the ports open but they could have a firewall in front of them.
			the flag -sV we received the banner of the service, in this case ssh, the usually port for this service is 22.

5. ### Data leak
	1. ### Data leak discovery tools
		Sometimes, due to errors or oversights, sensible data could be published in the repository of company apps or subidiaries apps. It possible get information abou email adress, users accounts and passwords.
		- ### .git
			A common way is find a web or public app with the folde .git in the public folder, in dis .git folde you can get information. Sometimes when you access to this folder yo received a 403 response, this not must means that you don't have access to this folder. With tool like git-dumper you can download the folder .git 

			`pip install git-dumper`

			To use it we have to put the url and the folder, with this download the foldet content, then we have to check the information inside to use them.

			`git-dumper https://revistas.urldetest.es/ .git`

			![Image Not Found](/assets/img/git-dump.png)
			
			For exmaple with `grep -rE "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b"` we do a recursive search email addres in the folders.

			![Image Not Found](/assets/img/git-dump2.png)

		- ### Metadata	
			The metadata is information added in the png, jpeg, pdf, doc... files. In these data it could see the locations where the photo was taken or software used, or users. You can use tools like FOCA or metagoofil to check the files associatted to the domains, and extrac these metadata.

			*Metagoofil*() is a tool in python2 to download the files with the extension that you put and get all the metadata in these files. To run it:

			`python metagoofil.py  -d DOMINIO -t xdoc,doc,pdf -l 250 -n 50 -o endesafiles -f results.html`

			![Image Not Found](/assets/img/metagoofil.png)

			The proyect is deprectaed and actually do queries to google. Google can bloock to do a lot of queries and the metagoofil stop download the files.
		
		- ### Noseyparker
                        [NoseyParker](https://github.com/praetorian-inc/noseyparker) this tool is usefull, do a scan to the repository that we want and try to get the sensible data inside.
                        Can do scan to find all user repositories, for that we use the next with this flag:

                        `noseyparker github repos list --user alhkytran`


                        ![Image Not Foud](/assets/img/noseyparker1.png)

                        If we know the reposotory we can add to scan it and stored the data in a data base. 

                        `noseyparker scan --datastore np.test --git-url https://github.com/praetorian-inc/noseyparker`

                        ![Image Not Foud](/assets/img/noseyparker2.png)

                        It could ask for the content to the data base

                        ![Image Not Foud](/assets/img/noseyparker3.png)

                        To ask for the information to the database we have to run:

                        `noseyparker report --datastore np.test`

                        ![Image Not Foud](/assets/img/noseyparker4.png)

                        ![Image Not Foud](/assets/img/noseyparker5.png)


6. ### Extra
	When this post was created in the DefCon 2023 where explain the recon tool [DorXNG](https://github.com/ResearchandDestroy/DorXNG) this tool get information about differents resources, even in the deep web with tor network. [here](https://github.com/ResearchandDestroy/DorXNG/blob/main/query.lst) they have an example to differents queries to use. 
To begin, as example, we can use this query.

	`DorXNG.py -q "DB_USERNAME filetype:env" -d test.db`

	![Image Not Foud](/assets/img/dorxng1.png)

	Search in the files with name *env* with the content *DB_USERNAME* and storage in database *test.db*, when you ask to the results can see the next: 

	![Image Not Foud](/assets/img/dorxng2.png)

	Other query to do is `DorXNG.py -q 'intitle:"index of" "pass.txt"' -d test.db` for search files with the name pass.txt and you cant find interesting things. 

	![Image Not Foud](/assets/img/dorxng3.png)

	You can see things like this one. 

	![Image Not Foud](/assets/img/dorxng4.png)	![Image Not Foud](/assets/img/dorxng5.png)

	It could play with the queries like google dorks. 

	For ask in the database we can use the next command
	
	`DorXNG.py -D "DB_USERNAME" -d test.db`

	![Image Not Foud](/assets/img/dorxng6.png)

---
<br>
## References
- https://blog.intigriti.com/2021/06/08/hacker-tools-amass-hunting-for-subdomains/
- https://medium.com/@sherlock297/install-massdns-on-kali-linux-a743ef044bc8
- https://github.com/praetorian-inc/noseyparker
- https://github.com/ResearchandDestroy/DorXNG


## Thanks
- [Mario Bartolome](https://github.com/MarioBartolome) - *For tireless help and inspiration* 
