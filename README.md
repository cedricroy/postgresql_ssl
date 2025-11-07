# Guide SSL/TLS PostgreSQL


## 1. Objectifs

- Chiffrer les communications entre serveur PostgreSQL et clients (QGIS, pgAdmin, psql).
- Supporter l'authentification par mot de passe et par certificat client (mTLS / clé).
- Assurer confidentialité, intégrité et authenticité des données.

## 2. Génération du certificat et de la clé serveur

Vous avez 2 possibilités, soit vous générez un certificat auto-signé, soit vous passez par une autorité de certification (exemple avec Let's Encrypt).

### 2.1. Auto-signé

#### 2.1.1. Créer le répertoire et s'y rendre :
```bash
sudo mkdir -p /etc/ssl/postgresql
cd /etc/ssl/postgresql
```
#### 2.1.2. Créer le certificat et la clé :
Remplacer ```postgresql.fr``` et ```192.168.1.10``` par vos valeurs
```bash
sudo openssl req -new -x509 -days 3650 -nodes \
   -out server.crt -keyout server.key \
   -subj "/CN=postgresql.fr"
   -addext "subjectAltName = IP:192.168.1.10,DNS:postgresql.fr"
```
#### Explications :  

```openssl req``` : gérer les requêtes de certificats

```\-new``` : nouvelle requête de certificat

```\-x509``` : certificat auto-signé (on peut utiliser certbot à la place pour avoir un certif let's encrypt)

```\-days``` : nombre de jours de validité

```\-nodes``` : NO DES, clé non chiffrée

```\-text``` : infos sur certificat dans le fichier .crt.

```\- out``` : nom du certificat

```\-keyout``` : nom de la clé privée

```\- subj``` : informations du certificate : CN = nom avec lequel les clients se connecte à la base (ip ou dns)

```\-addext``` : IP + dns alors -> "subjectAltName = IP:192.168.1.10,DNS:postgresql.fr"

#### 2.1.3. Donner les bons droits :
```bash 
sudo chmod 600 server.key  
sudo chown postgres:postgres server.\*
```
Si vos collègues utilisent verify-full ou verify-ca dans vos applications, cela signifie qu'ils doivent vérifier l'autorité de certification avec le certificat, il faut donc installer le certificat server.crt (en l'important dans qgis par exemple) sur chaque poste si les postes utilisent verify-full ou verify-ca.

### 2.2. Let's encrypt

#### 2.2.1. Installer certbot 
```bash 
sudo apt install certbot
```
#### 2.2.2. Générer le certificat, la clé et la chaine de confiance

Let’s Encrypt doit pouvoir vérifier le domaine. Donc ajouter votre paire nom de domaine/ adresse IP dans vos DNS. 
```bash 
sudo certbot certonly --standalone -d postgresql.fr
```
Les fichiers se sauvegardent ici 
```/etc/letsencrypt/live/postgresql.fr/```

Explications des fichiers

```privkey.pem``` : clé privée

```cert.pem ``` : certificat

```chain.pem``` : chaîne de confiance

```fullchain.pem ``` : certificat + chaîne complète

#### 2.2.3. Créer le dossier 
```bash
sudo mkdir -p /etc/postgresql/ssl
```
#### 2.2.3. Copier les fichiers créés précedemment 
```bash
sudo cp /etc/letsencrypt/live/db.mondomaine.fr/fullchain.pem /etc/postgresql/ssl/server.crt
sudo cp /etc/letsencrypt/live/db.mondomaine.fr/privkey.pem /etc/postgresql/ssl/server.key
```
#### 2.2.4. Donner les bons droits 
```bash
sudo chown postgres:postgres /etc/postgresql/ssl/server.*
sudo chmod 600 /etc/postgresql/ssl/server.key
```

## 3. Configuration PostgreSQL

### 3.1. Modifier la configuration de postgresql

#### 3.1.1. Ouvrir postgresql.conf 
Remplacer ```17``` par votre version de postgresql.
```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```
puis allez à la section SSL et modifier ces lignes avec :

```bash
ssl = on  
ssl_cert_file = '/etc/ssl/postgresql/server.crt'  
ssl_key_file = '/etc/ssl/postgresql/server.key'
```

#### 3.1.2. Ouvrir pg_hba.conf
Remplacer ```17``` par votre version de postgresql.
```bash
sudo nano /etc/postgresql/17/main/pg_hba.conf
```
Puis modifier les valeurs désirées, vous pouvez forcer ssl avec ```hostssl```.

Exemple pour SSL + mot de passe :
```bash
hostssl all all 10.0.0.0/24 scram-sha-256
```
Exemple pourSSL + certificat client (mTLS) :
```bash
hostssl all all 10.0.0.0/24 cert clientcert=1
```
Puis redémarrer PostgreSQL :
```bash
sudo systemctl restart postgresql
```

## 5. Vérification serveur
Checker dans la base si le ssl est actif.
```bash
SHOW ssl;  
SELECT ssl, version, cipher, bits  
FROM pg_stat_ssl  
WHERE pid = pg_backend_pid();
```

## 6. Connexion côté client
En fonction du type de connexion (certificat ou mot de passe), tester la connexion ou générer le certificat/clé client.

### 6.1. Connexion par mot de passe :
Lancer un terminal afin de tester la connexion. Changer ```postgresql.fr```, ```ma_base```, etc. par vos valeurs.
```bash
psql "host=postgresql.fr dbname=ma_base user=johndoe sslmode=require password=secret"
```

### 6.2. Connexion par certificat client (mTLS) :
Créer le certificat et la clé client
```bash
openssl req -new -nodes -keyout client.key -out client.csr -subj "/CN=johndoe"  
openssl x509 -req -days 365 -in client.csr -CA server.crt -CAkey server.key -set_serial 01 -out client.crt  
```
Tester la connexion
```bash
psql "host=postgresql.fr dbname=ma_base user=johndoe sslmode=verify-full sslcert=client.crt sslkey=client.key sslrootcert=server.crt"
```
## 7. Bonnes pratiques

- Restreindre IP dans pg_hba.conf  

- Penser à renouveler les certificats avant expiration  

- Protéger la clé privée (chmod 600)  

- Vérifier les connexions clients (sslmode=require ou verify-full)
