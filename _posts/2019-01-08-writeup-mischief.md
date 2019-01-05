---
layout: post
title: Writeup - Mischief
categories: Writeup
---
## Introduction
Mischief est une machine virtuelle vulnérable de la plateforme hack The box. Celle-ci est une des machines les plus difficiles des 10 machines que j’ai complété. La machine sera joignable via l’IP 10.10.10.92, et est proposée par l’utilisateur ‘Trickster0’.
![mischief](/img/mischief/A.PNG){:class="img-responsive"}
## Obtention d’un shell low privilège
Je commence par effectuer un scan de routine avec nmap :
> nmap -sC -sV -A 10.10.10.92

Le scan ne trouve qu’un service SSH sur le port 22. Sachant que le serveur SSH est à jour et que je ne peux rien en tirer, je lance donc un scan complet sur tous les ports TCP de la machine :
> nmap -p- -sC -sV -A 10.10.10.92

Et sur tous les ports UDP :
> nmap -p- -sU -sV -sC 10.10.10.92

Deux nouveaux services apparaissent ainsi :
- Un serveur HTTP python sur le port 3366.
- Un service SNMP port 161 UDP.
En visitant le service web, je tombe sur un prompt d’authentification.
![mischief](/img/mischief/B.PNG){:class="img-responsive"}
Les identifiants par défauts les plus courants ne fonctionnant pas, je décide de chercher des informations complémentaires sur ce serveur HTTP Python.
Le header du service indique “Python BaseHTTPServer”, et le bandeau de demande de mot de passe affiche “Test”. En cherchant sur le net, je tombe sur la page Github du serveur web dont il est question :
https://github.com/AGProjects/python-eventlib/blob/master/eventlib/green/BaseHTTPServer.py
Mauvaise nouvelle, celui-ci n'inclut aucunes vulnérabilités ni mots de passes par défauts.

Il ne me reste donc plus que le service SNMP à vérifier. Le scan nmap nous retourne déjà des informations utiles :
![mischief](/img/mischief/C.PNG){:class="img-responsive"}
En effet, la communautée SNMP semble être celle par défaut, à savoir ‘public’. Il devrait alors être possible d’interroger l’hôte avec le protocole SNMP, et potentiellement de retrouver des informations ou des vulnérabilités. L’outil snmpwalk sera ici utilisé. La syntaxe à utiliser est la suivante :
> snmpwalk -c <communautée> <IP> <version> <élément>

![mischief](/img/mischief/D.PNG){:class="img-responsive"}
Premier soucis, les MIBs ne sont pas chargés. Une MIB est une sorte de bibliothèque d’information SNMP propre à chaques constructeurs de produits compatibles SNMP. Les MIB comportent plusieurs champs qui identifient des informations fournis via le protocole SNMP. Ici, les MIBs ne sont pas chargés, puisque chaques champ correspond à une suite de numéro. Pour y remédier, il suffit de commenter le contenu du fichier /etc/snmp/snmpd.conf.
![mischief](/img/mischief/E.PNG){:class="img-responsive"}
Voilà qui est déjà plus lisible. Afin d'énumérer correctement le service, je procède méthodiquement. Je commence par obtenir des informations sur le système d’exploitation:
![mischief](/img/mischief/F.PNG){:class="img-responsive"}
J’obtiens ensuite la liste des paquets installés sur la machines, diverses informations sans importances, puis la liste des processus.
Je décide de creuser la piste des processus plus en détails; puisque d’après la page Github de serveur HTTP python, celui-ci doit être lancé avec comme paramètre un utilisateur et un mot de passe. Si je réussi à obtenir la liste complète des processus, j’obtiendrai certainement un utilisateur est un mot de passe.
Après plusieurs recherches, c’est finalement l‘attribut “hrSWRunParameters” qui me permet d’avoir accès à ces informations. En les décortiquant, je tombe bien sur le processus attendu, comportant un couple d’identifiant en clair :
![mischief](/img/mischief/G.PNG){:class="img-responsive"}
Ces identifiants me permettent ainsi d’accéder à la page web tournant sur le port 3366, et d’y trouver une autre mot de passe :
![mischief](/img/mischief/H.PNG){:class="img-responsive"}
J’essai immédiatement de me connecter au compte utilisateur avec ce mot de passe via ssh, mais cela semble trop beau pour être vrais. Comme prévu, ceci ne fonctionne pas. Ce mot de passe doit servir quelque part, mais il me manque l’information.
Je retourne donc sur le service SNMP afin d’énumérer de nouvelles choses, et de trouver des indices. C’est lorsque passe devant moi un champ SNMP relatif à une adresse IPv6 que je décide de vé”rifier si une instance web IPv6 ne tourne pas sur la machine. En tombant sur l’outil permettant de récupérer l’adresse IPv6 à partir d’une adresse IPv4, je me rend compte que je suis sur la bonne voie : l’outil en question à été écrit par le créateur de cette machine.
![mischief](/img/mischief/I.PNG){:class="img-responsive"}
L’outil me retourne bien l’adresse IPv6 de la machine cible. Attention à dé-commenter le contenu du fichier /etc/snmp/snmp.conf avant de l'exécuter :
![mischief](/img/mischief/J.PNG){:class="img-responsive"}
En visitant la page web IPv6 relative à la machine, je tombe bien sur une nouvelle page. Petite astuce, pour visiter une page IPv6, il suffit de taper l’adresse IPv6 dans l’URL entre crochets “[“ “]”
![mischief](/img/mischief/K.PNG){:class="img-responsive"}
Avant d’essayer de me connecter à ce panel d'administration à distance, je lance un scan gobuster sur la racine du serveur web IPv6 :
![mischief](/img/mischief/L.PNG){:class="img-responsive"}
En jouant avec la page d’authentification, je découvre que les identifiants trouvé auparavant ne fonctionnent pas. Je lance également un Intruder BurpSuite afin de bruteforcer le nom de l’utilisateur pour le mot de passe trouvé.

