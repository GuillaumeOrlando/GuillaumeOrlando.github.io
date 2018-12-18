---
layout: post
title: Malware Generic Manual Unpacking 
categories: Recherches
---

UPX est un packer reconnu qui peut faire office de très bon entraînement à l'unpacking de malware : https://upx.github.io/. Le malware analysé sera ici au format PE, et l’aspect unpacking sera uniquement couvert.
### Identification
Il existe de nombreuses manières de détecter si un exécutable est packé.
Puisque la plupart des packers modifient les sections du binaire, l’observation des sections du fichier analysé peut s’avérer intéressante:
![image-left](/img/PMA/chap1/unpack/UPX/A.PNG)
Premièrement, les noms des sections est un bon indicateur de l’état du fichier. De plus, certains packers donnerons quelques informations via le nom des sections attribuées. Qu’il s’agisse directement du nom du packer, ou d’un nom permettant de remonter au packer, l’information reste très importante.
Afin de confirmer la théorie, l’utilitaire PEiD peut être très utile. Celui-ci recherche des signatures de packers connus, permettant ainsi de rapidement identifier le packer utilisé:
![image-left](/img/PMA/chap1/unpack/UPX/B.PNG)
Enfin, il est possible d’observer l’entropie du fichier analysé. L’entropie est une fonction de mesure de l’aléatoire et de désordre. Dans le cas présent, elle permet de mesure, sur une échelle de 0 à 8, la vraisemblabilité que le contenu du fichier soit chiffré ou obfusqué:
![image-left](/img/PMA/chap1/unpack/UPX/C.PNG)
Le pic centrale avoisinant les 7,5 ne laisse aucun doutes: cette section du binaire est certainement compressée / chiffrée / obfusquée. A titre de comparaison, l’entropie d’un fichier non packé dispose d’une courbe similaire à celle-ci:
![image-left](/img/PMA/chap1/unpack/UPX/D.PNG)
Dans le cas du fichier analysé, il n’y à aucuns doutes que celui-ci est packé, avec l’utilitaire UPX.

### Recherche de l’OEP
Afin de dumper le contenu du malware en claire, il sera nécessaire de l’ouvrir avec OllyDbg ou tout autre débugger équivalent.
Le but sera ici de trouver le point d’entrée (entrypoint ou OEP) du malware. L’entrypoint sur lequel le malware débute actuellement est en fait le stub du packer. Cette portion de code est responsable du déchiffrement de la version en clair recherchée. C’est donc tout naturellement les premières instructions exécutés par le binaire:
![image-left](/img/PMA/chap1/unpack/UPX/E.PNG)
Beaucoup de packers peuvent être manuellement contournés avec cette même méthode (et non seulement UPX, mais aussi ASPacker, NSPacker et de très nombreux autres). Le but sera ici de cherche un saut (instruction “JMP” correspondant à l’opcode E9) qui mènera vers une adresse mémoire distante. Ce saut nous mènera jusqu’à l’entrypoint du malware:
![image-left](/img/PMA/chap1/unpack/UPX/F.PNG)
Puisqu’il peut être très fastidieux de chercher ce saut, il est aussi possible d’observer le registre ESP au lancement du programme. Ce registre contiendra l’adresse de retour du stub. A ce moment de l'exécution du packer, cela signifie que le déchiffrement / la décompression du malware est terminée, et que celui-ci est prêt à être exécuté. C’est à ce moment que nous allons intercepter le contenu en clair de ce dernier.
Un fois cette adresse identifiée, il faudra afficher la portion mémoire associée via l’option “Follow in Dump”:
![image-left](/img/PMA/chap1/unpack/UPX/G.PNG)
Il ne reste plus qu'à placer un breakpoint dès que cette portion mémoire est utilisée:
![image-left](/img/PMA/chap1/unpack/UPX/H.PNG)
En reprenant le flow d'exécution du binaire, le breakpoint est bien atteint juste avant le JMP remarqué plus haut:
![image-left](/img/PMA/chap1/unpack/UPX/I.PNG)
Il ne reste plus qu’à exécuter pas à pas les instructions restantes pour être emmené à l’OEP réel du malware déchiffré:
![image-left](/img/PMA/chap1/unpack/UPX/J.PNG)
L’adresse de l’OEP est à noté soigneusement si le dump souhaite être fait manuellement. Ici, cette adresse est <0x00401190>.

### Dump du malware
Tout comme la phase précédent, le dump du malware unpacké peut être fait automatiquement ou manuellement. L’inconvénient d’un dump manuel réside dans le fait que l’IAT (Import Address Table) doit être reconstruit à la main. Sans cette reconstruction, les adresse des fonctions ne vont plus coïncider avec celles appelés par le binaire, ce qui va empêcher l'exécution de celui-ci.
Pour se faciliter la tâche, le plugin OllyDump peut être utilisé:
![image-left](/img/PMA/chap1/unpack/UPX/K.PNG)
Le nouvel entrypoint doit être renseigné, avant de dumper le programme. Attention à bien activer le reconstruction de l’iAT avec la case “Rebuild Import”:
![image-left](/img/PMA/chap1/unpack/UPX/L.PNG)
Un fois ce dump enregistré, il ne reste qu'à confirmer que celui-ci est fonctionnel et exploitable.

### Vérification de l’état du dump
Après l’extraction, il peut être intéressant de vérifier que le malware n’est plus packé. Une comparaison de la taille du fichier initial avec le dump permet de vérifier que le fichier obtenu n’est plus packé:
![image-left](/img/PMA/chap1/unpack/UPX/M.PNG)
Le niveau d’entropie permet lui aussi de confirmer l’état du nouveau binaire:
![image-left](/img/PMA/chap1/unpack/UPX/N.PNG)
L’entropie est ici très basse, ce qui correspond bien à un programme en claire.
Pour finir, PEiD détecte bien le fichier comme une application C++ native, non packée:
![image-left](/img/PMA/chap1/unpack/UPX/O.PNG)
Attention, le nom des sections n’a pas changé, mais cela n’indique pas que le programme est encore packé. C’est pourquoi le nom seul des sections ne permet pas d’effectuer un diagnostic pertinent quand à l’état du malware.
La phase d’analyse statique du binaire peut désormais débuter.
