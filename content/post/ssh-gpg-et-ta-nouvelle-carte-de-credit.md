+++
authors = [ "eau" ]
description = ""
draft = false
tags = [ "ssh", "auth", "bofh", "gpg", "smartcard", "loose", "openpgp" ]
topics = [ ]
date = "2016-11-20T13:05:55+01:00"
title = "ssh gpg et ta nouvelle carte de credit"

+++


**Update 22/11/2016** : rajouts des liens sur le hardware + une section fun sur l'abus d'une fonctionnalite de l'implementation d'un HSM (heureusement qu'il est open cela dit...)

## C'est quoi l'histoire?

Crise de la trentaine..

Comme beaucoup de bouseux/geeks de l'opensource (dont je fais partie), j'ai beaucoup de machines qui "traînent" et servent _sporadiquement_ pour certains trucs elec-digitale/FPGA/OTG/ARM/decouverte/blabla (comme le [novena](https://www.crowdsupply.com/sutajio-kosagi/novena)) ou certaines "tâches"/"test" (comme bientôt l'[ORWL](https://www.crowdsupply.com/design-shift/orwl)), j'ai plusieurs "laptops" selon que je bosse (et sur quoi) ou que je "traîne" sur mes projets persos (ah ça je traîne..) ou de l'apprentissage ([yeelong](https://www.amazon.com/Screen-Lemote-Yeeloong-8101_B-Netbook/dp/B005XH10NQ/ref=sr_1_2?ie=UTF8&qid=1319989211&sr=8-2?tag=electronicfro-20) pour le MIPS ou le novena pour l'arch ARM). 

Le truc relou c'est que je voulais pouvoir me logguer (ah mon IRC...) souvent depuis la machine où je suis sur le moment (OS hétérogènes: bsd, linux, os/x) mais sans transférer/laisser trainer la/les clefs SSH et/ou GPG et en limitant (un peu, faut pas rêver non plus.) les risques qu'un putain de malware/bouseux concurrent (oui on sait jamais) me les tapent simplement...

Alors j'ai juste cherché, vite fait, comment je pourrais faire.. je me suis aussi demandé pourquoi je l'ai pas fait avant...  je me forçais à me logguer depuis UNE seule machine trusted blablabla... bref.

## Comment?

Y a pleins d'approches "potentielles":

* un disque externe/USB chiffré qui se monte automagiquement, mais c'est relou selon les FS/crypto supportés, pas multiplateforme et bon on peut encore te taper ta clef privée (genre en mémoire) meme si elle est chiffrée avec ta gentille passphrase.
* un HSM, une variante du FS qui te file une interface d'accès qui "devrait" marcher partout.. mais bon.. on sait ce que c'est.. driver, interface peu/pas portables/etc.. ou qui finalement se comporte comme un FS avec une interface pseudo-custom..
* [keybase.io](https://keybase.io) qui te propose un moyen de dealer avec tes clefs GPG (mais pas que..) de manière (IMHO) assez bordelique (mais pas que..) et complexe, mais avec une jolie CLI et une jolie interface _yoyoyo-je-suis-une-startup-à-SF-donc-jai-forcément-la-solution-to-build-a-better-world_
* smartcards (hmm?! comment ça marche?)
* copier-partout-et-croire-en-dieu-ou-des-esprits-que-tu-te-feras-jamais-défoncer
* what else? (vous pouvez commenter!)

Apres une brillante analyse (qui me caractérise bien moi, le bon et prétentieux bouseux opensourceux), la définition d'un threat model, le calcul du risque associé et la production de slides incroyables pour la prochaine conf à la con ou j'irais vomir/étaler ma mediocrite pour me vendre un peu plus en tentant de changer de statut (e.g. passer du cassoulet LIDL au cassoulet Williams Saurin, c'est une évolution en soi).

J'en suis venu aux smartcards, qui loin d'être parfaites, proposent quand même un compromis "sympa" pour peu que l'on _TRUSTE_ le hardware, le protocole (CCID) et son implementation (il y a des choses intéressantes d'ailleurs.. ;)), finalement, je truste mon laptop et tous ses composants même les plus "blobesques" (hélas..) et critiques (ethernet / BIOS / etc..?!).


## Setup..

Honnêtement, je vais pas reprendre le setup pas à pas, j'ai compilé une serie de liens qui m'ont aidé à piger/faire mon setup, ca devrait largement suffire pour démarrer.

Voilà les liens qui m'ont aidés:

* https://wiki.fsfe.org/TechDocs/CardHowtos/CardWithSubkeysUsingBackups
* https://www.gnupg.org/howtos/card-howto/en/smartcard-howto.html
* https://budts.be/weblog/2012/08/ssh-authentication-with-your-pgp-key
* https://www.sidorenko.io/blog/2014/11/04/yubikey-slash-openpgp-smartcards-for-newbies/
* https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/
* https://www.preining.info/blog/2016/04/gnupg-subkeys-yubikey/
* https://spin.atomicobject.com/2014/02/09/gnupg-openpgp-smartcard/
* https://incenp.org/notes/2014/gnupg-for-ssh-authentication.html
* https://openpgpcard.org/makecard/
* https://lists.gnupg.org/pipermail/gnupg-devel/2016-January/030682.html
* https://www.gnupg.org/documentation/manuals/gnupg/GPG-Configuration.html
* https://wiki.archlinux.org/index.php/GnuPG
* https://wiki.debian.org/Smartcards/OpenPGP#SSH
* https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/
* https://www.programmierecke.net/howto/gpg-ssh.html
* https://www.makomk.com/2016/01/23/openpgp-crypto-token-using-gnuk/
* https://alexcabal.com/creating-the-perfect-gpg-keypair/

En ce qui concerne les smartcards, voila ce que j'ai listé:

* [Yubikey v4](https://www.yubico.com/products/yubikey-hardware/yubikey4/), elles ont plusieurs mode de fonctionnement qui collaborent (FIDO/U2F/CCID/HSM blablabla), en gros yubikey (si vous les ouvrez) c'est 2 MCU, un "secure" MCU qui gère la crypto/storage, NXP chaisplus combien et un MCU qui gère la comm USB/CCID, [quelqu'un l'avait fait avant moi](http://www.hexview.com/~scl/neo/), ça ne m'a pas empêché de l'ouvrir... oui, vous avez compris j'adore ouvrir les boîtes de cassoulets mais lui au moins, il a pris/poste des photos.
* [OpenPGP card v2.1](http://www.g10code.com/p-card.html) on les trouve un peu partout, un peu lentes mais marchent très bien (recommandées par la FSF).
* [Nitrokey](https://shop.nitrokey.com/shop/product/nitrokey-pro-3), une implementation "opensource"/freemium d'une smartcard à base de MCU / gnuk.
* [Gnuk](https://www.gniibe.org/pdf/fosdem-2012/gnuk-fosdem-20120204.pdf), MCU avec le code opensource Gnuk, qui utilise une [lib de threading](https://github.com/jj1bdx/chopstx/blob/master/README) comme OS et fait tourner les opérations de crypto et la minipile USB/CCID pour repondre comme une smartcard.
* what else?

Hardware:

* https://www.seeedstudio.com/FST-01-with-White-Enclosure-p-1279.html
* https://github.com/jj1bdx/chopstx/blob/master/README
* http://www.fsij.org/doc-gnuk/intro.html#usages
* https://github.com/ggkitsas/gnuk
* https://wiki.debian.org/Smartcards#Some_common_cards



## les merdes, y a toujours des merdes...

* _keytocard_ ca MOVE (donc efface) les clefs privées de votre _keyring_ et les remplace par un _stub_, rappelez vous en (genre avant le backup).
* gnupg v1 VS gnupg v2.0 VS gnupg v2.1 ça cohabite, mais pas très bien, il y a des VRAIES différences et ça peut causer quelques emmerdes, ne vous faites pas avoir. En gros démarrez avec avec 2.1.X et stick to it now... c'est pas parfait mais ça évolue...
* gnupg 2.1.15 l'avant dernière release (celle avec laquelle je me suis battu...) avait un bug où la wrapper lib de threading (npth) etait utilisée AVANT d'être d'initialisée et _gpg-agent_, _scdaemon_, _dirmngr_ se mangeaient un bel _assert()_ et donc ne fonctionnaient pas, mais ça compilait sans soucis, du coup la release était une release... mais inutilisable.. j'ai fait un patch, mais 2.1.16 a été releasé entre temps qui resoud ce BUG mais n'est pas encore forcément dans tous les repositories de packages.
* gnupg v2.1 crée des _stubs_ des clefs privée À CHAQUE instanciation de l'agent (--card-status par exemple), si vous avez plusieurs smartcards avec la MÊME clef, oubliez pas de tuer l'agent et de "cleaner" (rm -rf $HOME/.gnupg/private-keys-v1.d/\*) sinon il va associer/garder les stubs de la smartcard précédente et au moment de l utilisation vous demander d'insérer la dite smartcard.
* impossible d'avoir le _pinentry_ au moment de mon ssh, il me faut "préparer" l'agent, un petit _gpg2 --card-status_, suivi d'un _gpg2 -d <unfichierchiffreavecmaclefGPG>_ le tout avec ma carte insérée (+ le bon PIN) et juste apres je peux faire mon ssh et ça passe.
* _$HOME/.gnupg/scd-events_ peut être exécuté à chaque instanciation de l'agent et/ou insertion d'une SC, attention, le lecteur SC standard USB et une yubikey ne se comportent pas de manière identique, pour avoir un comportement consistant (genre killer l'agent ou cleaner les stubs est pas évident) il faut travailler un peu..
* veillez comme toutes les docs le disent à BIEN FAIRE DES BACKUPS apres les différentes étapes, création masterkey, subkeys, etc.. et à mettre ces backups safe et offline, sinon tout ça ne sert à rien.

## Encore des emmerdes... 

Le probleme est toujours le meme, le _TRUST_ sur une plateforme hardware dont vous ne connaissez RIEN et ou le joli autocollant _SECURE_ est appose pour bien vous
faire sentir au chaud, je crois qu'il serait sympa d'avoir une review de plus des implementations ouvertes et peut-etre proposer des updates (soft ET hard) afin d'avoir des "smartcards" qui tiennent le coup.

On nous a gentillement signale cette "attaque", ce "post" interessant, mais c'est plutot un l'abus d'une fonctionnalite qui a mon HUMBLE avis ne devrait pas etre implementee.

Ce monsieur s'est bien amuse, [d'abord il joue avec son device..](https://raymii.org/s/articles/Nitrokey_Start_Getting_started_guide.html), puis il [gratte un peu, par curiosite](https://raymii.org/s/tutorials/FST-01_firmware_upgrade_via_usb.html), puis [la encore](https://raymii.org/s/tutorials/Nitrokey_gnuk_firmware_update_via_DFU.html) plus a droite, plus a droite, un peu a gauchhhheee, laaaaaaaaaaaaaAAAAaaaa et hop [decouvrir une belle croute de fonctionnalite](https://raymii.org/s/articles/Decrypt_NitroKey_HSM_or_SmartCard-HSM_private_keys.html).

Ca laisse reveur quand on reflechit un tout petit peu a l'utilite d'un HSM (je rappelle ce n'est pas une SmartCard CCID bla)



## Conclusion

Mon français sent des pieds mais ça marche, mes clefs ne sont plus "online", à part sur mes 2 SC, la SC donne une interface pour signer/chiffrer (via CCID) mais ne permet pas de récupérer les clefs privées directement (comme sur un FS), je peux plugger ma SC sur différents systèmes (BSD, Linux, OS/x) mon auth est faisable sans pour autant compremettre aussi facilement mes clefs, je n'ai rien à copier et j'ai un élément hardware en plus de mon PIN, pour chiffrer mes datas et pour m'auth sur mes machines, plus feignant que ça tu meurs.

Un seul bémol, il faut un gnupg 2.0.X ou 2.1.16+ et c'est pas encore super "smooth", voilà en vous remerciant.
