# ROGER-sky
projet 19, réseaux

## Création VM
   Dans le paramètres network ajouter un réseaux hôte privé
    
   Dans hostnetworkmanager créer un réseau mettre 255.255.255.252 en netmask
    
    — Taille de 8Go (7.41GB se rapproche le plus de 8Go)
    — Partitionner
        |
        | ____ Une parition de 4.2Gb primaire mount en '/'
         | ____ Une parition swap de 1Gb logique en swap
          | ____ Une partition avec le reste logique en '/home'
          
## Installer des outils
   Mise a jour des packages
   
    apt install -y vim sudo net-tools iptables-persistent fail2ban sendmail apache2
    apt install -y git (partie déploiement)
    
    
## Interface STATIC
   Mise en place du réseau, modifier le fichier
        
    vim /etc/network/interfaces
        
   enp0s8
                   
     iface enp0s8 inet static
     address 192.168.56.2
     netmask 255.255.255.252
                  
   Mettre a jour
       
    reboot
        
## Service *SSH*
### Serveur
   Modifier le fichier /etc/ssh/sshd_config

     port 2222
     PermitRootLogin no
     PubkeyAuthentication yes
        
        
   Mettre a jour
 
    service ssh restart
          
### Client
   Générer une clé publique avec
          
    ssh-keygen
          
  Rendre la clé publique éffective
          
    ssh-copy-id -i id_rsa.pub -p 2222 user@192.168.56.2
       
## Firewall
   Mise des règles de pare-feu
   
   Listes les règles existantes
   
    sudo iptables -L
   
  Ajouter ce fichier et écrire les règles de votre choix
  
    sudo vim /etc/network/if-pre-up.d/iptables
    
  *Exemple de règle*
  	
	#!/bin/bash

	iptables-restore < /etc/iptables.test.rules

	iptables -F
	iptables -X
	iptables -t nat -F
	iptables -t nat -X
	iptables -t mangle -F
	iptables -t mangle -X

	iptables -P INPUT DROP

	iptables -P OUTPUT DROP

	iptables -P FORWARD DROP

	iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

	iptables -A INPUT -p tcp -i enp0s8 --dport 2222 -j ACCEPT

	iptables -A INPUT -p tcp -i enp0s8 --dport 80 -j ACCEPT

	iptables -A INPUT -p tcp -i enp0s8 --dport 443 -j ACCEPT

	iptables -A OUTPUT -m conntrack ! --ctstate INVALID -j ACCEPT

	iptables -I INPUT -i lo -j ACCEPT

	iptables -A INPUT -j LOG

	iptables -A FORWARD -j LOG

	iptables -I INPUT -p tcp --dport 80 -m connlimit --connlimit-above 10 --connlimit-mask 20 -j DROP

	exit 0

  Puis le rendre executables
  
    sudo chmod +x /etc/network/if-pre-up.d/iptables
                  
## Denial Of Service Attack *DOS*
   Création du fichier log pour le serveur apache, fait office de jorunal de bord
   
    sudo touch /var/log/apache2/server.log
   
   Créer un fichier de configuration local permet de ne pas compromettre les règles par defaut
   
    sudo vim /etc/fail2ban/jail.local
    
   Créer un fichier de filtrages
    
    sudo vim /etc/fail2ban/filter.d/http-get-dos.conf
    
   Insérer ceci dans le fichier créer
   
   	[Definition]

	# Option: failregex
	# Note: This regex will match any GET entry in your logs, so basically all valid and not valid entries are a match.
	# You should set up in the jail.conf file, the maxretry and findtime carefully in order to avoid false positives.

	failregex = ^<HOST> -.*"(GET|POST).*

	# Option: ignoreregex
	# Notes.: regex to ignore. If this regex matches, the line is ignored.
	# Values: TEXT
	#
	ignoreregex =
   
   Relancer le tout
   
    sudo systemctl restart fail2ban.service
    
  Vous pouvez télécharger ceci pour tester que votre serveur tourne toujours
  
    https://github.com/gkbrk/slowloris
   
## Vérification des ports ouverts
   La commande
    
    sudo netstat -paunt
    
## Désactiver les services inutiles
   La commande
    
    service --status-all
    systemctl disable <SERVICE>
    
