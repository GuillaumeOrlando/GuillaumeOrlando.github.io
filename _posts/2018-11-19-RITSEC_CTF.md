---
layout: post
title: RITSEC CTF 2018
categories: Writeup
---
## Space Force
![devoops-A](/img/ritsecctf/L-1.PNG){:class="img-responsive"}
Le site en question se compose d'un simple formulaire de recherche permettant d'accéder aux scores d'un jeux.
![devoops-A](/img/ritsecctf/B-1.PNG){:class="img-responsive"}
Puisqu'il s'agit du premier challenge de la catégorie, je vérifie le comportement de l'application avec les différents caractère d'échappement SQL, afin de potentiellemnt couvrir une injection SQL.
![devoops-A](/img/ritsecctf/C-1.PNG){:class="img-responsive"}
Une erreur applicative est levée, ce qui confirme la théorie.
Après plusieurs tentatives, je trouve enfin une façon de dumper le contenu de la base de données avec une simple
> Test' or '1' = '1

A noter qu'il est aussi possible d'automatiser l'opération avec SQLmap:
> sqlmap -u "http://fun.ritsec.club:8005/index.php" --data="name="

![devoops-A](/img/ritsecctf/E-1.PNG){:class="img-responsive"}
Il ne restera plus qu'à se déplacer au sein des différentes tables de la bases jusqu'a trouver le flag de validation:
![devoops-A](/img/ritsecctf/K-1.PNG){:class="img-responsive"}

## The Tangled Web
![devoops-A](/img/ritsecctf/G-2.PNG){:class="img-responsive"}
Ce second challenge pointe vers un site comportant un nombre impressionnant de liens par pages, ce qui rend très compliqué la compréhension de l'architecture globale du site.
![devoops-A](/img/ritsecctf/H-2.PNG){:class="img-responsive"}
Je lance donc un spider BurpSuite sur la racine de l'application web, afin d'avoir un premier aperçu.
Je remarque immédiatement une page qui pourrait contenir le flag de validation:
![devoops-A](/img/ritsecctf/D-2.PNG){:class="img-responsive"}
Malheuresement, c'est un piège:
![devoops-A](/img/ritsecctf/C-2.PNG){:class="img-responsive"}
Après quelques recherches, et en filtrant sur la taille des pages et sur des termes potentiellement présent sur la page contenant le flag, je trouve enfin ce dernier dans l'arborescence:
![devoops-A](/img/ritsecctf/E-2.PNG){:class="img-responsive"}

## What a cute dog!
![devoops-A](/img/ritsecctf/A-3.PNG){:class="img-responsive"}
Ce challenge comprend un simple page web qui affiche des informations générales sur le serveur web.
![devoops-A](/img/ritsecctf/B-3.PNG){:class="img-responsive"}
En observant la source HTML, une référence vers l’URL “fun.ritsec.club:8008/cgi-bin/stats”  permet de comprendre d’ou viennes les informations affiché sur le site:
![devoops-A](/img/ritsecctf/C-3.PNG){:class="img-responsive"}
Ce genre d’inclusion de pages fait directement penser à un exploit de la vulnérabilité shellshock. Un simple test se montre concluant, et permet effectivement d’injecter des commandes
![devoops-A](/img/ritsecctf/D-3.PNG){:class="img-responsive"}
Il ne reste plus qu’à chercher le flag de validation avec la commande find pour valider ce challenge:
![devoops-A](/img/ritsecctf/E-3.PNG){:class="img-responsive"}

## Burn The Candle On Both Ends
![devoops-A](/img/ritsecctf/A-4.PNG){:class="img-responsive"}
Ce challenge de forensic met à disposition l’image suivante
![devoops-A](/img/ritsecctf/B-4.PNG){:class="img-responsive"}
En analysant l’image, Binwalk confirme la présence d’une archive cachée au sein de l’image JPG:
![devoops-A](/img/ritsecctf/C-4.PNG){:class="img-responsive"}
En analysant le contenu de l’archive protégée par mot de passe, un fichier flag.txt semble être présent
![devoops-A](/img/ritsecctf/D-4.PNG){:class="img-responsive"}
Avec l’outil “zip2john”, il est possible d’extraire le hash du fichier zip afin de le cracker avec JohnTheRipper dans la foulée
![devoops-A](/img/ritsecctf/E-4.PNG){:class="img-responsive"}
Après quelques temps face à la wordlist rockyou.txt, le mot de passe est enfin révélé: il s’agit de “stegosaurus”.
Il ne reste plus qu'à extraire l’archive avec ce mot de passe pour obtenir le flag de validation
![devoops-A](/img/ritsecctf/F-4.PNG){:class="img-responsive"}
