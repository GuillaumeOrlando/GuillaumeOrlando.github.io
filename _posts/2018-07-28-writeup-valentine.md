---
layout: post
title: Writeup - Valentine
categories: Writeup
---
Valentine est une machine boot2root disponible sur la plateforme HackTheBox. Celle-ci à une difficulté générale de 4.1 / 10.
Cette machine ne devrait pas être des plus difficiles au vu des notes laissées par les précédents joueurs. L'adresse IP de la machine est 10.10.10.79.


## Enumération initiale
### Nmap
Par habitude, je commence par lancer un scan nmap de l’adresse IP fournie :
{% highlight bash linenos %}
nmap -sC -sV -T4 -A 10.10.10.79
{% endhighlight %}
Un total de 4 services sont découverts :
>22/tcp     open    ssh         OpenSSH 5.9p1
>80/tcp     open    http         Apache httpd 2.2.22 (Ubuntu)
>443/tcp    open    ssl/http    Apache httpd 2.2.22 (Ubuntu)
>9000/tcp   open    cslistener?

### Service web
En accédant à l’index du serveur web, l’image suivante se présente, sans interactions possibles :
![valentine-index-img](/img/valentine/index.png){:class="img-responsive"}
Tout en lançant un scanner automatique nikto sur le service web, je décide d’étudier l’image à la recherche d’informations pertinentes. En essayant de déchiffrer une quelconque information cachée dans l’image je passe à côté de la signification de celle-ci, mais je reviendrai plus tard sur ce point. L’étude stéganographique de l’image ne m’apporte rien de concret, je me tourne donc vers le résultat de mon scan nikto.

Premièrement, celui-ci me confirme la version du service Apache, soit 2.2.22.
Ensuite, la présence du mod_negotiation activé en multiviews devrait permettre de bruteforcer le contenu de serveur web. Des fichiers PHP par défaut sont accessibles, nous informants des versions utilisées. Et enfin, une information des plus utiles : un répertoire /dev/ semble exister à la racine du serveur web.

Je décide donc de naviguer jusqu’à ce fameux répertoire pour trouver des fichiers forts utiles :
![valentine-dir](/img/valentine/dir.png){:class="img-responsive"}

Le premier contient une chaîne de caractères hexadécimaux :
![valentine-hex](/img/valentine/hex.png){:class="img-responsive"}

Je m’empresse de les convertir au format ASCII pour me rendre compte que je viens d’obtenir une clé privé :
![valentine-key](/img/valentine/key.png){:class="img-responsive"}

Malheureusement, comme le montre les deux premières lignes de la clé, celle-ci est protégée par une passphrase … A noter que le nom du fichier hexadécimal d'où provient cette clé est “hype_key”. Il pourrait s’agir d’un indice laissant penser que cette clé appartient à l’utilisateur “hype”, bien qu’il est impossible de vérifier cette théorie pour l’instant.

Le deuxième fichier dans le répertoire /dev/ du serveur web est une note laissée par l’administrateur de la machine :
![valentine-note](/img/valentine/note.png){:class="img-responsive"}

Cette note indique la présence d’un service de chiffrement et de déchiffrement sur la plateforme, et précise que ce service n’est pas nécessairement terminé. Après quelques essaies, je tombe sur les pages dont parle la note à l’adresse http://10.10.10.79/encode et http://10.10.10.79/decode :
![valentine-cipher](/img/valentine/cipher.png){:class="img-responsive"}

En jouant avec l’encodeur et le décodeur, je me rend compte qu’il s’agit en fait d’un simple service de chiffrement et de déchiffrement en base64. Après avoir passé de longues heures à essayer d’exploiter le formulaire des services pour exécuter du code PHP côté serveur, je décide de prendre une pause pour observer la globalité des informations obtenues jusque là.

### Exploitation

