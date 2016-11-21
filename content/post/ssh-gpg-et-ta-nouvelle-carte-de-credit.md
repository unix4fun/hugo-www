+++
authors = [ "eau" ]
description = ""
draft = true
tags = [ "ssh", "auth", "bofh", "gpg", "smartcard", "loose", "openpgp" ]
topics = [ ]
date = "2016-11-20T13:05:55+01:00"
title = "ssh gpg et ta nouvelle carte de credit"

+++

## C'est quoi l'histoire?

Crise de la trentaine..

Comme beaucoup de bouseux/geek de l'opensource (dont je fais parti), j'ai beaucoup de machines qui "trainent" servent _sporadiquement_ pour certains trucs elec-digitale/FPGA/OTG/ARM/decouverte/blabla (comme le [novena](https://www.crowdsupply.com/sutajio-kosagi/novena)) ou certaines "taches"/"test" (comme bientot l'[ORWL](https://www.crowdsupply.com/design-shift/orwl)), j'ai plusieurs "laptops" selon que je bosse (et sur quoi) ou que je "traine" sur mes projets persos (ah ca je trainee..) ou de l'apprentissage ([yeelong](https://www.amazon.com/Screen-Lemote-Yeeloong-8101_B-Netbook/dp/B005XH10NQ/ref=sr_1_2?ie=UTF8&qid=1319989211&sr=8-2?tag=electronicfro-20) pour le MIPS ou le novena pour l'arch ARM). 

Le truc relou c'est que je voulais pouvoir me logguer (ah mon IRC...) souvent depuis la machine ou je suis sur le moment (OS heterogenes: bsd, linux, os/x) mais sans transferer/laisser trainer la/les clefs SSH et/ou GPG et en limitant (un peu, faut pas rever non plus.) les risques qu'un putain de malware/bouseux concurrent (oui on sait jamais) me les tapent simplement...

Alors j'ai juste cherche, vite fait, comment je pourrais faire.. je me suis aussi demande pourquoi je l'ai pas fait avant...  je me forcais a me logguer depuis UNE seule machine trusted blablabla...bref.

## Comment?

Y a pleins d'approches "potentielles":

* un disque externe/USB chiffre qui se monte automagiquement, mais c'est relou selon les FS/crypto supportes, pas multiplateforme et bon on peut encore te taper ta clef privee (genre en memoire) meme si elle est chiffree avec ta gentille passphrase.
* un HSM, une variante du FS qui te file une interface d'access qui "devrait" marcher partout.. mais bon.. on sait ce que c'est..driver, interface peu/pas portables/etc..
* keybase.io qui te propose un moyen de dealer avec tes clefs GPG (mais pas que..) de maniere (IMHO) assez bordelique (mais pas que..) et complexe, mais avec une jolie CLI et une jolie interface _yoyoyo-je-suis-une-startup-a-SF-donc-jai-forcement-la-solution-to-build-a-better-world_
* smartcards (hmm?! comment ca marche?)
* copier-partout-et-croire-en-dieu-ou-des-esprits-que-tu-te-feras-jamais-defoncer
* what else? (vous pouvez commenter!)

Apres une brillante analyse (qui me caracterise bien moi, le bon et pretentieux bouseux opensourceux), la definition d'un threat model, le calcul du risque associe et la production de slides incroyables pour la prochaine conf a la con ou j'irais vomir/etaler ma mediocrite pour me vendre un peu plus en tentant de changer de statut (e.g. passer du cassoulet LIDL au cassoulet Williams Saurin, c'est une evolution en soi).

J'en suis venu au smartcards, qui loin d'etre parfaites, proposent quand meme un compromis "sympa" pour peu que l'on _TRUSTE_ le hardware, le protocole (CCID) et son implementation (il y a des choses interessantes d'ailleurs.. ;)), finalement, je truste mon laptop et tout ses composants meme les plus "blobesques" (helas..) et critiques (ethernet / BIOS / etc..?!).


## Setup..

Honnetement, je vais pas reprendre le setup pas a pas, j'ai compile une serie de liens qui m'ont aide a piger/faire mon setup, ca devrait largement suffire pour demarrer.

En ce qui concerne les smartcards, voila ce que j'ai liste:

* Yubikey v4, elles ont plusieurs mode de fonctionnement qui collaborent (FIDO/U2F/CCID/HSM blablabla), en gros yubikey (si vous les ouvrez) c'est 2 MCU, un "secure" MCU qui gere la crypto/storage, NXP chaisplus combien et un MCU qui gere la comm USB/CCID, [quelqu'un l'avait fait avant moi](http://www.hexview.com/~scl/neo/), ca ne m'a pas empeche de l'ouvrir... oui, vous avez compris j'adore ouvrir les boites de cassoulets [http://www.hexview.com/~scl/neo/](http://www.hexview.com/~scl/neo/) mais lui au moins, il a pris des photos.
* OpenPGP card v2.1 on les trouve un peu partout, un peu lentes mais marchent tres bien (recommandees par la FSF).
* Nitrokey, une implementation "opensource"/freemium d'une smartcard a base de MCU / gnuk.
* Gnuk, MCU avec le code opensource Gnuk, qui utilise une lib de threading comme OS et faire tourner les operations de crypto et la minipile USB/CCID pour repondre comme une smartcard.
* what else?


Voila les liens qui m'ont aide:

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
* https://alexcabal.com/creating-the-perfect-gpg-keypair/

Hardware:

* https://www.seeedstudio.com/FST-01-with-White-Enclosure-p-1279.html
* https://github.com/jj1bdx/chopstx/blob/master/README
* http://www.fsij.org/doc-gnuk/intro.html#usages
* https://github.com/ggkitsas/gnuk
* https://wiki.debian.org/Smartcards#Some_common_cards



## les merdes, y a toujours des merdes...

* _keytocard_ ca MOVE (donc efface) les clefs privees de votre _keyring_ et les remplace par un _stub_, rappelez vous en (genre avant le backup).
* gnupg v1 VS gnupg v2.0 VS gnupg v2.1 ca cohabite, mais pas tres bien, il y a des VRAIS differences et ca peut causer quelques emmerdes, ne vous faites pas avoir. en gros demarrez avec avec 2.1.X et stick to it now... c'est pas parfait mais ca evolue...
* gnupg 2.1.15 l'avant derniere release (celle avec laquelle je me suis battu...) avait un bug ou la wrapper lib de threading (npth) etait utilise AVANT d'etre d'initialisee et _gpg-agent_, _scdaemon_, _dirmngr_ se mangeait un bel _assert()_ et donc ne fonctionnaient pas, mais ca compilait sans soucis, du coup la release etait une release... mais inutilisable.. j'ai fait un patch, mais 2.1.16 a ete release entre temps qui resouds ce BUG mais n'est pas encore forcement dans tout les repository de packages.
* gnupg v2.1 cree des _stubs_ des clefs privee A CHAQUE instanciation de l'agent (--card-status par exemple), si vous avez plusieurs smartcards avec la MEME clef, oubliez pas de tuer l'agent et de "cleaner" (rm -rf $HOME/.gnupg/private-keys-v1.d/\*) sinon il va associer/garder les stubs de la smartcard precedente et au moment de l utilisation vous demander d'inserer la dite smartcard.
* impossible d'avoir le _pinentry_ au moment de mon ssh, il me faut "preparer" l'agent, un petit _gpg2 --card-status_, suivi d'un _gpg2 -d <unfichierchiffreavecmaclefGPG>_ le tout avec ma carte insere (+ le bon PIN) et juste apres je peux faire mon ssh et ca passe.
* _$HOME/.gnupg/scd-events_ peut etre execute a chaque instanciation de l'agent et/ou insertion d'une SC, attention, le lecteur SC standard USB et une yubikey ne se comportent pas de maniere identique, pour avoir un comportement consistent (genre killer l'agent ou cleaner les stubs est pas evident) il faut travailler un peu..
* veillez comme tout les docs le disent a BIEN FAIRE DES BACKUPS apres les differentes etapes, creation masterkey, subkeys, etc.. et a mettre ces backups safe et offline, sinon tout ca ne sert a rien.


## Conclusion

Mon francais sent des pieds mais ca marche, mes clefs ne sont plus "online", a part sur mes 2 SC, la SC donne une interface pour signer/chiffrer (via CCID) mais ne permet pas de recuperer les clefs privees directement (comme sur un FS), je peux plugger ma SC sur different systeme (BSD, Linux, OS/x) mon auth est faisable sans pour autant compremettre aussi facilement mes clefs, je n'ai rien a copier et j'ai un element hardware en plus de mon PIN, pour chiffrer mes datas et pour m'auth sur mes machines, plus feignant que ca tu meurs.

Un seul bemol, il faut un gnupg 2.0.X ou 2.1.16+ et c'est pas encore super "smooth", voila en vous remerciant.
