# Aperçu de CLIP OS 4

[CLIP OS](https://clip-os.org) est une méta-distribution GNU/Linux conçue avec les hypothèses suivantes :
* tout le monde (développeur, administrateur, utilisateur final) est faillible et peut commettre des actions dangereuses par erreur;
* tout code peut contenir des vulnérabilités, ce qui signifie que nous devons prendre en compte la menace, la minimiser (p. ex., réduire la surface d'attaque), identifier les vulnérabilités résiduelles potentielles et être en mesure de déployer des mises à jour en direct;
* seul le code de confiance doit être exécutable (pas d'exécution de code arbitraire ni de persistance);
* nous considérons qu'il y a plusieurs rôles d'administrateur, selon les mesures organisationnelles ou pour limiter les privilèges disponibles à une personne;
* le système peut avoir plusieurs utilisateurs autorisés;
* the system can be connected to untrusted networks (e.g., in a road warrior environment);
* les événements du journal devraient être disponibles pour détecter les attaques ou les mauvais comportements;
* et de multiples environnements d'utilisateurs peuvent gérer des données de différentes sensibilités : *faible* (p. ex., Internet) et *élevé* (réseau privé).

Pour répondre à ces exigences, CLIP OS fournit de multiples améliorations de sécurité telles que l'isolation des processus, un modèle de permission adapté et de multiples développements de durcissement profondément intégrés dans le système.
Il existe actuellement deux principaux cas d'utilisation du projet CLIP OS, chacun étant fourni par une espèce de CLIP OS dédiée : un système d'utilisateur final ciblant les tâches de bureau communes, et une passerelle VPN.

CLIP OS a débuté en 2005 et est maintenant développé publiquement avec la [version 5](https://github.com/clipos).
Cette version 4 de CLIP OS n'est pas destinée à être utilisée telle quelle.
Certains dossiers peuvent être manquants et d'autres peuvent être incomplets en raison du processus de publication initial.
Cependant, comme pour la version 5, vous êtes libre de choisir les composants ou les correctifs (selon les termes de leur licence) qui peuvent répondre à vos besoins.

## Développement, conditionnement et mise à jour

CLIP OS est basé sur [Gentoo Hardened](https://wiki.gentoo.org/wiki/Hardened_Gentoo), qui vient avec un [hardened toolchain](https://wiki.gentoo.org/wiki/Hardened/Toolchain) et de multiples améliorations de la sécurité.
Ainsi, le gestionnaire de paquets du côté des développeurs est Portage, mais les constructions résultantes pour une distribution CLIP OS sont livrées avec des paquets Debian grâce à un [custom wrapper](https://github.com/clipos-archive/src_platform_clip-deb).
Deux [signatures](https://github.com/clipos-archive/src_platform_gen-crypt), avec un format spécifique au système d'exploitation CLIP, sont ajoutés à chaque paquet Debian : un pour le développeur et un autre pour un contrôleur, permettant ainsi une signature à deux niveaux pour chaque mise à jour.
Le [mécanisme de mise à jour](https://github.com/clipos-archive/src_platform_clip-install-clip) est entièrement automatisé et exécuté en arrière-plan dans un conteneur VServer dédié.
Les [composants du système essentiels](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/clip-conf/clip-core-conf) (p. ex. le noyau, le mécanisme de mise à jour, la gestion des utilisateurs, la configuration du réseau, etc.) sont mises à jour à la suite par le [principe de mise à jour A/B](https://source.android.com/devices/tech/ota/ab/) avec deux jeux de partitions différents mis à jour alternativement.
Cela garantit que le système reste amorçable et récupérable si une mise à jour critique échoue ou est interrompue.
Une mise à jour à la volée peut être demandée pour les [éléments non critiques](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/clip-conf/rm-apps-conf) (par exemple, les applications utilisateur), ne nécessitant donc pas de redémarrage pour les mises à jour mineures, mais pouvant toujours être automatiquement annulées si une erreur se produit.

Plusieurs paquets sont patchés pour répondre aux besoins de CLIP OS.
Ces modifications peuvent être trouvées dans [portage-overlay](https://github.com/clipos-archive/clipos4_portage-overlay) (identifié avec [USE flags](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/profiles/use.desc)), et les développements sur mesure (et importés) sont emballés dans [portage-overlay-clip](https://github.com/clipos-archive/clipos4_portage-overlay-clip).
Les sources correspondantes sont répertoriées dans le Repo [manifest](https://github.com/clipos-archive/clipos4_manifest).

## Cloisonnement et durcissement des services

CLIP OS utilise beaucoup de conteneurs (appelés en interne *jails*), suivant le principe de la défense en profondeur, grâce aux [Linux-VServer](http://www.linux-vserver.org) caractéristiques.
Linux-VServer est un correctif du noyau qui exploite les espaces de noms Linux pour créer un partitionnement sécurisé des processus.
Il permet entre autres de marquer les processus, les ressources et les réseaux avec des identificateurs de contexte (XID) et des identificateurs de réseau (NID).
Il ajoute également un réseau local par prison (sans NIC), des restrictions PTS, de multiples restrictions de visibilité `/proc`, des rôles WATCH (audit) et ADMIN, réduit les canaux cachés et ajoute des restrictions à plusieurs niveaux.
CLIP OS n'utilise pas les outils de l'espace utilisateur VServer en amont mais un [vsctl](https://github.com/clipos-archive/src_platform_vsctl).
Un exemple de configuration de l'environnement jail peut être trouvé pour un [user jail](https://github.com/clipos-archive/src_platform_clip-vserver/blob/master/jails/rm_b) ou un [web server jail](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/www-servers/nginx/files/clip/rb/jail).

De multiples [plans de prison] (https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/clip-layout), pour la plupart en lecture seule, sont utilisés pour les [services système] (https://github.com/clipos-archive/src_platform_core-services/blob/master/jails).
Le contenu de chaque prison est minimal pour réduire les outils disponibles pour les attaquants (par exemple, [BusyBox](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/sys-apps/busybox/busybox-1.25.1-r1.ebuild#L82)).
Les communications entre les environnements jail sont gérées par des IPC sécurisés (par exemple, les sockets UNIX ou SSH sur la [boucle locale](https://github.com/clipos-archive/src_platform_clip-patches/blob/master/2002_loopback_classB.patch)).
Les [règles de pare-feu](https://github.com/clipos-archive/src_platform_clip-generic-net/blob/master/lib/netfilter) contrôlent tous les accès au réseau (par exemple, la boucle locale, Ethernet, Wi-Fi, UMTS, etc.).
). De plus, les services critiques sont renforcés par des patchs personnalisés : [strongSwan](https://github.com/clipos-archive/src_platform_strongswan-patches), [Syslog-NG](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/app-admin/syslog-ng/files/syslog-ng-3.4.7-clip-jail.patch), [DHCP client](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/net-misc/dhcpcd/files/dhcpcd-6.4.7-clip.patch), [Nginx](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/www-servers/nginx/files/nginx-1.7.6-clip-chroot.patch), etc.

Pour aider à créer ces conteneurs et réduire les privilèges, CLIP OS fournit des bibliothèques d'aides à la sécurité : [clip-lib](https://github.com/clipos-archive/src_platform_clip-lib) (gestion sécurisée des privilèges) et [clip-libvserver](https://github.com/clipos-archive/src_platform_clip-libvserver) (gestion des conteneurs).

## CLIP-LSM et correctifs Linux personnalisés

CLIP-LSM](https://github.com/clipos-archive/src_platform_clip-lsm) est un module de sécurité Linux personnalisé qui améliore le modèle de permission Linux (capacités), ajoute des permissions VServer supplémentaires (p. ex., application d'IPsec) et tire parti des fonctions de renforcement de [PaX](https://pax.grsecurity.net) et [grsecurity](https://grsecurity.net).
De nombreux autres [correctifs Linux](https://github.com/clipos-archive/src_platform_clip-patches) corrigent certains problèmes et ajoutent diverses fonctionnalités de sécurité.

Les processus racine et noyau sont limités par un ensemble de limites de capacités supplémentaires, une application limitée des bits setuid de la racine et une exécution stricte du Trusted Path.

[Devctl](https://github.com/clipos-archive/src_platform_clip-lsm/blob/master/security/clsm/devctl.c) est un mécanisme fournissant un contrôle d'accès étendu aux périphériques, par exemple pour verrouiller les options de montage pertinentes pour la sécurité, et appliquer le contrôle d'accès obligatoire aux périphériques et aux montages (lecture, écriture et exécution).

[Veriexec](https://github.com/clipos-archive/src_platform_clip-lsm/blob/master/security/clsm/veriexec.c) est un système simple de gestion des droits (inspiré de NetBSD) qui vérifie également l'intégrité des fichiers.
La [configuration](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/sys-apps/coreutils/coreutils-8.20-r2.ebuild#L193) est indépendante du système de fichiers cible et permet des mises à jour faciles à la volée grâce à l'outil [verictl](https://github.com/clipos-archive/src_platform_verictl).
Veriexec gère les [droits supplémentaires](https://github.com/clipos-archive/src_platform_clip-lsm/blob/master/include/linux/clip_lsm.h) par prison, soit pour ajouter un contrôle d'accès supplémentaire pour les opérations qui sont autorisées par défaut sur un noyau vanille (par exemple, l'accès réseau ou des opérations spécifiques liées à IPsec), soit pour fournir un moyen limité d'autoriser certaines opérations qui nécessitent habituellement des capacités étendues (par exemple, l'accès au journal du noyau).
Il est également chargé d'accorder certaines permissions aux scripts, selon une composition de drapeaux effectifs, hérités et permissifs.

Un des principes de base de CLIP OS est l'application d'une politique "write *xor* execute", tant au niveau de la gestion de la mémoire qu'au niveau des droits d'accès au système de fichiers.
Cela signifie qu'un processus ne doit pas être autorisé à exécuter quelque chose qui n'est pas fourni par le système, évitant ainsi l'exécution arbitraire de code et les attaques persistantes.
L'objectif principal est de protéger le noyau en limitant les appels système arbitraires qu'un attaquant pourrait effectuer avec un binaire élaboré ou certains langages de script.
Il améliore également l'isolation multiniveau en réduisant la capacité d'un attaquant à utiliser des canaux secondaires avec un code spécifique.
Ces restrictions peuvent être appliquées nativement pour les binaires ELF (avec l'option de montage `noexec`) mais nécessitent un correctif du noyau pour gérer correctement les scripts (par exemple, Python, Perl).
Un nouveau drapeau ouvert ([O\_MAYEXEC](https://github.com/clipos-archive/src_platform_clip-patches/blob/master/1901_open_mayexec.patch)) est alors utilisé par les [interprètes modifiés](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/dev-lang/perl/files/perl-5.16.3-clip-mayexec.patch).

Comme l'appel système chroot est utilisé comme un mécanisme d'isolation secondaire, pour isoler davantage certains processus dans un conteneur VServer donné, certaines restrictions supplémentaires sur le chroot sont également appliquées : Protection contre les fuites et accès aux FD, restrictions ptrace, intégration avec grsecurity et VServer, etc.

## Sécurité à plusieurs niveaux

CLIP OS peut être utilisé comme système d'exploitation [multiniveau] (https://en.wikipedia.org/wiki/Multilevel_security), ce qui permet de traiter des données de différentes sensibilités sur un même système et de limiter les fuites de données.
L'utilisateur final peut utiliser deux environnements de bureau : un niveau *faible* (appelé `RM_B`) connecté à Internet et un niveau *élevé* (appelé `RM_H`) pour traiter les données sensibles, uniquement accessible par un VPN (IPsec).
Chaque environnement contient des applications habituelles (par exemple, navigateur web, suite bureautique, logiciel graphique) qui sont confinées à leur niveau assigné (par exemple, réseau, fichiers, interface graphique).

De multiples composantes doivent connaître ce modèle de sécurité, l'appliquer et permettre à l'utilisateur final de gérer plusieurs niveaux (https://www.ssi.gouv.fr/uploads/2018/04/salaun-m_these_manuscrit.pdf) :
* afficher les niveaux à l'utilisateur ([gestionnaire de fenêtre](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/x11-wm/openbox/files/openbox-3.5.0-clip-domains.patch) et [panneau de confiance](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/x11-misc)) ;
* isolation sécurisée et fiable de l'interface graphique ([domaines de l'interface graphique](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/x11-base/xorg-server/files/xorg-server-1.19.3-clip-domains.patch) et [visionneuses VNC](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/net-misc/tigervnc)) ;
* des diodes noires et rouges ([cryptd](https://github.com/clipos-archive/src_platform_cryptd) et [cryptclt](https://github.com/clipos-archive/src_platform_cryptclt)) pour pousser ou décrypter les fichiers du niveau *faible* au niveau *élevé* (suivant le [modèle Bell-LaPadula](https://en.wikipedia.org/wiki/Bell%E2%80%93LaPadula_model)), ou crypter les fichiers du niveau *élevé* au niveau *faible* ;
* niveaux VServer ;
* signature de mémoire de masse externe et cryptage des données ;
* et [proxy à carte à puce] multiniveau (https://github.com/caml-pkcs11/caml-crush).

CLIP OS peut également gérer des périphériques externes par prison : un scanner, une imprimante, une webcam, une carte son et une carte à puce.

## Rôles d'administration et d'audit

Un compte utilisateur CLIP OS peut être configuré avec une composition de rôles privilégiés : administrateur (admin) et auditeur (audit).
Il est important de noter que la délégation de privilèges pour ces rôles n'est pas basée sur l'octroi de privilèges root partiels ou totaux (c'est-à-dire pas de `sudo` ou équivalent), mais sur l'accès en lecture et/ou en écriture à des fichiers de configuration spécifiques dans le conteneur VServer dédié à chaque rôle.
Les fichiers de configuration modifiés sont ensuite analysés de manière sécurisée, et appliqués, s'ils sont valides, par des démons privilégiés en dehors de ces conteneurs.

Le but du rôle d'administrateur est de configurer le système.
Cependant, un tel rôle ne doit pas être en mesure d'altérer le système ni d'accéder aux données des autres utilisateurs.
CLIP OS est conçu pour accorder ces accès à l'administrateur :
* gestion de la visibilité des périphériques ;
* configuration de l'heure et de la date ;
* gestion des utilisateurs ([userd](https://github.com/clipos-archive/src_platform_userd)) ;
* gestion des réseaux et IPsec ([clip-netd](https://github.com/clipos-archive/src_platform_clip-netd)) ;
* configuration de l'affichage ;
* installation et désinstallation de paquets optionnels (p. ex., applications utilisateur) ;
* et configuration de la mise à jour du système ([downloadrequest](https://github.com/clipos-archive/src_platform_downloadrequest)), mais pas les autorités de signature des paquets.

Le rôle d'audit est utilisé pour collecter des informations du système, mais sans possibilité d'accéder à une configuration non liée aux journaux du système.
Les événements des journaux ne sont accessibles qu'à ce rôle, mais en lecture seule.
La gestion des journaux (par exemple, les limites de stockage, les transferts à distance) est exclusivement autorisée au rôle d'audit.

Ces rôles peuvent être utilisés à travers des interfaces graphiques dédiées (par exemple, [clip-config](https://github.com/clipos-archive/src_platform_clip-config)), ou des CLI et des fichiers.
Ils peuvent être disponibles localement par un utilisateur connecté, ou par un VPN dédié via une session SSH.

## Authentification et cryptographie

Nous utilisons un gestionnaire de mots de passe système renforcé ([PAM tcb](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/sys-apps/tcb) d'Openwall) avec l'algorithme de hachage bcrypt.
Sur les périphériques embarqués, les utilisateurs sont automatiquement emprisonnés dans le conteneur VServer approprié lorsqu'ils se connectent en ligne de commande, grâce au module [PAM jail](https://github.com/clipos-archive/src_platform_pam_jail), tandis que sur les environnements de bureau, un emprisonnement similaire est appliqué par l'interface graphique de connexion.
Les données des utilisateurs sont stockées sur des partitions chiffrées dédiées.
Il y a une partition par environnement utilisateur : le système de base, le niveau *faible* et le niveau *élevé*.
Les partitions des utilisateurs sont automatiquement ouvertes, décryptées et montées (dans leur prison dédiée) lorsque l'utilisateur se connecte.
La clé secrète utilisée pour les opérations cryptographiques associées est dérivée d'un mot de passe ou décryptée par une carte à puce.
Enfin, la partition est démontée et fermée lorsque la session utilisateur est fermée.

Une carte à puce peut être utilisée simultanément pour l'authentification de l'utilisateur et dans plusieurs environnements isolés en même temps grâce à [Caml Crush et plusieurs jails dédiés](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/app-crypt/pkcs11-proxy/pkcs11-proxy-1.0.7-r3.ebuild#L86).
La gestion de la carte à puce est assurée par plusieurs paquets : [ckiutl](https://github.com/clipos-archive/src_platform_ckiutl), [smartcard-monitor](https://github.com/clipos-archive/src_platform_smartcard-monitor), un [scdaemon](https://github.com/clipos-archive/src_platform_scdaemon)-like (PGP), etc.

Même si le jeu de partitions du système peut être chiffré et qu'une première étape de développement pour le support de TPM est présente (par exemple, [tpm-cmd](https://github.com/clipos-archive/src_platform_tpm-cmd), [clip-livecd](https://github.com/clipos-archive/src_platform_clip-livecd/blob/master/sbin-scripts/full_install.sh#L119), [clip-kernel](https://github.com/clipos-archive/clipos4_portage-overlay-clip/blob/master/sys-kernel/clip-kernel/files/initrd-clip#L473) et [syslinux-tpm-patches](https://github.com/clipos-archive/src_platform_syslinux-tpm-patches)), CLIP OS 4 n'inclut pas l'altération physique dans son modèle de menace.
C'est l'une des raisons principales pour lesquelles la 5ème version de CLIP OS n'est pas évolutive par rapport à la 4ème.

Plusieurs VPN IPsec peuvent être établis par un client : réseau *haut* niveau, réseau de mise à jour, réseau d'administration et réseau d'audit.
Une [passerelle] CLIP OS (https://github.com/clipos-archive/src_platform_clip-gtw-net) et une PKI dédiée peuvent être mises en place pour gérer les clients CLIP OS et leurs réseaux.

Pour éviter les problèmes d'entropie courants, un [correctif du noyau](https://github.com/clipos-archive/src_platform_clip-patches/blob/master/1702_get_random_bytes_rdrand.patch) et un [démon de l'espace utilisateur](https://github.com/clipos-archive/clipos4_portage-overlay/blob/master/sys-apps/timer_entropyd) alimentent le PRNG.

CLIP OS fournit également des outils utilisateurs tels qu'un outil de gestion de l'ICP simplifié et renforcé : [ANSSI-PKI](https://github.com/clipos-archive/src_platform_anssipki-cli).

# documentation en Français

La principale conférence publique sur CLIP OS 4 a été donnée lors de la [conférence du SSTIC en 2015] (https://www.sstic.org/2015/presentation/clip/).

Ce référentiel contient une sélection des documents initiaux en français destinés à de multiples publics : utilisateurs finaux, administrateurs et développeurs.

## Utilisateur

* [Le changement de mot de passe dans CLIP](utilisateur/changer-mdp.pdf)
* [Écran étendu](utilisateur/configurer-ecran_etendu.pdf)
* [KRDC](utilisateur/configurer-krdc.pdf)
* [Échanger des documents entre les deux niveaux de CLIP](utilisateur/echanger-entre-clip-haut-et-clip-bas.pdf)
* [Guide de démarrage du client CLIP](utilisateur/tour-du-poste-clip.pdf)
* [Clés USB](utilisateur/utiliser-cles-usb.pdf)

## Administrateur

* [Administration en ligne de commande d'une passerelle CLIP](administrateur/administration-ligne-de-commande-passerelle.pdf)
* [Mise en place de l'administration à distance des passerelles CLIP](administrateur/ajouter-acces_ssh.pdf)
* [Configuration DNS d'un poste CLIP](administrateur/configurer-dns.pdf)
* [Pare-feu](administrateur/configurer-firewall.pdf)
* [Fichier resolv.conf](administrateur/configurer-resolvconf.pdf)
* [Effacement manuel d'un poste bureautique CLIP](administrateur/effacer-manuellement-poste-bureautique-clip.pdf)
* [Guide d'utilisation d'une passerelle CLIP](administrateur/guide-utilisation-passerelle.pdf)
* [Installation de CLIP](administrateur/installer-passerelle_clip.pdf)
* [Rôle et gestion des certificats dans CLIP](administrateur/roles-certificats.pdf)

## Développeur

* [Description Générale](developpeur/0001_Description_Generale_1.0.pdf)
* [Description Fonctionnelle](developpeur/1001a_Perimetre_Fonctionnel_CLIP-RM_1.0.4.pdf)
* [Architecture de sécurité](developpeur/1002_Architecture_Securite_1.2.pdf)
* [Paquetages CLIP](developpeur/1003_Paquetages_CLIP_1.1.pdf)
* [Support de l'UEFI](developpeur/1004_Support_de_l_UEFI_1.0.pdf)
* [Génération de paquetages](developpeur/1101_Generation_Paquetages_1.5.2.pdf)
* [Génération d'un support d'installation CLIP](developpeur/1102_Generation_CD_Installation_1.4.1.pdf)
* [Guide d'installation de l'environnement de développement](developpeur/1103_Environnement_de_Developpement_2.3.pdf)
* [CLIP-LSM](developpeur/1201_CLIP_LSM_2.2.pdf)
* [Patch VServer](developpeur/1202_Vserver_1.2.pdf)
* [PaX & grsecurity](developpeur/1203_PaX_Grsecurity_1.1.2.pdf)
* [Privilèges Linux](developpeur/1204_Privileges_Linux_1.0.1.pdf)
* [Générateur d'aléa noyau](developpeur/1206_Generateur_Alea_Noyau_1.0.pdf)
* [Séquences de démarrage et d'arrêt](developpeur/1301_Sequence_Demarrage_1.0.7.pdf)
* [Authentification locale](developpeur/1302_Authentification_CLIP_1.1.2.pdf)
* [X11 et cloisonnement graphique](developpeur/1303_X11_cloisonnement_graphique_1.1.3.pdf)
* [Cages et socle CLIP](developpeur/1304_Cages_CLIP_1.1.0.pdf)
* [Support des cartes à puce sous CLIP](developpeur/1305_Cartes_Puce_3.0.pdf)
* [TPM](developpeur/1307_TPM_0.0.pdf)
* [Cages RM](developpeur/1401_Cages_RM_1.2.pdf)
* [Configuration Réseau](developpeur/1501_Configuration_Reseau_2.6.pdf)
* [Installation CLIP](developpeur/2001_Installation_CLIP_1.1.0.pdf)
* [Guide de l'utilisateur CLIP-RM](developpeur/2101a_Guide_Utilisateur_CLIP-RM_1.4.pdf)
* [Guide de création de paquetage](developpeur/4001_Guide_de_Creation_de_Paquetage_1.3.pdf)

---

Copyright © 2018 [ANSSI](https://www.ssi.gouv.fr/).

CLIP OS est une marque de la République Française.
En conséquence, toute utilisation du nom "CLIP OS" doit être préalablement autorisée par l'ANSSI.
Cela n'empêche pas les modifications des logiciels mis en ligne et leur republication ou citation d'identifier le logiciel original selon les termes de la licence LGPL v2.1+.
Néanmoins, l'utilisation du nom " CLIP OS " sur une version modifiée ne doit pas suggérer que cette version est l'œuvre originale publiée par l'ANSSI.

Le contenu de cette documentation est disponible sous la licence Open License version 2.0 (compatible avec la licence [CC-BY](https://creativecommons.org/licenses/by/2.0/)) telle que publiée par [Etalab](https://www.etalab.gouv.fr/) (groupe de travail français pour l'Open Data).