Je pars à la recherche d’exploit pour les versions des services OpenSSH et Apache sans succès. Je m’attarde sur le port 9000 avec lequel je n’arrive pas à interagir, avant de réaliser que l’image trouvée plus tôt représente la solution à mon problème.
En traitant l’image comme un hypothétique moyen de cacher de l’information, je n’ai pas pris le temps de m’attarder sur sa signification. L’illustration est la traduction littéral du nom d’une vulnérabilité bien connu : HeartBleed !

HeartBleed est une vulnérabilité dans l’implémentation des bibliothèques de cryptographie d’OpenSSL, permettant d’effectuer des fuites mémoires sur le serveur (CVE-2014-0160). Je m’empresse alors de lancer Metasploit et de chercher un module compatible.
Je décide d’utiliser l’exploit suivant :
![valentine-msf](/img/valentine/msf1.png){:class="img-responsive"}

Celui-ci me permet de scanner l’hôte avant de l’exploiter pleinement, ce qui s’avère être une bonne chose pour confirmer la théorie de la présence d’une vulnérabilité HeartBleed :
![valentine-msf2](/img/valentine/msf2.png){:class="img-responsive"}

Je paramètre donc le module pour se comporter comme un simple scanner :
![valentine-msf3](/img/valentine/msf3.png){:class="img-responsive"}

Avant de l'exécuter et ainsi, confirmer les théories précédentes :
![valentine-msf4](/img/valentine/msf4.png){:class="img-responsive"}

Il ne reste donc plus qu’à configurer l’exploit pour réaliser un dump de la mémoire du serveur cible :
![valentine-msf5](/img/valentine/msf5.png){:class="img-responsive"}

Il faudra répéter l’exploit plusieurs dizaines de fois pour obtenir des données mémoires utiles. Au bout d'un certain temps, une chaîne semblant être encodée en base64 attire mon attention :
![valentine-msf6](/img/valentine/msf6.png){:class="img-responsive"}

Après l’avoir décodé via le service fournis par la machine cible, j’obtiens ce qui s’apparente à la passphrase manquante de la clé privé trouvée plus tôt :
![valentine-pass](/img/valentine/pass.png){:class="img-responsive"}

De plus, le contenu de la passphrase me confirme l'existence de l’utilisateur ‘hype’.
Il ne me reste plus qu’à me connecter en ssh en utilisant la clé privé pour obtenir un shell utilisateur sur la machine cible :
![valentine-shell](/img/valentine/shell.png){:class="img-responsive"}

## Escalation de privilège
Je commence par observer le contenu du répertoire /home/ de l’utilisateur ‘hype’, mais aucuns indices ne s’y trouvent. La commande ‘uname -a’ me retourne la version de l’OS et du noyau Linux, à savoir une machine Ubuntu 3.26 et un noyau version 3.2.0-23. En cherchant des exploits potentiellements intéressants, je tombe sur la faille ‘Dirty c0w’ :
![valentine-dirty1](/img/valentine/dirty1.png){:class="img-responsive"}

Cette dernière permet, via un utilisateur low-privilège, d’écrire dans des fichiers uniquement accessibles par l’utilisateur root. Je décide d’utiliser cet exploit en modifiant le code C ‘40616.c’ pour que celui-ci soit cohérent avec l’architecture de la machine cible, puis le compile, avant de l’envoyer sur la machine distante via netcat.
L'exécution de l’exploit se passe sans problèmes :
![valentine-dirty2](/img/valentine/dirty2.png){:class="img-responsive"}

Et me permet d’obtenir très simplement un compte root sur la machine :
![valentine-root](/img/valentine/root.png){:class="img-responsive"}

Il est désormais possible de chercher le flag de validation pour terminer la machine.
Je n’ai pas été un grand fan de cette machine, même si celle-ci à eu le mérite de me faire jouer avec des vulnérabilités systèmes bien connus sur lesquelles je n'avais encore jamais tenté d’exploits.2018-07-28-writeup-valentine.md
