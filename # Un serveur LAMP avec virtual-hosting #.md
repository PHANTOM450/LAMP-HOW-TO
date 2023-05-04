# Un serveur LAMP avec virtual-hosting #

> *Un petit HOW-TO sur comment mettre en place plusieur hebergement web sur un unique serveur LAMP*

## Installation de unzip :

    apt install unzip -y

## Installation de Curl :

apt install curl

## Installation de WEBMIN :

> *Webmin est un projet github opensource et fort utile comme vous allez le découvrir par vous mêmes, Webmin ne fais pas parti des repos de base de Debian, nous devons donc les ajoutez, avant de l'installer.*

> *Téléchargements et lancement du script d'installation du repository de WEBMIN:*

```
curl -o setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sh setup-repos.sh
```
> *Une fois fais, vous n'avez plus qu'a procédez a l'installation avec aptitude.*

    apt install webmin

## Installation d'apache 2.4

> *apache est le service web le plus commun pour linux vient en deuxième position NGINX, cette outils vas crées des repertoires lu par apache qui presentera les fichiers HTML mis en page sur le port 80 ou 443, il est possible en html de crée un index afin de naviguer de page en page avec des hyperliens, c'est les prémisses d'un site web (STATIQUE).*
> 
    apt install apache2

## activation du module SSL

>* l'activation du SSL vas nous permettre de mettre en place le protocole HTTPS a coter ou en lieu et place de HTTP, nous opterons pour le second choix.*

>* Le mode SSL vas nous permettres de chiffrer les tunnels HTTP avec TLS et non plus SSL comme son nom peut le faire penser, car dépasser depuis des années, cela donne le HTTPS, pour se faire nous l'activons et le déclarons en mode par défault d'apache.*

    a2enmod ssl
    a2ensite default-ssl

## relancer le service ( systemctl restart apache2)

> *Je ne le redirais plus mais vous le remarquerez vite, certaine modification nécéssite le rédemarage du service, afin comme les fichiers de configuration soit de nouveau lu*

    service apache2 restart

## génération du certificat auto signé pour 10 ans

> *pensez à bien mettre votre FQDN en nom d'hote ou COMMON NAME (CN).Dans la même idée veillez à avoir une bonne concordance de nom dans hostname,hosts et resolv.conf.*

> *Faute de quoi des erreurs de sécurité peuvent apparaitre sur une première navigation sur votre site web, cela peut invalider votre certificat comme étant de confiance, car des informations non concordante sur l'identité de l'entité propriétaire et d'administration de celui à été trouvé.*

> *Nous allons maintenant avec cette commande crée un certificat autosigné, apache.pem correspond au nom de votre certificat pensez a vérifiez que le nom correspond a l'utilisateur du certificat pour plus de facilité d'adminsitration, vérifiez le chemin d'accès, si vous souhaitez stocker vos certificat dans un répertoire dédiée, et les identifiers par leur nom vous n'avez qu'as changé le path ainsi que le nom.*

    openssl req $@ -new -x509 -days 3650 -nodes -out /etc/apache2/apache.pem -keyout /etc/apache2/apache.pem

## activation des modules complémentaires

> *Nous allons maintenant activez le module de réecriture d'URL "rewrite" et le module "headers" permettant de modifier l'entete des requette HTTP en HTTPS, ils permets aussi de rediriger des requetes sur différents mirroir, si besoin.*

    a2enmod rewrite
    a2enmod headers

## écriture du nouveau certificat ssl dans la conf apache

    cd /etc/apache2/sites-available/
    nano default-ssl.conf 
> *Ajoutez les lignes SSLCertificateFile et vérifiez que SSLEngine est parametrer en "on".*

    service apache2 restart

## Mises en places d'une redirection de HTTP vers HTTPS

> *Dans le fichier ci-dessous :*

    nano /etc/apache2/sites-available/000-default.conf

> *Vous allez rajoutez ceci :*

    <VirtualHost *:80>
            Redirect Permanent / https://www.infra.lan
                                #PROTOCOLE://HOSTS.DOMAINE.TLD
    </VirtualHost>

> *Ceci vas décretez que toute les requetes HTTP sur le port 80 doivent être renvoyé en HTTPS sur le port 443.*

## Changement du repertoire du site web

