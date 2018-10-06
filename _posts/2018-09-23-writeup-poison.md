---
title: Writeup : Poison
---
Poison est une machine boot2root disponible sur la plateforme HackTheBox. Celle-ci à une difficulté générale de 3.6 / 10, pour un total de 30 points.
L'adresse IP de la machine est 10.10.10.84.

## Enumération initiale
### Nmap
Je commence par scanner la machine avec nmap afin d’avoir une première idée des services à ma disposition :
{% highlight bash linenos %}
nmap -sC -sV -T4 -A 10.10.10.84
{% endhighlight %}

J’obtiens deux résultats, un service SSH et un service Web Apache :
> 22/tcp     open    ssh         OpenSSH 7.2p2
> 80/tcp     open    http         Apache httpd 2.4.29

Une première rapide recherche d’exploits ne me montre rien d'intéressant vis-à-vis de ces deux services. Par habitude, je teste tout de même quelques combinaisons d’identifiants et mots de passes courants sur le service SSH, mais ceci ne donne bien évidemment rien.

### Serveur web
Je décide alors de me pencher sur le service web de la machine. De toute évidence, celui-ci contient un vecteur d’attaque exploitable.
Un index est bien présent sur ce servic web, pointant vers un service d’exécution de scripts PHP :
![poison-web-index](/img/poison/index.PNG){:class="img-responsive"}

Les scripts proposés permettent de récupérer quelques informations sur le système distant, comme la version PHP utilisé et la version du système d’exploitation : FreeBSD 11.1.
Une recherche d’exploits compatibles avec ce kernel ne me donne rien.
En revanche, après avoir testé les différents scripts disponibles, je remarque deux choses :
• Premièrement, le script “listfiles.php” permet d'observer un fichier "pwdbackup" dans la racine du répertoire web courant.
![poison-web-1](/img/poison/vuln1.PNG){:class="img-responsive"}

• Deuxièmement, il semble probable qu’une faille de type ‘Directory Transversal’ soit présente au niveau de l’URL d'exécution des scripts : ![poison-web-2](/img/poison/vuln2.PNG){:class="img-responsive"}

Pour en avoir le coeur net, il suffit de manipuler l’URL afin d’obtenir la fameuse erreur PHP suivante :
![poison-web-3](/img/poison/vuln3.PNG){:class="img-responsive"}

### Directory Transversal

Avant de me ruer sur le fichier “pwdbackup.txt”, je décide d’effectuer quelques recherches d'informations supplémentaires sur la machine, en me focalisant sur quelques fichiers intéressants. A noter que le système cible tourne sous FreeBSD, et que les fichiers génériques des systèmes dont j'ai l'habitude ne sont pas tout à fait les mêmes.
Première bonne nouvelle, le fichier /etc/passwd est accessible :
![poison-pwd](/img/poison/pwd2.PNG){:class="img-responsive"}
![poison-pwd](/img/poison/pwd3.PNG){:class="img-responsive"}

Celui-ci retourne trois noms d’utilisateurs qui peuvent s’avérer intéressants par la suite.

Je cherche ensuite à obtenir les hash des mots de passes de ces utilisateurs via l’équivalent du fichier /etc/shadow sous FreeBSD : pwd.db. Malheureusement, le fichier est accessible mais est inutilisables.

Je décide donc de retourner sur le fichier “pwdbackup.txt”, et celui-ci s’avère des plus intéressant :
![poison-pwd1](/img/poison/pwd-url.PNG){:class="img-responsive"}
![poison-pwd](/img/poison/pwd-content.PNG){:class="img-responsive"}

### Base64 is NOT encryption

A première vue, la chaîne de caractères obtenu est donc un mot de passe encodé 13 fois à la suite. Le caractère “=” à la fin de la chaîne semble servir de padding pour une chaîne en base64.
Puisque l’opération de décoder la chaîne 13 fois de suite à la main est des fastidieuse, j’ai décidé d’écrire un rapide script python s’en chargeant à ma place :
![poison-python](/img/poison/python.PNG){:class="img-responsive"}

En lançant le script, un mot de passe est dévoilé :
![poison-pwd-crack](/img/poison/pwd-crack.PNG){:class="img-responsive"}

Sachant que l’utilisateur “charix” est présent dans le fichier /etc/passwd de la machine, cette chaîne lui semble directement destinée. J'essaye alors de me connecter en SSH à la session de cet utilisateur avec le mot de passe fraîchement acquis :
![poison-entrypoint](/img/poison/user.PNG){:class="img-responsive"}

Voilà donc mon point d’entrée dans la machine ‘Poison’ au travers de l’utilisateur ‘charix’ !

## Escalation de privilège
Afin de maximiser mes chances de découvrir une faille système permettant d'élever mes privilège à un compte root, il est important d’énumérer méthodiquement le maximum d’informations possible sur l'environnement présent.
Je commence par vérifier les permissions de l’utilisateur actuel :
![poison-su](/img/poison/su.PNG){:class="img-responsive"}

