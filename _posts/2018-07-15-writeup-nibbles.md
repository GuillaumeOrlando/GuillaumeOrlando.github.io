---
layout: post
title: Writeup - Nibbles
categories: Writeup
---
Nibbles est une machine provenant de la plateforme [Hack The Box](https://www.hackthebox.eu). Celle-ci est indiquée comme étant la plus simple du lab avec une note de difficultée s’élevant à 3.6/10, et rapportant 20 points.
L'adresse IP de la machine est 10.10.10.75.

## Enumération initiale
### Nmap
Je commence par scanner les ports de la machine cible avec la commande suivante :
{% highlight bash linenos %}
nmap -sC -sV -T4 -A 10.10.10.75
{% endhighlight %}
Le scan me retourne 2 services en façade : un service SSH et un service web.
> 22/tcp     open    ssh         OpenSSH 7.2p2                                                                
> 80/tcp     open    http         Apache httpd 2.4.18 (Ubuntu)

### Service web
La visite de l’index web n’apporte rien de concret à première vue, mais en inspectant la page, la présence d’un répertoire /nibbleblog/ nous est révélé :
{% highlight html linenos %}
<html>
  <head></head>
  <body>
    <b>Hello world!</b>
    <!--/nibbleblog/ directory. Nothing interesting here!...-->
  </body>
</html>
{% endhighlight %}


### NibbleBlog
NibbleBlog est un CMS aux allures de Wordpress, ce qui justifie donc le titre de cette machine. Etant donnée la difficulté de la machine, et mon expérience avec les différents CMS, j’ai déjà une idée bien précise d’un schéma d’exploitation classique, qui consistera très certainement à uploader un shell au travers d’une fonctionnalitée du CMS (un thème, une page php, etc …).
Voyons si mes prédictions sont bonnes.

Le contenu du CMS n’est pas très important, et aucunes informations ou noms d’utilisateurs ne semble se démarquer. Je décide donc de me lancer à la recherches d’informations sur la structure de ce CMS, et notamment les URL par défauts des pages d’authentifications. Je lance également un scanner nikto sur l’URL http://10.10.10.75/nibbleblog/ en parallèle de mes recherches.
> nikto -h http://10.10.10.92/nibbleblog

Je réussi ainsi à obtenir la page d’authentification par défaut : admin.php !
Le scanner nikto me remonte quand à lui la présence d’un dossier /admin/.
La page d’authentification est des plus classiques :

![nibbles-admin-login](/img/nibbles/login.png){:class="img-responsive"}

Le répertoire admin contient quand à lui des tonnes de dossiers et de fichiers relatifs au CMS. En cherchant à interagir avec ces fichiers, je me rend compte que des droits administrateur sont requis.
Toujours en cherchant des informations sur le CMS, j’apprend l'existence du dossier /content/.
Après de très longues minutes d'épluchures minutieuse des nouveaux fichiers à disposition, je tombe sur un ligne qui me permet d’obtenir un nom d’utilisateur :
> <user username=”admin”>

## Exploitation
De ce fait, il m’est possible de retourner sur la page d’authentification et de commencer à essayer des mots de passes par défauts. Après seulement quelques essais, il s’avère que le mot de passe correspondant était simplement “nibbles”.

Maintenant qu’un accès aux pages d’administrations du CMS sont ouvertes, cherchons un endroit ou il serait possible d’uploader un shell. Ce sera finalement dans le plugin “My images”, d'où je vais uploader un reverse shell en PHP, d'après les instructions de cet [article](https://curesec.com/blog/article/blog/NibbleBlog-403-Code-Execution-47.html).
Il ne me reste plus qu'a déployer un listener netcat en accord avec mon reverse shell pour obtenir un terminal utilisateur sur la machine cible !
> nc -lvvp 1234

Un shell tournant avec les privilèges du compte utilisateur 'nibbler' s'ouvre ainsi à moi.

## Escalation de privilège
Le répertoire courant de l’utilisateur nibbler contient une archive “personal.zip”. En dézippant celle-ci, un script se présente : ‘monitor.sh’. Ce script semble permettre de monitorer l’état de santé de la machine.
Avant de continuer, je décide de faire spawn un shell intercatif :
> which python3                           
> python3 -c 'import pty; pty.spawn("/bin/bash")'

Après avoir fait spawn un shell plus présentable via python 3, je décide d’énumérer le contenu de la machine. C’est alors que l’environnement de travail se reset de manière automatique, me laissant le message suivant :
{% highlight bash linenos %}
Matching Defaults entries for nibbler on Nibbles:
  env_reset, mail_badpass,
  secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bien
User nibbler may run the following commands an Nibbles:
  (root) NOPASSWD: /home/nibbler/personnal/stuff/monitor.sh
{% endhighlight %}

La faille se présente donc à moi avec ce message. La dernière ligne indique que l’utilisateur courant nibbler à le droit d'exécuter le script monitor.sh en tant que root, et ce, sans fournir de mot de passe. Il suffit ici de remplacer le contenu du script par l’instruction ‘su’, de façon à faire apparaître un terminal superutilisateur.
> echo 'su' > /home/nibbler/personnal/stuff/monitor.sh

Le contenu du script n’a pas d’importance, tant que le fichier ‘monitor.sh’ existe à cet emplacement précis, alors son exécution se fera via le compte root.

En lançant le script via la command indiquée, j'obtiens un terminal administrateur, qui me permet de récupérer le flag de validation, et de terminer cette machine.
> /home/nibbler/personnal/stuff/monitor.sh

Une machine effectivement très simple, mais qui reste des plus plaisante pour autant.