> *Nous allons maintenant changez le repertoire du site web permettant à l'utilisateur d'avoir un accès a tout son hebergement personnel stocker dans son repertoire personnel, ainsi nous pourrions crée moult utilisateur, et moult site web, chacun restant cloisonné, les personnes n'aurais plus qu'as chargé en FTP, SMB, ou en SFTP, charger et télécharger leur fichier sur le herbegement web.*

> *Rendez vous dans se fichier :*

    nano /etc/apache2/sites-available/default-ssl.conf

> *Puis remplacer le path de DocumentRoot par celui-voulu, ici nous mettrons.*

    DocumentRoot /home/user/www

>*Ici pour ma part "user" sera remplacer par mon nom d'utilisateur.*

> *Rendez-vous par la suite dans :*

    nano /etc/apache2/apache2.conf

> *Et commenter la section <Directory /var/www/html>*

> *Pour finir nous allons crée un repertoire www et un index.html dans le repertoire /user/, puis nous nous l'approprierons et nous donnerons des droits de modifications dessus, afin que l'utilisateur puisse etre admin à 100% de son site sans pour autant toucher au systèmes.*

    su user
    mkdir /home/user/www
    nano /home/user/www/index.html
    su -
    chown -R www-data:www-data /home/user/www
    usermod -a -G www-data user
    chmod -R 775 /home/user/www

### CONSEIL DE GRAND-MERE :

** Avoir un site web c'est bien mais vous pourriez en avoir plusieur, avec une même IP, comme je vous le montre, grace au VirtualHosts que nous avons commencez a parametrer, pour cela, il faudrait crée dans "/HOME/USER/WWW/" un repertoire par site web, afin de facilité l'administration de votre usine à gaz.**

## Fusion des fichiers de Configuration d'Apache

> *Nous allons fusionner le dossier de configuration par défault pour HTTP ainsi qu'HTTPS (ou mode par default et mode SSL), a question de ne pas avoir une centaine de site heberger en virtualhosting sans automatiser l'adminsitration, cela vous faciliteras grandement la vie, et vous feras gagné beaucoup de temps.*

    cd /etc/apache2/sites-available
    cat 000-default.conf >site.conf
    cat default-ssl.conf >>site.conf

> *Nous désactivons donc les deux anciens fichiers avat leurs suppressions et nous allons déclarer et activer notre fichier fusionnés, puis redemarrer le service Apache.*

    a2dissite 000-default
    a2dissite default-ssl
    rm 000-default.conf
    rm default-ssl.conf
    a2ensite site.conf
    service apache2 restart

> *ET LA SA MARCHE PAS A L'AIDE !!!*
> 
> *Pas de panique rouvrez le fichier sites.conf, et vous allez analyser les sections Directory, se sont des repertoires que vas monter Apache sur les ports que vous lui aurez données.*

> *Après Analyse vous remarquez qu'il ne comporte pas votre repertoire DocumentRoot, rajoutez-le puis veillez à donnez l'accès a tout le monde dans "Require all granted.*

> *Vous devriez donc avoir ceci, à ceci pret que vous devriez adaptater a votre utilisateur, domaine et nom d'hôtes.*

    <VirtualHost *:80>
       Redirect Permanent / https://www.infra.lan
    </VirtualHost>
    <IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerName www.infra.lan
                DocumentRoot /home/user/www

                <Directory /home/user/www/>
                        Options Indexes FollowSymLinks
                        AllowOverride None
                        Require all granted
                </Directory>

                ErrorLog ${APACHE_LOG_DIR}/www.error.log
                CustomLog ${APACHE_LOG_DIR}/www.access.log combined
                SSLEngine on
                SSLCertificateFile      /etc/apache2/apache.pem
        </VirtualHost>
    </IfModule>

> *Nous en avons finis pour le moment avec la configuration d'Apache.*

##Installation et Configuration de MySQL :

> *nous allons nous attelez a l'installation et la configuration d'une base de donnée pour gérer les utilisateurs, celle-ci servira pour les services, que vous souhaiterez installé.*

> *Pour cela nous allons installer un serveur MySQL, au lieu de MariaDB, malheureusement celui-ci ayant été racheter par ORACLE, il n'est pas intégrée dans les repos d'origine de debian, nous devons donc télécharger le binaire debian, le dépackager, puis rafraichir la liste des repos avec update et upgrade.*