Visiblement l’utilisateur actuel n’a aucun droits d'administration sur la machine, et est incapable d’accéder à d’autres comptes.

Je continue en listant les tâches exécutés automatiquements sur la machine :
![poison-cron](/img/poison/cron.PNG){:class="img-responsive"}

Après quelques recherches j’en parviens à la conclusion que ces tâches planifiés ne me seront d’aucunes utilitées.

Je remarque que le dossier home de l’utilisateur actuel comprend le flag de validation de la plateforme Hack The Box, mais également une archive zip protégée par un mot de passe :
![poison-home](/img/poison/home.PNG){:class="img-responsive"}

Je décide de me transférer l’archive en local afin de l'examiner de plus près :
![poison-archive](/img/poison/archive.PNG){:class="img-responsive"}

En essayant d’extraire l’archive protégée avec le mot de passe de session de l’utilisateur charix, j’obtiens un fichier au contenu non compatible UTF-8. Après traduction en ASCII du contenu j’obtiens l’énigmatique chaine de caractère suivante:
![poison-garbage](/img/poison/garbage.PNG){:class="img-responsive"}

Cette chaîne semble inutilisable, et sera certainement utilisée par la suite.
Je continue d’énumérer en me penchant cette fois-ci sur les services de la machine. Chose très intéressante, une session VNC tourne pour l’utilisateur root sur le port 5901 :
![poison-ps-vnc](/img/poison/ps-vnc.PNG){:class="img-responsive"}

En observant le contenu complet de la ligne du processus, je remarque l’argument “-auth /root/.Xaut /root/.vnc”. N’ayants pas de connaissances particulière sur les procédés d’authentifications VNC, je décide de me renseigner.
Il s’avère que l’authentification sur une session VNC se fait via mot de passe, et qu’une version encodée de ce mot de passe est présente en local dans le dossier caché “.vnc” de répertoire personnel de chaques utilisateurs.
Pour en avoir le coeur net, j'essaye d’obtenir les identifiants de connexion VNC pour l’utilisateur charix. Je trouve donc dans le fichier /home/charix/.vnc/passwd une chaîne qui correspond au mot de passe VNC encodée de charix :
![poison-hex-vnc](/img/poison/hex-vnc.PNG){:class="img-responsive"}

Quelques recherches sur le net me permettent de trouver un script capable de reverse le processus de chiffrement des mots de passes VNC : https://github.com/jeroennijhof/vncpwd.
Après avoir compilé le code, je décide de le tester sur l’utilisateur charix :
![poison-pass-vnc](/img/poison/pass-vnc.PNG){:class="img-responsive"}

Le script fonctionne correctement. Je me permet de créer une connexion VNC sur la machine cible afin de tester la pertinence du mot de passe révélé. La connexion VNC fonctionne correctement, mais ne m’apporte pas plus d’informations.

Par toute logique, il ne reste plus qu’à mettre la main sur le fichier d’authentification VNC du compte root afin d’accéder à sa session VNC. Malheureusement, après plusieurs échec ce fichier reste hors de portée. C’est alors que la présence de l’archive secret.zip prend tout son sens. En essayant de déchiffrer le contenu de ce fichier avec le script utilisée précédemment, j’obtiens le mot de passe VNC du compte root :
![poison-pass-vnc-root](/img/poison/pass-vnc-root.PNG){:class="img-responsive"}

Le fichier en question était depuis le début sous mes yeux, et son contenu est désormais plus clair. il ne reste plus qu’à se connecter à la session VNC root :
![poison-vnc-refuse](/img/poison/pass-vnc-refuse.PNG){:class="img-responsive"}
La session n’est pourtant pas accessible. Je décide donc de réfléchir aux méthodes potentiellements mises en places pour sécuriser cette connexion VNC. C’est l'article suivant, tiré de la documentation officiel VNC, qui m’indiquera le chemin à suivre : https://www.cl.cam.ac.uk/research/dtg/attarchive/vnc/sshvnc.html.

Il est en effet possible de rediriger les ports VNC via SSH afin d’ajouter une étape sécurisée en plus pour atteindre cette session. Je redirige donc le contenu du port distant 5901 via SSH vers mon port VNC 5902 via le compte utilisateur charix :
![poison-ssh-tunnel](/img/poison/pass-ssh-tunnel.PNG){:class="img-responsive"}

Ce qui me permet désormais d’accéder à la session VNC en local, via mon tunneling de port :
![poison-vnc-tunnel](/img/poison/pass-vnc-tunnel.PNG){:class="img-responsive"}

Et d’obtenir un shell root via la connexion distante VNC :
![poison-vnc-root](/img/poison/pass-vnc-root2.PNG){:class="img-responsive"}


La machine Poison est ainsi terminée ! Ce fut une machine très intéressante, spéciallement au niveau de l’élévation de privilège.
