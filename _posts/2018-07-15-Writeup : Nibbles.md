---
title: Writeup : Nibbles
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

La visite de l’index web n’apporte rien de concret à première vue :