>*Nous pourrons donc par la suite l'installer comme n'importe quelle autre paquets debian avec aptitudes, maintenant faite place au commande pour réalisez cela.

    wget https://repo.mysql.com/mysql-apt-config_0.8.23-1_all.deb
    dpkg -i mysql-apt-config_0.8.23-1_all.deb 
    apt update
    apt upgrade
    apt install mysql-server

> *Pour vous connectez à votre services MySQL en tant qu'utilisateur ROOT, et intervenir en cli vous pouvez utilisez cette commande, qui nous permet de valider, la bonne installation du service, pour en sortir faite "exit;"*

    mysql -u root -p

> * pour sécurisez et rendre votre serveur MySQL opérationnel autant en build ou run, nous allons utilisez la fonction d'initialisation sécurisé du serveur MySQL, et parametrer le Mot de Passe ROOT, supprimer les utilisateurs anonymes, et désactiver la connexion root a distance, nous créerons par la suite un utilisateur avec ses droits, nous supprimerons les BDDs de test, et mettrons a jours les privilèges utilisateurs.*

    mysql_secure_installation

> *Vous n'avez qu'a suivre les instructions qui vous guideront, vous saurez jugez des réponses au question, avec le paragraphe précedent.*

##Installation et configuration de PHP

> *PHP est un language permettant d'obtenir du dynamisme d'avoir des widgets et de completer notre serveur web qui jusqu'a maintenant ne possède que HTML, qui ne permet qu'un site web statique.

> *On installe donc php puis on vas configurer PHP, pour augmentez la taille des fichiers chargeable sur notre site à 32Mo, se qui nous permet d'avoir quelque chose de plus fonctionnel, en 2023*

    apt install php
    nano /etc/php/7.4/apache2/php.ini

> *Une fois dans le fichier faite une recherche sur "File Uploads", puis en dessous sur la ligne "upload_max_filesize =" rentré la variable "32M".

> *Par la suite comme php est utilisé par Apache et que nous avons modifier des variable sur le fichier de conf de php, je vous laisse redemarrer le service Apache.*

    service apache2 restart

> *pour vérifiez et obtenir une interface plus visuel pour controler le parametrage de PHP, vous pouvez créez un fichier phpinfo.php, néamoins il énumère des informations qui pourrais permettre une reconnaissances avant exploit de votre site web veillez donc a restraindre son accès ou supprimer le après vérification !*

su user
nano /home/user/www/phinfo.php

> *Puis rajoutez cette ligne dans se fichier :*

    <?php phpinfo() ?>

> *Vous pouvez maintenant ouvrir votre navigateur sur "https://FQDNduSite/phpinfo.php*

> *Comme dit plus haut supprimez le fichier juste après.*

    rm ./phpinfo.info

> * Nous aurons bientot toute les dépendances nécéssaires au divers CMS que vous pourriez installer sur votre serveur web, nous allons donc installez les principales, pensez a lire documents de ceux-i afin de vérifier si il vous en manque, nextcloud et glpi peuvent en reclamez d'autre afin de sécurisé le site, et l'optimisez.*

    apt install php-curl php-gd php-zip php-apcu php-xml php-ldap php-mbstring
    apt install php-mysql
    service apache2 restart

## Installation de PHPMyAdmin :

> * Vous l'aurez vus précedemment manager une base de donnée SQL en cli c'est sympa ca flatte l'égo, mais c'est lourd a la longue, quand on est pas un expert, nous avons maintenant installé les dépendances nécéssaires à l'installation d'un services web nous permettant de manager nos BDD MySQL via une interfaces web, vous allez voir c'est relativement rapide.*

    apt install phpmyadmin

> *C'est rapide et déjà opérationnel nous avions fais le nécéssaire juste avant, vous pouvez dès maintenant tapez dans votre navigateur internet "https://fqdn-de-l'host.tld/phpmyadmin.*

> BON CA FONCTIONNE C'EST TROP COOL MAIS C'EST LONG A TAPEZ ET C'EST CHIANT SI IL Y A BEAUCOUP DE SOUS DOMAINE !!!

> *Bon bas nous allons voir commment mettre en place un alias pour pouvoir taper une abréviation du repertoire "phpmyadmin" soit "pma" soit "my" "mysql", personnelement pma me fais penser a autre chose, donc pour rester avec un url court je choisis "my".

