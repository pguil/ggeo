---
author: ["P Guil"]
title: "ODK"
date: "2025-05-04"
draft: "false"
description: "Introduction à ODK"
summary: "Introduction à ODK"
categories: ["sig"]
tags: [ "odk", "linux", "qgis" ]
ShowToc: true
TocOpen: true
---
## Introduction à ODK
![logo ODK](/img/2025/odk-logo.png)

Open Data Kit (ODK) permet de créer des formulaires pour collecter de la donnée depuis le terrain.

Ce kit est composé: 
* d'un standard XLSForm pour réaliser des formulaires avec un tableur comme LibreOffice, Excel ou Google Spreadsheet,
* de l'application mobile ODK Collect pour collecter les données sur le terrain à partir du formulaire créé,
* et d'un serveur pour agréger les soumissions de formulaires comme ODK Central ou KoboToolBox. 

Les avantages sont la facilité de mise en œuvre et d'adaptation à une problématique. Il n'y a pas de base de données à installer et tout le paramétrage du formulaire se fait à l'aide d'un tableur. Le serveur ne sert qu'à agréger et télécharger les données. Rien ne reste sur le serveur une fois l'enquête terminée. 

Autre avantage: la saisie sur le terrain en mode déconnecté lorsque la couverture réseau est incertaine. De plus, comme il n'y a pas de grosse base de données, le serveur n'a pas besoin d'être un monstre. Les données collectées avec une composante spatiale (réponses à la question "où?") peuvent servir à alimenter un Système d'Information Géographique (SIG).

