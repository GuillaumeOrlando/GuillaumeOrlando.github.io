---
layout: post
title: Writeup - DevOops
categories: Writeup
---

### Introduction
DevOops est une machine vulnérable proposée par Hack The Box, rapportant 30 points. La machine est accessible à l’adresse IP 10.10.10.91.

### Scan

Je commence par scanner l’ensemble des ports TCP et UDP de la machine avec masscan:
> masscan -p1-65535,U:1,65535 10.10.10.91 --rate=500 -e tun0

Deux ports sont ouverts : l’un qui semble être un service http sur le port 5000, et un service SSH sur le port 22.
> 5000/tcp    open    http     Gunicorn 19.7.1
> 22/tcp        open    ssh      OpenSSH 7.2p2 ubuntu

Le service SSH est à jour, et ne comporte aucunes vulnérabilités exploitables.
Le port 5000 pointe quand à lui sur la page web suivante :
![devoops-A](/img/devoops/A.PNG){:class="img-responsive"}

N’ayant pas d’autres options d'interactions avec la page web, je lance un scan gobuster :
> gobuster -u http://10.10.10.91 -w /directory-list-medium-2-3.txt -x php,py -s 200,403,204

Deux pages me sont retournées après quelques temps :
> http://10.10.10.91/feed

> http://10.10.10.91/upload

### XML upload

La première ne correspond qu’à l’image statique présent sur l’index, ce qui est sans intérêts. La deuxième page, en revanche, me mène à un service d’upload de fichier XML :
![devoops-B](/img/devoops/B.PNG){:class="img-responsive"}
La page insiste sur le fait que les éléments “author”, “subject” et “content” doivent êtres présents dans le fichier XML à uploader. J’essai donc de crafter un fichier XML valide aux yeux du serveur :
![devoops-C](/img/devoops/C.PNG){:class="img-responsive"}

Malheureusement, la syntaxe ne semble pas bonne, puisque le serveur me retourne une erreur 500 :
![devoops-D](/img/devoops/D.PNG){:class="img-responsive"}
En tâtonnant avec mon fichier XML, j’arrive à la conclusion que ce format est le bon :
![devoops-E](/img/devoops/E.PNG){:class="img-responsive"}

### XML External Entity

Puisque le serveur web accepte et traite mon fichier sans erreurs, l’injection de fichier XML est possible. Une faille de type XXE (XML External Entity) serait alors potentiellement exploitable. Une faille XXE permet d’uploader un fichier arbitraire sur un serveur web acceptant un fichier XML malicieux, ou de lire le filesystem de la machine cible.
Je décide de crafter un fichier XML malicieux pour essayer de lire le contenu du fichier /etc/passwd du serveur web :
![devoops-F](/img/devoops/F.PNG){:class="img-responsive"}

En envoyant ce fichier au serveur web, celui-ci me répond bien avec la liste des utilisateurs du système :
![devoops-G](/img/devoops/G.PNG){:class="img-responsive"}

Les trois utilisateurs surlignés sortent du lot. N’ayant que la possibilitée de lire les fichiers de la machine, il me faut trouver un fichier sensible ayant la capacité de faire spawn un shell interactif. Je commence donc naturellement par les emplacements de clés ssh des trois utilisateurs.
C’est pour l’utilisateur ‘roosa’ que je découvre en aveugle les clés publiques et privée à la disposition de tous. Les clés sont dans le répertoire par défaut, sous /home/roosa/.ssh/.
Celles-ci sont nommés par défaut, à savoir ‘id-rsa’ et ‘id_rsa.pub’.

Je m’empresse de récupérer la clé privée en craftant un nouveau fichier XML malicieux :
![devoops-H](/img/devoops/H.PNG){:class="img-responsive"}

Il ne me reste plus qu'à copier-coller le contenu de la clé en local, de lui appliquer les bonnes permissions (chmod 600), et de me connecter avec :
![devoops-I](/img/devoops/I.PNG){:class="img-responsive"}

### Escalation de privilège
Mon premier réflexe lors de l’obtention d’un compte low privilège consiste à observer l’historique des commandes disponible pour situer le contexte de l’utilisation de ce compte. Dans le cas présent, l’historique est plein de surprises. De nombreux ‘git commit’ sont effectuées, et l’un d’eux semble des plus intéressants :
 ![devoops-J](/img/devoops/J.PNG){:class="img-responsive"}

### Watch your git commit!

Il semblerait qu’une clé ssh indésirable ai été poussé dans le projet via l’outil de versionning git. Le fichier ‘authcredentials.key’ actuel contient la clé ssh de notre utilisateur.
La clé root aurait pu être poussée par inadvertance, avant d’être retirée du projet !
Je vérifie la théorie en commençant par observer l’historique des log gits :
> cd work/ressources/ && git log -10

![devoops-K](/img/devoops/K.PNG){:class="img-responsive"}

L’indicateur de version mis en valeur correspond donc à la version antérieure qui m'intéresse. Pour restaurer le fichier de cette version, j’utilise la commande suivante:
> git checkout <commit number> -- ressources/integration/authcredentials.key

Il ne me reste donc qu’à récupérer cette version anterieur du fichier, de l’exporter sur mon système local, et de l’utiliser pour obtenir un compte root sur la machine :
![devoops-L](/img/devoops/L.PNG){:class="img-responsive"}