> *Vous allez vous dirigez dans se fichier pour l'éditez :

    /etc/phpmyadmin/apache.conf

> *Vous commenterez la ligne Alias par défault ou non si vous souhaitez en avoir plusieur, puis rajoutez le ou les Alias de votre choix comme ceci.*

    # Alias /phpmyadmin /usr/share/phpmyadmin
    Alias /my /usr/share/phpmyadmin

> *Veillez a donnez l'accès a tout le monde dans le Directory /usr/share/phpmyadmin*

> *Puis vérifiez qu'Apache puisse lire la configuration de PHPMyAdmin.

    # echo "Include /etc/phpmyadmin/apache.conf" >> /etc/apache2/apache2.conf

> *Enfin redemarrer le service Apache2 pour qu'il relise le fichier de configuration de phpmyadmin, désormais vous devriez pouvoir accedez à phpmyadmin par votre alias.*

    systemctl restart apache2

## Création et Don de privilèges Utilisateur sur MySQL :

> *Allez sur phpmyadmin, dans la BDD "user", puis editez le compte root, en changeant l'HOST (localhost) par "%".

> *Puis nous allons créer un utilisateur dans phpmyadmin et changer le "host" en "%" et cocher la case "Créer une base portant son nom...".*

> *Puis ouvrer une fenetre de commande SQL et taper :*

    mysql> flush privileges;

## Installation et Configuration du service FTP :

> *Ca en ai maintenant finis de MySQL et PHP pour le moment nous allons mettre en place un serveur FTPS, puis SAMBA, afin de pouvoir Chargé dans les repertoires "/HOME/USER/WWW/" de cette manière vous pourrez developpez vos pages web sur votre laptop puis une fois faites les chargés en build ou run sur vos serveur lamp.*

> *Pour commencez on installe le service.*

    apt install vsftpd

> *Nous allons sauvegardé le fichier de configuration originel dans un fichier de backup (*.bak), puis nous supprimerons le fichier de conf pour en recréer un simplifier et avec des parametres spécifiques.*

    cp /etc/vsftpd.conf /etc/vsftpd.bak
    rm /etc/vsftpd.conf
    nano /etc/vsftpd.conf

> *Dans se fichier rentré ceci puis ajuster les variables en fonctions de votre environnement.*
```
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
#
# Run standalone?  vsftpd can run either from an inetd or as a standalone
# daemon started from an initscript.
listen=NO
#
# This directive enables listening on IPv6 sockets. By default, listening
# on the IPv6 "any" address (::) will accept connections from both IPv6
# and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
# sockets. If you want that (perhaps because you want to listen on specific
# addresses) then you must run two copies of vsftpd with two configuration
# files.
listen_ipv6=YES
#
# Allow anonymous FTP? (Disabled by default).
#anonymous_enable=YES
#anon_root=/var/ftp/pub
#
# Uncomment this to allow local users to log in.
local_enable=YES
#
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
#anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# If enabled, vsftpd will display directory listings with the time
# in  your  local  time  zone.  The default is to display GMT. The
# times returned by the MDTM FTP command are also affected by this
# option.
use_localtime=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/vsftpd.log
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
#xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
ftpd_banner=If you doesn't had the Entitie's Authorization, you encurse an penal sanction for accèss and maintened's connection on an unauthorized system.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd.banned_emails
#
# You may restrict local users to their home directories.  See the FAQ for
# the possible risks in this before using chroot_local_user or
# chroot_list_enable below.
chroot_local_user=YES
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
allow_writeable_chroot=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd.chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# Customization
#
# Some of vsftpd's settings don't fit the filesystem layout by
# default.
#
# This option should be the name of a directory which is empty.  Also, the
# directory should not be writable by the ftp user. This directory is used
# as a secure chroot() jail at times vsftpd does not require filesystem
# access.
secure_chroot_dir=/var/run/vsftpd/empty
#
# This string is the name of the PAM service vsftpd will use.
pam_service_name=vsftpd
#
# This option specifies the location of the RSA certificate to use for SSL
# encrypted connections.
rsa_cert_file=/etc/apache2/apache.pem
rsa_private_key_file=/etc/apache2/apache.pem
ssl_enable=YES
pasv_enable=yes
pasv_min_port=65000
pasv_max_port=65500
```
> *Enregistrez votre fichier de configuration, quittez l'éditeur, et redemarrer le serviceur de serveur FTP, comme ceci :*

    service vsftpd restart
    service vsftpd status