Pour toute information complémentaire, vous pouvez consulter le site [getodk](https://getodk.org/), la [documentation](https://docs.getodk.org/) ou le dépot de code [github](https://github.com/getodk)

## Auto-héberger le serveur ODK Central

Vous pouvez choisir d'auto-héberger le serveur ODK Central pour avoir la main sur tout le workflow. Pour ceux qui ne veulent pas s'embêter, vous pouvez utiliser un serveur hébergé par un fournisseur de service, payant ou pas, mais là n'est pas l'objet de mon post. Enfin, à la fin du post je détaille l'utilisation de KoboToolBox quand même.

### Installer ODK Central sur un serveur

Ce n'est pas bien compliqué un fois qu'on a son domaine et son serveur VPS. J'ai suivi ce [tuto](https://docs.getodk.org/central-install/#installing-elsewhere) et notamment cette [section](https://docs.getodk.org/central-install-digital-ocean/) pour une installation avec Docker sur un VPS OVH.

### Louer un VPS

J'ai pris un VPS de base avec 2Go de RAM chez OVH. Les option de snapshot et de backup automatique font grimper la facture. A voir car les données soumises sont téléchargées au fur et à mesure. Le snapshot reste intéressant pour une sauvegarde de l'installation à un instant T. Il est possible de s'en sortir pour 10 à 12€ TTC /mois avec snapshot et backup automatique, et pour 5€ sans les sauvegardes mais c'est sans filet. 

J'ai choisi un serveur avec OS (debian 12) et Docker préinstallé mais vous pouvez choisir un serveur Debian nu et faire votre installation de Docker à la mano. Un fois fait, vous recevrez un lien par mail pour définir un mot de passe pour le compte root. La connexion au serveur se fera par SSH. Pour un minimum de sécurité, pensez à changer le port par défaut du SSH, voire mettre en place un fail2ban. Cette [doc OVH](https://help.ovhcloud.com/csm/fr-vps-security-tips?id=kb_article_view&sysparm_article=KB0047708) peut vous aider. Pas besoin d'être parano car le serveur debian est une "coquille vide", tout se passe dans les containers Docker.  

Vous aurez accès à tous les paramètres dans le manager OVH dans la section Bare Metal Cloud correspondant aux VPS.
Pour l'exemple, l'IP du serveur sera `137.24.255.255`.  

### Configurer son domaine

#### Prendre un domaine chez un Registrar

Vous avez besoin de prendre un nom de domaine chez un Registrar. Il y en a pour tous les gouts faites vous plaisir. Attention le coût peut varier en fonction de l'extension de premier niveau et de la promo la première année. Un .fr coute environ 7-8€ /an. Cela va permettre de faire pointer un sous domaine vers votre serveur VPS. Dans l'exemple le domaine sera `toto.fr`.

#### Configurer un sous-domaine

Pourquoi configurer un sous-domaine et pas le domaine entier? Parce que votre domaine `toto.fr` peut servir à plein d'autres choses. Il suffit de créer un sous domaine `odkcentral`et  de l'affecter à votre VPS sur lequel sera installé ODK Central. 

Pour cela, dans votre manager OVH, allez dans l'onglet Web Cloud, puis sur votre domaine et dans la section Zone DNS.  

Cliquez sur le bouton "Ajouter un entrée", choisissez un type de champ DNS "A", remplissez votre sous domaine comme `odkcentral`, laissez la TTL par défaut, et définissez la cible vers l'adresse IP de votre serveur VPS `137.24.255.255` et validez.

Lorsque quelqu'un remplira l'url `https://odkcentral.toto.fr` dans son navigateur, il sera redirigé sur le VPS hébergeant ODK Central.

### Installer ODK Central

#### Installation avec Docker
Avec un terminal, connecté en SSH au VPS avec un compte ayant des droits root lancer les commandes indiquées.  
Adapté d'après ce [tuto](https://docs.getodk.org/central-install-digital-ocean/).

S'assurer d'avoir les bonnes versions de Docker Engine > v23x et de Docker Compose > v2.16.x
```bash
$ docker --version && docker compose version
```

Télécharger le logiciel à partir de github
```bash
$ git clone https://github.com/getodk/central
```

Aller dans le nouveau dossier central
```bash
$ cd central
```

Obtenir la dernière version du client et du serveur avec git
```bash
$ git submodule update -i
```

Mettre à jour les paramètres en copiant le modèle de fichier des paramètres pour les éditer
```bash
$ cp .env.template .env
```

Editer le fichier `.env` avec l'éditeur de texte nano
```bash
$ nano .env
```

Naviger avec les flèches directionnelles à l'intérieur de nano  
Changer la ligne `DOMAIN=MyOdkCentralServer.com` pour `DOMAIN=odkcentral.toto.fr`  
Changer la ligne `SYSADMIN_EMAIL=administrator@email.com` pour `SYSADMIN_EMAIL=votre@email.com`. Cette adresse mail est utilisée par le service de certicats let's encrypt
Enregistrer avec les touches `CTRL` + `x` et `Entrée`

On veut la denière version de la base de donnée postgres (comme quoi il y a quand même une base de données)
```bash
$ touch ./files/allow-postgres14-upgrade
```

Lancer la construction des containers avec Docker compose
```bash
$ docker compose build
```

Attendre. Surveillez le "Successfully built". Super, ODK Central est installé. 

Démarrer le serveur Central
```bash
$ docker compose up -d
```

Regardez si ODK a fini de charger
```bash
$ docker compose ps
```

Sous la colonne `status` attendre que `central-nginx-1` soit `Up` ou `Up (healthy)`

Vérifier que le site répond en allant sur `https://odkcentral.toto.fr`


#### Créez un compte administrateur de ODK Central

Aller dans le dossier central sur le VPS

```bash
$ cd central
``` 

Créer un nouveau compte avec une adresse email

```bash
$ docker compose exec service odk-cmd --email YOUREMAIL@ADDRESSHERE.com user-create
``` 

Rentrer un mot de passe

Rendre ce compte administrateur

```bash
$ docker compose exec service odk-cmd --email YOUREMAIL@ADDRESSHERE.com user-promote
```

Si vous perdez votre mot de passe réinitialisez le

```bash
$ docker compose exec service odk-cmd --email YOUREMAIL@ADDRESSHERE.com user-set-password
```

Retournez sur le site `https://odkcentral.toto.fr` et connectez vous avec le compte administrateur créé.


## Modéliser son formulaire XLSForm
Le modèle fourni par ODK Central lors de la création du premier formulaire comprends 3 feuilles qu'il faut remplir.
1. Survey: le formulaire à proprement parler
2. Choices: les listes de choix et leurs valeurs servant au formulaire
3. Settings: les paramètres généraux comme le nom du formulaire et les champs servant à identifier chaque soumission (ex: concaténation d'un champ date et d'un champ nom d'observateur).

Les autres feuilles indicatives contiennent des informations commes la bibliothèque des différents types de données possibles.

Consultez [la doc](https://docs.getodk.org/) ou [le forum](https://forum.getodk.org/) d'ODK Central pour cela.

Une fois modélisé, uploadez le fichier Excel pour créer une ébauche de formulaire et testez-le. 

Une fois testé, vous pouvez le publier et définir les droits d'accès au formulaire:
* ouvert à tous,
* aux seuls détenteurs du lien, 
* aux utilisateurs définis, 
* aux collecteurs à partir d'application mobile)

## Installer le client mobile Android ODK Collect
L'application mobile ODK Collect permet de saisir les formulaires créés. Elle est disponibles sur les stores 

Android: [ODK Collect](https://play.google.com/store/apps/details?id=org.odk.collect.android&pcampaignid=web_share)

Apple: pas de version Apple d'ODK Collect mais [d'autres applications compatibles](https://forum.getodk.org/t/ios-version-of-odk-collect/10151) non testées.

Le paramétrage du formulaire se fait automatiquement avec le QR Code généré dans le serveur ODK Central et fournit par l'administrateur au collecteur des données. De même, les droits sont définis par l'administrateur dans ODK Central. On peut aussi saisir l'url du serveur, l'identifiant et le mot de passe.

Pour plus de détails sur l'utilisation d'ODK Collect vous pouvez consulter ce [guide](https://ariegenature.fr/wp-content/uploads/2022/07/Guide-premiere-utilisation-ODK.pdf) du CEN de l'Ariège qui est très bien fait.

## Utiliser le formulaire web Enketo
ODK Central permet de saisir directement le formulaire web avec Enketo.

## Télécharger les soumissions de formulaires avec ODK Central
Dans ODK Central, vous pouvez valider les soumissions, les agréger et les télécharger au format CSV. Le traitement des informations géolocalisées peut poser problème. En effet les champs `geopoint`, `geotrace` et `geoshape` contiennent les coordonnées des points qui composent les points, lignes et polygones mais ne sont pas dans un format facilement utilisable avec les SIG pour une visualisation cartographique.

## Résoudre les problèmes d'envoi de mails
ODK Central envoi des mails lors de la création des comptes utilisateurs web pour la saisie avec Enketo. Des mails sont aussi générés lorsque les utilisateurs réinitialisent leurs mots de passe. L'envoi de mails par le serveur ODK Central peut être difficile s'il est configuré sur un sous-domaine mal authentifié (politiques DKIM et DMARC). Dans mon exemple les mails proviennent de no-reply@odkcentral.toto.fr ce qui devient compliqué si @toto.fr est déjà paramétré comme serveur mail avec une politique stricte en matière de délivrabilité mail. Il faut donc penser à bien authentifier aussi le sous domaine pour l'envoi mail. 

Mais le plus simple est d'utiliser un serveur SMTP alternatif dédié.

Dans le terminal aller dans le dossier `central` et éditer le ficher `.env` avec le serveur hôte, le port et les détails d'authentification.

```bash
$ cd central
$ nano .env
```

Décommenter et remplir les lignes suivantes:

```bash
EMAIL_FROM=my-no-reply-email-address
EMAIL_HOST=my-email-host
EMAIL_PORT=my-email-port
EMAIL_IGNORE_TLS=true-or-false
EMAIL_SECURE=true-or-false
EMAIL_USER=my-email-user
EMAIL_PASSWORD=my-email-password
```

* EMAIL_FROM l'adresse mail de l'expéditeur (sender address) pour les réponses.
* EMAIL_HOST le serveur de mail hôte pour le SMTP.
* EMAIL_IGNORE_TLS réglé sur `false`.
* EMAIL_SECURE réglé sur `true` si vous utilisez le port 465 et sur `false` pour les autres ports.
* EMAIL_USER et EMAIL_PASSWORD sont tous deux requis.

Lancer le build et redémarrer le container:

```bash
$ docker compose build service && docker compose stop service && docker compose up -d service
```


## Utiliser les données dans un SIG avec QGis
Pour visualiser et télécharger les soumissions des formulaires vous pouvez aussi utiliser le logiciel libre QGis.

Des plugins ou extensions permettent de télécharger depuis ODK Central et de traiter les données géolocalisées (geopoint, geotrace, geoshape) pour les afficher dans QGis. J'en ai testé 2.

### ODKTraceToWKT
[ODKTraceToWKT](https://plugins.qgis.org/plugins/odktrace2wkt/)

Après téléchargement des soumissions ODK Central en CSV, permet de convertir des geotraces en tableur CSV avec colonne géographique en WKT pour visualiser sur QGis.

### QuODK
[QuODK](https://forum.getodk.org/t/quodk-another-connector-between-odk-central-and-qgis-to-get-your-data-more-easily/51526/6)

Permet la connexion directe au serveur ODK Central. Paramétrer le plugin en spécifiant l'url du serveur ODK Central (ne pas mettre le /final à la fin de l'url du serveur ODK), l'identifiant de l'utilisateur et le mot de passe. Ensuite spécifier le projet, le formulaire et le type de geolocalisation à afficher.

### QRealTime

Permet la synchronisation directe d'une couche SIG QGis au serveur ODK Central ou à KoboToolBox. Mais je n'ai pas réussi à me connecter à mon instance.

## KoboToolBox
La plateforme commerciale [KoboToolbox](https://www.kobotoolbox.org/) regroupe un agrégateur compatible ODK et les outils de gestion, visualisation, de traitement et d'export des soumissions. Les mêmes formulaires Excel peuvent être utilisés comme modèles. La version gratuite permet de traiter jusqu'à 5000 soumissions par mois. Ce peut être une alternative à ODK Central pour ceux qui ne veulent pas faire d'auto-hébergement.

## Ressources, tutoriels, vidéos

Vidéos:
* Cette [vidéo](https://youtu.be/8rS2fM7LO8s?si=7dilLfGLGY_VMioz) montre bien l'installation d'ODK Central et la configuration du sous-domaine sur un serveur Linux.  
* [Celle-ci](https://youtu.be/22l0xHxJ3vo?si=YeU9z7S9RTqhttsV) vous guide pour réaliser votre premier formulaire.  
* [Celle-là](https://youtu.be/4gs50sF_qJc?si=8BlAcHBclT_4rm5C) vous explique comment utiliser KoboToolBox pour télécharger les données.  

Articles:
* [Geotribu](https://geotribu.fr/articles/2021/2021-06-08_odk_postgis_1/)

Forum:
* [getODK](https://forum.getodk.org/)

Docs:
* [getODK](https://docs.getodk.org/)


