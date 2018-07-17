---
layout: post
title: Sysadmin - Déploiement de Honeypots
categories: Sysadmin
---
Un honeypot est une ressource informatique volontairement vulnérable destinée à attirer des adversaires malveillants. Dans le cas présent, l’infrastructure servira à collecter des échantillons de malwares actuellement en circulation et à observer les différentes tentatives d’attaques les plus utilisées. Un serveur principal servira de console de supervision pour deux honeypots. Le premier sera un honeypot tournant avec l’IDS Snort pour détecter les attaques réseaux bas niveaux. Le second sera une plateforme vulnérable aux exploits du service Windows SMB afin de recueil le maximum de codes malveillants. Enfin, en local, une machine d’analyse de malwares sera déployées pour observer les échantillons recueillis.

## Installation du serveur MHN.
### Préparation du serveur
De l’acronyme Modern Honey Network, le projet MHN permet de déployer différents honeypots. Le projet open-source est disponible sur ce répo [Github](https://github.com/threatstream/mhn).
Tous les serveurs seront hébergés chez DigitalOcean.
Dans un premier lieu, il faudra donc créer la machine chez l’hébergeur. Le serveur MHN ne nécessitant pas énormément de ressources, l’offre de base suffit amplement :
![mhn-droplet](/img/sysadmin/mhn/droplet){:class="img-responsive"}

L’OS par défaut, à savoir Ubuntu 16.04.4 x64 sera lui aussi suffisant. Je passerai volontairement les étapes de configuration élémentaires de la machine. A noter qu’une authentification via clé SSH est vivement recommandée.
La première étape consistera à créer un nouvel utilisateur sur la machine :
> adduser user

Puis à lui donner les droits root avec la commande visudo :
> root    ALL=(ALL:ALL) ALL                        
> user    ALL=(ALL:ALL) ALL

Il sera ensuite nécessaire, au sein du répertoire courant du nouvel l’utilisateur, de télécharger le projet MHN avec la commande git, puis de l’installer :
> git clone https://github.com/threatstream/mhn.git                     
> cd mhn                     
> sudo ./install.sh

Pendant l’installation, plusieurs informations nous serons demandée. La première concerne l’activation du mode “débug” de la solution. Ceci n’a pas d’importance dans notre cas.

Il sera ensuite demandé d’entrer une adresse mail superutilisateur et un mot de passe qui servira d’identifiant pour se connecter sur la plateforme.
Les autres prompts concernent l’utilisation d’un serveur de messagerie pour envoyer automatiquement des alertes.
Je n’ai pas décidé d’en renseigné un, mais les informations sont suffisamment claires pour le faire sans aide supplémentaire :
![mhn-screen1](/img/sysadmin/mhn/infos){:class="img-responsive"}

Enfin, il sera possible d’intégrer la solution Splunk et le stack ELK à notre projet. Puisque la machine serveur est relativement faible en terme de performance, je n’ai pas choisis d’installer et d’activer ces deux modules.

Après plusieurs minutes, l’installation se terminera d’elle même. Il sera désormais possible de se connecter au service web MHN de la machine distante pour débuter l’administration des futurs honeypots :
![mhn-welcome](/img/sysadmin/mhn/welcome){:class="img-responsive"}

### Installation de la sonde Snort
Le honeypot Snort ne nécessite que quelques minutes de configuration. Toujours sur le serveur MHN, il faudra se rendre dans l’onglet “Deploy” :
![mhn-snort](/img/sysadmin/mhn/snort){:class="img-responsive"}

Sous cet onglet, plusieurs script de déploiement sont disponibles. Je choisirai ici la sonde Snort dans le menu déroulant. Il ne restera plus qu'à exécuter la commande fraîchement générée par le serveur, sur la première machine honeypot, pour que celle-ci se transforme en honeypot Snort :
![mhn-snort1](/img/sysadmin/mhn/snort1){:class="img-responsive"}

Après avoir renseignée cette commande sur le premier serveur honeypot, notre nouvelle sonde devrait être visible sous l’onglet “Sensors” de l’interface web MHN.
Après quelques secondes, nous pouvons déjà observer des événements relatifs à des scans automatiques de notre machine :
![mhn-snort2](/img/sysadmin/mhn/snort2){:class="img-responsive"}

Les évènements sont visibles sous l’onglet “Payloads”, puis en sélectionnant la regex “snort.alerts”.
Il est aussi possible d’observer un condensé des événements dans l’onglet “Attacks” :
![mhn-snort3](/img/sysadmin/mhn/snort3){:class="img-responsive"}

Enfin, un résumé des positions géographiques des attaquants est disponible en live sous l’onglet “Map” :
![mhn-snort4](/img/sysadmin/mhn/snort4){:class="img-responsive"}

### Installation de la sonde Dionaea
Le second serveur honeypot va simuler un service SMB vulnérable, ce qui va permettre de collecter différents malwares, en vue de les analyser de manière dynamique par la suite. Cette fois-ci, l’installation et la configuration de la sonde se fera de manière manuelle. Commençons par créer la sonde côté serveur MHN via l’onglet “Sensors” puis “Add sensor”. Il suffira de renseigner les trois champs, même si le contenu n’a pas d’importance :
![mhn-dio](/img/sysadmin/mhn/dio){:class="img-responsive"}

Un numéro d’identification unique (uuid) nous sera généré. Attention à bien le prendre en note, il nous sera nécessaire pour la prochaine étape.
Pour terminer de déclarer notre nouvelle source, il sera nécessaire d’ouvrir un shell sur le serveur MHN, puis de déclarer les variables suivantes :
> IDENT=<UUID noté plus tôt>              
> SECRET=<Mot de passe aléatoire>                
> PUBLISH_CHANNELS="dionaea.connections,dionaea.capture,mwbinary.dionaea.sensorunique,dionaea.captures,dionaea.capture.anon"              
> SUBSCRIBE_CHANNELS=""

Il est désormais possible de terminer l’enregistrement de la sonde en lançant le script MHN dédié :
> cd /opt/hpfeeds         
> source env/bin/activate            
> cd broker                
> python add_user.py "$IDENT" "$SECRET" "$PUBLISH_CHANNELS" "$SUBSCRIBE_CHANNELS"

Passons maintenant à l’installation des composants du honeypot sur le serveur dédié. Commençons par mettre à jour curl :
> sudo apt-get build-dep curl             
> mkdir ~/curl             
> cd ~/curl            
> wget http://curl.haxx.se/download/curl-7.50.2.tar.bz2                  
> tar -xvjf curl-7.50.2.tar.bz2                
> cd curl-7.50.2                 
> ./configure     
> make                   
> sudo make install                   
> sudo ldconfig

Il sera ensuite nécessaire de mettre à jour les sources avant d’installer dionaea :
> sudo apt-get update               
> sudo apt-get dist-upgrade                
> sudo apt-get install software-properties-common                   
> sudo add-apt-repository ppa:honeynet/nightly

> sudo apt-get update              
> sudo apt-get install dionaea

Afin de capturer des samples de malwares, il est nécessaire de configurer un vecteur d’attaque sur notre honeypot. Dans le cas présent, ce sera un service SMB vulnérable. Pour activer ce service, il faudra éditer le fichier smb.yaml situé sous /opt/dionaea/etc/dionaea/services-enabled/ :
{% highlight yaml linenos %}
- name: smb
  config:

    os_type: 4

     ## Windows 7 ##
    native_os: Windows 7 Professional 7600
    native_lan_manager: Windows 7 Professional 6.1
    shares:
      ADMIN$:
        comment: Remote Admin
        path: C:\\Windows
        type: disktree
      C$:
        coment: Default Share
        path: C:\\
        type:
          - disktree
          - special
      IPC$:
        comment: Remote IPC
        type: ipc
      Printer:
        comment: Microsoft XPS Document Writer
        type: printq
{% endhighlight %}

Enfin, pour activer le service, nous allons copier ce fichier de configuration dans le dossier des services actifs :
> sudo cp ihandlers-available/hpfeeds.yaml ihandlers-enabled/hpfeeds.yaml

Pour terminer, il faudra modifier le fichier de configuration de notre honeypot de façon à ce que celui-ci remonte les informations sur notre serveur MHN central :
> nano ihandlers-enabled/hpfeeds.yaml

{% highlight yaml linenos %}
- name: hpfeeds
  config:
    server: "IP du serveur MHN"
    port: 10000
    ident: "UUID de la sonde"
    secret: "Mot de passe créé lors de l’enregistrement de la sonde"
{% endhighlight %}

Il ne reste plus qu'à redémarrer le service pour activer notre second honeypot :
> service dionaea restart

Après quelques minutes, les premiers hits dans le serveur devraient apparaitres :
![mhn-dio1](/img/sysadmin/mhn/dio1){:class="img-responsive"}

Les premiers malwares devraient ensuite être uploadés sur notre honeypot sans plus attendre. Les informations relatives aux malwares ayants infectés notre machine sont disponibles sous l’onglet “Payload” avec la regex “dionaea.capture” :
![mhn-dio2](/img/sysadmin/mhn/dio2){:class="img-responsive"}

Les fichiers binaires ne sont pas envoyés aux serveur MHN, en revanche ceux-ci sont stockés sous /opt/dionaea/var/dionaea/binaries/ sur la machine dionaea :
![mhn-dio3](/img/sysadmin/mhn/dio3){:class="img-responsive"}

Une rapide recherche de premiers samples nous ayant atteint sur VirusTotal (via le md5 des binaires) nous montre qu’à l’heure de l’écriture de cette documentation, de nombreuses variantes du ransomware WannaCry sont encore monnaie-courantes :
![mhn-dio4](/img/sysadmin/mhn/dio4){:class="img-responsive"}

Nous sommes désormais prêt à récolter patiemment de nombreux samples de malwares avant de les analyser manuellement par la suite.