> *Vérifier le status du service de serveur FTP, puis nous allons  initier, une connexion, avec le nom d'hote du serveur, et un utilisateur ayant droit de connection au serveur ftp, veillez a choisir le protocole "FTPES" afin d'avoir une connexion explicite avec tunnel TLS chiffré.

> *Vous avez maintenant accès a vos repertoire via FTP.*

## Installation et Configuration du service SAMBA

> *comme la cli mysql, le protocole FTP, vous obliges a lancé une session quand vous souhaitez avoir accès a votre repertoire, et vous oblige a installer un client sur votre client.*

> *Il serais plus pratique de monter le service Serveur SAMBA sur votre serveur lamp, afin de monter un lecteur réseau sur les repertoires qui vous sont utiles, de cette maniere vous pouvez avec une GPO a l'ouverture de session, monter le lecteur réseaux, et avoir constamment accès a votre repertoire, via l'authentification au partage réseau, via un compte user sur mysql synchronisez avec l'A.D. plus besoin de s'authentifier au partage.*

> *Nous allons déjà mettre en palce de le service SAMBA.*

```
apt install samba
mv /etc/samba/smb.conf /etc/samba/smb.bak
nano /etc/samba/smb.conf
```
> *Une fois dans le fichier de configuration nous allons parametrés, le workgroup windows, ici je met le nom de mon workgroup windows.
> *Remplacer le "netbios name:" par votre nom d'hote (serveur), ici www car c'est un serveur web, rajouter une description a votre partage, et remplacer le nom de votre interface réseaux au besoin.

```
[global]
workgroup = LAN
netbios name = www
comment = Partage WWW.*.LAN /HOME/USER/WWW
interfaces = ens33
encrypt passwords = true
security = user
smb passwd file = /usr/bin/smbpasswd
passdb backend = tdbsam
obey pam restrictions = yes

[www]
path = /home/user/www
valid users = user
#a remplacer par votre nom d'user
writeable = Yes
create mask = 777
directory mask = 777
```

> *Puis relancer les 2 services :*

```
service nmdb restart
service smbd restart
testparm
```

>*Cette derniere commande vas tester votre fichier de configuration, cela vous permet de savoir si votre service est actif et opérationnel.*

> *Chiffré les mot de passes c'est bien, donc voici la petite commande pour mettre un mot de passe au user déclarer dans le fichier précédent, et le chiffrer :*

    smbpasswd -a user
    #Changer l'utilisateur au besoin

>*Si oui vous pouvez maintenant monter et vous authentifier sur le partage SMB depuis votre laptop.*

*\\votreip\www ou \\votrenomnetbios\www ou \\votrefqdn\www*

> *Votre Serveur est bientot finis ne vous inquietez pas, si jamais des modifications une mises a jours ou une attaques est faites vous seriez content d'avoir des backups, nous allons donc crée un script de sauvegarde dans un repertoire précis une sauvegarde du repertoire "www" et des bases de données mysql.*

> *On commence par aller dans le repertoire "root"

    cd /root

> *Nous allons ensuite crée le fichier de script en "*.sh", nous allons changé les droits dessus, puis nous allons l'éditez avec ses commandes.*

```
#!/bin/bash
#Script de backup automatique de la BDD et du repertoire "/HOME/USER/WWW" pour le site de l'utilisateur.
#V1.1 par T.Cherrier sep 2022

clear
echo Compression :
 zip -rq /home/user/$(date +%Y%m%d%H%M)_www_backup-fichier.zip  /home/user/www
echo Dump de la base
mysqldump -u root -p'dadfba16' --databases user > /home/user/$(date +%Y%m%d%H%M)_www_backup-base.sql
#Sur cette ligne remplacer "--databases user" par l'utilisateur que vous avez crée dans phpmyadmin. 
echo Terminé
```

> *Enregistrez et fermez le fichier, puis nous allons tester le script.*

```
cd /root
./backup
```

> *Vous aurez maintenant dans "/HOME/USER/WWW/" les fichiers sauvegardé par le script, il sera vite commode d'organiser un repertoires bien ordonnés, avant que les backups ne s'entassent de trop.*