Tombant à cours d’idées, et observant mon scan BurpSuite, je découvre une code HTTP différent pour l’utilisateur “Administrator”.

En essayant de me connecter à ce compte avec le mot de passe précédent, je tombe sur une console d'exécution de commandes à distances :
![mischief](/img/mischief/M.PNG){:class="img-responsive"}
L’objectif est ici très simple : obtenir les identifiants de l’utilisateur loki, comme indiqué sur la page web. Il est très certain que les commandes seront filtrés sur cette page web, et un simple ls le confirme :
![mischief](/img/mischief/N.PNG){:class="img-responsive"}
Le challenge va donc consister à déterminer les commandes autorisées, afin de réussir à interagir avec le système correctement.
Je commence donc par tester des commandes unix basiques. Les résultats sont notés dans ce tableau :
![mischief](/img/mischief/O.PNG){:class="img-responsive"}
Comme le montre la dernière lignes, lorsque le code retours de la commande est différent de 0, un message de succès est quand même retourné. Deuxièmement, toutes les commandes sont exécutée en aveugles, ce qui complique particulièrement la tâche. Je me met donc à tester différents caractères d’échappement afin d’observer le comportement de cet outil d'exécution de commandes :
![mischief](/img/mischief/P.PNG){:class="img-responsive"}
Il semblerait donc que les caractères spéciaux ne soient pas filtrés. Je réussi ainsi à afficher le résultat d’une de mes commandes de la façon suivante :
![mischief](/img/mischief/Q.PNG){:class="img-responsive"}
Il va donc être possible d’énumérer le système et de partir à la recherche de ce fameu fichier de mot de passe. Mais avant, je décide d’observer le contenu du script afin d’observer les commandes autorisés.
![mischief](/img/mischief/R.PNG){:class="img-responsive"}
Malheureusement, en essayant d’afficher le contenu du fichier ‘index.php’, celui-ci est interprété dans le navigateur. je découvre également l’interdiction d’utiliser l'extension “php” :
![mischief](/img/mischief/S.PNG){:class="img-responsive"}
Pour y remédier, il suffit d’extraire le contenu au format base64 :
![mischief](/img/mischief/T.PNG){:class="img-responsive"}
Il ne reste plus qu’à décoder la chaîne pour retrouver le contenu du fichier php :
Attention, celle-ci s'est vu ajouter des espaces sur la page web, il sera donc nécessaire de les retirer puis de décoder le contenu :
> python -c ‘chaine = “PD9wa[...]cg==”; print(str(chaine).replace(“ “,””)) ’ | base64 -d