## Script *Update* && *Watching*
   Créer un script pour updater les packages automatiquement, le fichier doit se finir par ".sh"
   
   Ajouter ceci dans /etc/hosts, sinon les mails ne s'enveront pas
   
   	127.0.0.1	localhost localhost.localdomain
	127.0.1.1	RO test.home.com

	# The following lines are desirable for IPv6 capable hosts
	::1     localhost ip6-localhost ip6-loopback
	ff02::1 ip6-allnodes
	ff02::2 ip6-allrouters
    
   Contenu du script *Update*
    
    #! /bin/bash
        apt-get update && apt-get upgrade
    
   Rendez-le executable
   
    chmod +x FICHIER.sh
    
   Ouvrer /etc/crontab avec un éditeur et ajouter ceci
   
    0 4	* * 1	root	/home/USER/FICHIER.sh  >> /var/log/FICHIER.log
    @reboot		root	/home/USER/FICHIER.sh  >> /var/log/FICHIER.log
    
   Copier le contenu de votre crontab
   
    cp /etc/crontab /home/USER/tmp
   
   Créer le contenu du mail a envoyer
   
    vim /home/USER/email.txt
   
   Contenu du script *Watching*
   
    #!/bin/bash
    cat /etc/crontab > /home/USER/new
    DIFF=$(diff new tmp)
    if [ "$DIFF" != "" ]; then
	    sudo sendmail ROOT@MAIL.com < /home/USER/email.txt
	    rm -rf /home/USER/tmp
	    cp /home/USER/new /home/USER/tmp
    fi
    
   Rendez-le executable
   
    chmod +x FICHIER.sh
    
   Ouvrer /etc/crontab avec un éditeur et ajouter ceci
   
    0  0	* * *	root	/home/USER/watch_script.sh
    
## OpenSSl certificates
   Première partie
   Générer une clé *rsa* pour votre certificat auto-sginé ssl
	
   	openssl genrsa 1024 > COMMETUVEUX.key
	
   Pour la commande suivante des informations seront demandés
	
	openssl req -new -key ./COMMETUVEUX.key > COMMETUVEUX.csr

   Derniere commande
   
   	openssl x509 -in COMMETUVEUX.csr -out COMMETUVEUX.crt -req -signkey COMMETUVEUX.key -days 365
		
   Ou alors si t'a la flemme
   
   	sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -subj "/C=FR/ST=IDF/O=42/OU=Project-roger/CN=10.11.200.247" -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

   Tu peux aussi créer ce fichier etc/apache2/conf-available/ssl-params.conf et mettre ça dedans
   
   	SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
	SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
	SSLHonorCipherOrder On

	Header always set X-Frame-Options DENY
	Header always set X-Content-Type-Options nosniff

	SSLCompression off
	SSLUseStapling on
	SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

	SSLSessionTickets Off

   Dans /etc/apache2/sites-available modifier ceci
   
   	DocumentRoot /var/www/html
	
	SSLOptions +FakeBasicAuth +ExportCertData +StrictRequire
   		
   	SSLCertificateFile	/etc/ssl/certs/COMMETUVEUX.crt
	SSLCertificateKeyFile /etc/ssl/private/COMMETUVEUX.key

   Copier /etc/apache2/sites-available/000-default.conf
   Modifier votre copie /etc/apache2/sites-available/001-default.conf
   
   	ServerName www.COMMETUVEUX.com
	ServerAdmin TONBLAZE@mail.com
	DocumentRoot /var/www/TONDIR/
	
	Redirect "/" "https://192.168.56.2/"
	
   Apres cela il faut encore entrée ces commandes, afin d'activer le module ssl, désactiver l'ancienne config et lancer la nouvelle
   
   	sudo a2enmod ssl
	sudo apachectl configtest
	sudo a2dissite 000-default.conf
	sudo a2ensite 001-default.conf
	sudo a2ensite default-ssl
	sudo systemctl restart apache2.service
   
## Deploiement GIT
   Cette méthode permet de pouvoir update dans beaucoup de situation, autant en local, qu'en réseaux.
   
   Créer un dossier */var/repo/git-repo.git* et entrez la commande
   	
	git init --bare
	
   Créez /var/repo/git-repo.git/hooks/post-recieve
   
   	#!/bin/sh
	git --work-tree=/var/www/TONCHEMIN --git-dir=/var/repo/git-repo.git checkout -f
	
   Ensuite rendez le executable
   
   	chmod +x post-recieve
   
   Pour l'exemple dans votre home créer un dossier /home/USER/web, entrez
	
	git init
	git remote add live /var/repo/git-repo.git
	
   Pour cette dernière commande vous pouvez ajouter ssh://USER@192.168.56.2/var/repo/git-repo.git pour vous connecter en local
   
   Ensuite vous pouvez update tranquille
   
   	git add TONUPDATE
	git commit -m "TONUPDATE"
	git push live master
