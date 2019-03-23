---
layout: post
title: Malware traffic analysis (Cerber ransomware, DarkComet, Torrent)
categories: MalwareAnalysis
---

Cette série d'articles couvre différents exercices de la plateforme [malware-traffic-analysis](http://malware-traffic-analysis.net/). Ceux-ci nécessitent d'analyser un flux réseaux .pcap englobés un contexte précis, souvent en lien avec l'infection d'une machine par un malware.

* 2016-10-15 - TRAFFIC ANALYSIS EXERCISE - CRYBABY BUSINESSMAN
* 2018-07-15 - TRAFFIC ANALYSIS EXERCISE - OH NOES! TORRENTZ ON OUR NETWORK!
* 2018-02-13 - TRAFFIC ANALYSIS EXERCISE - OFFICE WORK

### CRYBABY BUSINESSMAN (2016-10-15)

[malware-traffic-analysis.net/2016/10/15/index.html](http://malware-traffic-analysis.net/2016/10/15/index.html)
![picture](/img/traffic-analysis/1/1.PNG){:class="img-responsive"}

D'après le texte introductif sur la situation, il s'agira ici d'étudier comment un utilisateur s'est retrouvé infecté avec un ransomware.


En regardant rapidement le trafic http disponible, certaines pages visitées ne semblent pas en adéquations avec un environnement professionnel.
En observant de plus près les requêtes  en question, il semblerait que l'utilisateur cherchait à obtenir des photos coquines de célébrités :
![picture](/img/traffic-analysis/1/2.PNG){:class="img-responsive"}


Une fois sur le site _unwrappedphotos.com_, une ressource distante a été chargée par le serveur web. L'hypothèse que le site ait été hijhacké est tout à fait possible. La requête malicieuse est faite en direction de l'URl _rew.kaghaan.com_.
![picture](/img/traffic-analysis/1/3.PNG){:class="img-responsive"}


La ressource est en fait le tristement célèbre exploit kit *Flash Rig*, téléchargé avec l'extension _.swf_ (pour un fichier flash):
![picture](/img/traffic-analysis/1/4.PNG){:class="img-responsive"}


L'exploit kit va ensuite télécharger le ransomware Cerber. La requête contenant le fichier binaire du ransomware est la suivante:
![picture](/img/traffic-analysis/1/5.PNG){:class="img-responsive"}


A noter que le fichier est chiffré, d'où l'absence de préfixe MS-DOS ou de chaines de caractères reconnaissables.
Une fois sur le système, le fichier est déchiffré, ce qui va alerter l'EDR:
![picture](/img/traffic-analysis/1/6.PNG){:class="img-responsive"}


Le ransomware cerber cherche ensuite à atteindre un serveur web distant, afin de présenter la page de rançon à la victime. Ce comportement permet de confirmer le fait que le ransowmare est en effet le ransowmare cerber, et pas juste un faux positif:
![picture](/img/traffic-analysis/1/7.PNG){:class="img-responsive"}


La page en question est hébergée à l'adresse ffoqr3ug7m726zou.19jmfr.top.
D'après quelques recherches, la victime devrait obtenir une note de rançon très similaire à la suivante:
![picture](/img/traffic-analysis/1/8.PNG){:class="img-responsive"}
![picture](/img/traffic-analysis/1/9.PNG){:class="img-responsive"}


Enfin, l'utilisateur a cherché à acheter des bitcoin pour déchiffrer ses fichiers:
![picture](/img/traffic-analysis/1/10.PNG){:class="img-responsive"}

La situation peut être résumé par la schéma suivante:
![picture](/img/traffic-analysis/1/11.PNG){:class="img-responsive"}

IOC:
107.161.95.138
ffoqr3ug7m726zou.le2brr.bid
-
173.254.231.111
ffoqr3ug7m726zou.19jmfr.top
-
109.234.36.251
rew.kaghaan.com

### OH NOES! TORRENTZ ON OUR NETWORK! (2018-07-15)

[(http://malware-traffic-analysis.net/2018/07/15/index.html]((http://malware-traffic-analysis.net/2018/07/15/index.html)
Instructions:
![picture](/img/traffic-analysis/2/1.PNG){:class="img-responsive"}


IP: 10.0.0.201
MAC: 00:16:17:18:66:c8 (MSI)
Computer Name: BLANCO-DESKTOP
![picture](/img/traffic-analysis/2/2.PNG){:class="img-responsive"}


LDAP Account: elmer.blanco
![picture](/img/traffic-analysis/2/3.PNG){:class="img-responsive"}


Windows version: Windows 10 64 bits (from http request from the blanco’s computer).
![picture](/img/traffic-analysis/2/4.PNG){:class="img-responsive"}