![mischief](/img/mischief/U.PNG){:class="img-responsive"}
La liste des commandes interdites en ma possession, je pars maintenant à la recherche du fichier évoqué plus haut sous /home/loki/credentials. A noter que les commandes sont ici exécutés via l’utilisateur www-data.
![mischief](/img/mischief/V.PNG){:class="img-responsive"}
En se connectant via ssh avec ce mot de passe, un shell utilisateur s’ouvre :
![mischief](/img/mischief/W.PNG){:class="img-responsive"}
## Escalation de privilège
Avec du recul, l’escalation de privilège n’est pas des plus difficile, mais celle-ci m’a pris plusieurs jours avant de comprendre mes erreurs. Comme pour toute machines, j’ai effectuer plusieurs actions afin de déterminer un potentiel vecteur d’élévation de privilège :
  Le contenu du home de l’utilisateur loki :
    - L’Historique bash ne mentionne rien.
    - Un historique MySQL inutile indique la présence d’un base MySQL.
    - Bashrc classique
  Service cron:
    - Aucuns jobs prévus pour l’utilisateur loki.
    - Les autres jobs sont inutilisables.
  La base MySQL :
    - Rien d'intéressant, pas de mots de passes en clair
  L’OS est récent (ubuntu 18), donc pas d’exploits disponibles. Pareil pour le kernel.
  Aucuns scripts customisés permettant d’abuser les privilèges actuels
  Impossible de lancer la commande sudo ou su.
  La liste des processus est banale.
  Les programmes SUID ne sont pas intéressants.

A ce stade, je commence à tourner en rond. C’est après plusieurs jours à creuser dans tous les sense que je me rend compte que l’historique contient bel et bien un indice :
![mischief](/img/mischief/X.PNG){:class="img-responsive"}
En passant trop rapidement sur ce fichier, j’ai cru que la ligne était celle retrouvée plus tôt avec le service SNMP. J’ai donc en ma possession un nouveau mot de passe. Malheureusement, il m’est impossible de changer de compte via loki. En inspectant les permissions du binaire /bin/su, je me rend compte que celui-ci possède des permissions étendues :
![mischief](/img/mischief/Y.PNG){:class="img-responsive"}
Pour observer les permissions étendues, j’utilise getfacl :
![mischief](/img/mischief/Z.PNG){:class="img-responsive"}
Ainsi, le binaire /bin/su n’est pas exécutable par l’utilisateur loki, mais l’est pour tous les autres. Il ne me reste donc qu’à trouver un autre utilisateur disponible sur lequel pivoter vers le compte root. Puisque loki est le seul utilisateur du système, il ne me reste plus qu’à essayer via www-data sur le panel web d'exécution de commandes.
Malheureusement, su doit être lancé depuis un TTY, et le panel d'exécution n’est est pas un.

La dernière solution reste donc d'exécuter un reverse shell depuis le panel web, puis d’y faire spawn un véritable terminal.

Après plusieurs essais, de découvre qu'aucuns des reverses shell ne réussissent à se connecter sur ma machine. En essayant d’utiliser un reverse shell IPv6, le problème est résolu :
![mischief](/img/mischief/ZA.PNG){:class="img-responsive"}
Attention à bien utiliser ncat et non nc pour récupérer la connexion IPv6. le premier ne la supporte pas.

Je fais ensuite spawn un terminal :
![mischief](/img/mischief/ZB.PNG){:class="img-responsive"}
Je passe ensuite root avec su et le mot de passe ‘lokipasswordmischieftrickery’. Dernière frayeur pour la fin, le flag root explique qu’il faut un vrais terminal pour être consulté.

Cela signifie que le flag root est situé ailleurs. Je lance la commande suivante depuis le compte loki afin de le trouver :
> find /-name root.txt -print 2>/dev/null

Et le fichier se trouve sous /usr/lib/gcc/x86_64/x86_64-linux-gnu/7.3.0/root.txt. De retours sur le terminal root, il est possible d’en lire le contenu et de valider la machine !

Une machine pleine de pièges plus intéressants les uns que les autres, avec beaucoup de sujets à découvrir.