> *Nous allons maintenant faire une tache de lancement du script automatiser avec Cron.*

> *Tapez :*

    crontab -e

> * Puis vous allez rajouter cette ligne, "*/30 * * * * est une variable, que vous pouvez adapter grace a se site.

> http://crontab.guru/

    */30 * * * * /root/backup.sh

## Mises en place du Certificat sur WEBMIN :

> *WEBMIN possede son propre certificat autosigné, d'origine qui ne comprend pas vos information, vus que vous en avez fais un pou votre nom de domaine tout à l'heure, vous pouvez désormais, en mettre un personnalisé sur WEBMIN.*

> *Rendez vous dans :*
> "https://fqdn.tld:10000"
> *Puis dans l'onglet Webmin > Webmin Confguration*
> *Onglet SSL Settings, activez le SSL si ce n'est pas le cas, puis dans "Private Key Files" faite parcourir et selectionnez votre certificat "*.pem".

Dernières Ligne droite avant de finir votre serveur lamp, et d'installer vos CMS ou CMDB ou autre service web fortement util ou non.

## Avoir un pare-feu c'est bien !!! :

> * bizzarement trop peu de fois reviennent une remarque telle-que "bizzare on a pas eu a ouvrir de port pour nos protocoles réseau ?!?"
> Bas Oui ! nous n'avons toujours pas installer et configuré de pare-feu mais nous allons corrigé cela tout de suite.*

N'ayant pas envie de m'embeter avec de long fichier de configuration et etant amoureux du service "UFW" c'est celui que je vous propose d'installé.

```
apt update && upgrade
apt install ufw

```
> * Maintenant nous devons avant de l'activer et faire sauter votre connection SSH et vos services réseaux, déclarer les ports que nous voulons ouvrir.*

ufw allow 80
ufw allow 443
ufw allow 10000
ufw allow 65000:65500
ufw allow 20
ufw allow 21
ufw allow 22
ufw allow 137
ufw allow 139
ufw allow 445
ufw allow 990
ufw allow 3306

> *Une courte liste des règles que vous voudriez ouvrirs pour les services que nous mettons en places, une fois ses règles ajouter faites :*

    ufw enable

> *Le firewall est maintenant actif, et bloque le reste des ports en connection entrante si non déclaré.*
> *Vous pouvez vérifier le status des règles avec :

    ufw status

## Installation de Wordpress :

> * Nous allons commencer par nous déplacer dans votre repertoire utilisateur puis www, et nous créerons un repertoire pour votre site wordpress, au nom de votre site par exemple, puis nous irons dedans.*

```
cd /home/user/www
```

> *Puis on télécharge et on dézippe wordpress, une fois cela fait, nous supprimerons les binaire de base.*

```
wget https://fr.wordpress.org/latest-fr_FR.zip
unzip ./latest-fr_FR.zip
rm ./latest-fr_FR.zip
```

> *Wordpress est prêt vous n'avez qu'as finalisez l'installation depuis l'interface web, si vous avez compris le fonctionnement d'un serveur lamp, vous avez dus crée un virtualhost, avec document-root et path menant a wordpress, dans se cas taper, "https://virtualfqdn".

>*Votre fichier Sites.conf devrais donc ressemblez a cela :*

> *Vous aurez maintenant pus comprendre le fonctionnement de tout cela, et vous serez a même de comprendre comment installer un nextcloud, un etherpad et autre services sympathique de vous meme, avec un seul serveur, une seule ip publique/privée, et des noms d'hote virtuel, menant a des repertoires différent.*

### DISCLAIMER SECURITE ###
Ceci est un bref apercus de comment mettre en place un serveur lamb dans un homelab, il est de votre devoir de le sécurisé plus que cela, et cela ne peut etre fait qu'au cas par cas, je ne saurais donc être retenu responsable de toute misconfiguration, ou oublie de parametrage causant une faille e séurité, et un exploit de votre site web.

Je n'ai pas le temps de mettre a jour se HOW-TO, ni d'y passer plus de temps, il est de votre devoir de vérifier et de mettre à jours les paquets, et d'installer les dernieres version "release".

Le petit parametrage d'UFW ne suffit pas, c'est simplement pour vous donner un apercu, de se à quoi ressemble un serveur lamp pour un administrateur systèmes sans sécurité poussé.