Création d'un serveur glpi
 

AIDE: IT CONNECT COMPUTASYS 
et Activation d'une connexion en pont sous distribution linux zorin os
Serveur GLPI sur Debian — 

Ce document décrit, étape par étape, l'installation d'un serveur GLPI (dernière version) sur une VM Debian, avec une carte réseau configurée en pont (bridge, virbr0) 
afin que la VM obtienne une adresse IP directement sur le réseau local, dans la plage 192.168.x.x.

Cette procédure s'appuie sur le tutoriel d'IT-Connect : Installation pas-à-pas de GLPI sur Debian.

Étape 1 — Configurer le réseau de la VM en pont (virbr0)
Sur l'hyperviseur (le PC Linux hôte), il faut d'abord configurer l'interface réseau de la VM en mode pont sur virbr0, plutôt qu'en NAT, 
pour que la VM obtienne une IP directement sur le sous-réseau local 192.168.x.x au lieu d'une IP privée en 10.x.x.x. 
Dans virt-manager, cela se fait en éditant le matériel réseau de la VM et en choisissant l'interface virbr0 en tant que "Bridge".
On peut vérifier sur l'hôte que le pont existe bien avec la commande suivante :

ip link show type bridge

Une fois la VM redémarrée avec cette nouvelle interface, on vérifie depuis l'intérieur de la VM Debian que l'adresse obtenue est bien dans la bonne plage :
ip a

Étape 2 — Mettre à jour le système
Avant toute installation, on met à jour la liste des paquets ainsi que les paquets déjà installés sur la Debian :
sudo apt-get update && sudo apt-get upgrade -y

Étape 3 — Installer le socle LAMP
GLPI a besoin d'un serveur web, de PHP et d'une base de données. On installe donc Apache2, PHP-FPM et MariaDB :
sudo apt-get install apache2 php-fpm mariadb-server -y

Ensuite, on installe les extensions PHP indispensables au bon fonctionnement de GLPI (gestion des images, des chaînes multi-octets, du XML, de la connexion à la base de données, etc.) :
sudo apt install php-{curl,gd,intl,mysql,zip,bcmath,mbstring,xml,bz2,ldap} -y

Étape 4 — Préparer la base de données
On commence par sécuriser l'installation de MariaDB (mot de passe root, suppression des comptes anonymes, etc.) :
sudo mariadb-secure-installation

Puis on se connecte à MariaDB en tant que root pour créer une base de données dédiée à GLPI, ainsi qu'un utilisateur qui n'aura de droits que sur cette base :
sudo mysql -u root -p

Une fois connecté, on exécute les requêtes suivantes :
CREATE DATABASE glpi_computasys;
GRANT ALL PRIVILEGES ON glpi_computasys.* TO 'glpi_adm'@'localhost' IDENTIFIED BY 'MotDePasseRobuste';
FLUSH PRIVILEGES;
EXIT;

Étape 5 — Télécharger GLPI
On récupère la dernière version disponible sur le dépôt GitHub officiel du projet GLPI, puis on télécharge l'archive dans /tmp avant de l'extraire dans /var/cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/<VERSION>/glpi-<VERSION>.tgz
sudo tar -xzvf glpi-<VERSION>.tgz -C /var/www/


Étape 6 — Préparer l'installation (droits et répertoires)
On commence par donner la propriété des fichiers de GLPI à l'utilisateur www-data, celui utilisé par Apache2 :
sudo chown www-data /var/www/glpi/ -R

Pour suivre les recommandations de sécurité de l'éditeur, on sort ensuite trois répertoires sensibles de la racine web. D'abord le répertoire de configuration :
sudo mkdir /etc/glpi
sudo chown www-data /etc/glpi/
sudo mv /var/www/glpi/config /etc/glpi
sudo mkdir /var/lib/glpi
sudo chown www-data /var/lib/glpi/
sudo mv /var/www/glpi/files /var/lib/glpi
sudo mkdir /var/log/glpi
sudo chown www-data /var/log/glpi

Il faut ensuite indiquer à GLPI où se trouve désormais le répertoire de configuration, en créant le fichier /var/www/glpi/inc/downstream.php avec ce code
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi/');
if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
    require_once GLPI_CONFIG_DIR . '/local_define.php';
}

Puis on crée le fichier /etc/glpi/local_define.php, qui indique à son tour où se trouvent les répertoires des fichiers et des logs :
<?php
define('GLPI_VAR_DIR', '/var/lib/glpi/files');
define('GLPI_LOG_DIR', '/var/log/glpi');

Étape 7 — Configurer Apache2
On crée un fichier de VirtualHost dédié à GLPI, par exemple /etc/apache2/sites-available/support.computasys.conf :
<VirtualHost *:80>
    ServerName support.computasys

    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        Require all granted

        RewriteEngine On

        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>
</VirtualHost>

Une fois le fichier enregistré, on active ce nouveau site, on désactive le site par défaut (inutile) et on active le module de réécriture d'URL nécessaire à GLPI :
sudo a2ensite support.computasys.conf
sudo a2dissite 000-default.conf
sudo a2enmod rewrite
sudo systemctl restart apache2

Étape 8 — Intégrer PHP-FPM à Apache2
On active les modules Apache nécessaires à PHP-FPM ainsi que sa configuration, puis on recharge Apache2 :
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php-fpm
sudo systemctl reload apache2

On redémarre ensuite le service PHP-FPM pour appliquer ces changements :
sudo systemctl restart php8.4-fpm.service

Enfin, on ajoute au VirtualHost la directive qui indique à Apache2 de transmettre les fichiers .php à PHP-FPM :
<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost/"
</FilesMatch>

Puis on redémarre Apache2 une dernière fois :
sudo systemctl restart apache2

Étape 9 — Terminer l'installation via le navigateur
Depuis un poste du même réseau local, grâce au pont virbr0, on ouvre un navigateur et on se rend sur l'adresse http://<IP-192.168.x.x>/ ou sur http://support.computasys/ si le nom d'hôte a été déclaré. On choisit ensuite la langue, on clique sur Installer, puis on vérifie que tous les prérequis affichés sont bien validés avant de continuer.

À l'étape suivante, on renseigne les informations de connexion à la base de données : le serveur SQL est localhost, l'utilisateur est glpi_adm et le mot de passe est celui défini à l'étape 4. On sélectionne ensuite la base glpi_computasys créée précédemment, puis on termine l'assistant.
Une fois l'installation terminée, GLPI indique les identifiants du compte administrateur par défaut : glpi / glpi.

Étape 10 — Finaliser et sécuriser l'installation
Pour finir, il est indispensable de changer le mot de passe de tous les comptes par défaut proposés par GLPI, puis de supprimer le fichier d'installation qui ne doit plus rester accessible :
sudo rm /var/www/glpi/install/install.php





