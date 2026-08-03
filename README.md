# 🐳 Conteneurisation avec Docker d'un serveur de galerie photo Python

Ce projet présente la conteneurisation avec **Docker** d'un serveur de galerie photo écrit en Python.

L'objectif n'est pas de recréer le serveur photo dans ce dépôt, mais d'expliquer simplement comment une application Python existante peut être adaptée puis exécutée dans un conteneur Docker.

Le projet original du serveur photo est disponible ici :

**➡️ [[LIEN VERS LE DÉPÔT GITHUB DU SERVEUR PHOTO]](https://github.com/philippe86220/Phototheque-Animaliere)**

Le serveur utilise notamment **Pillow** pour la création des miniatures des photographies.

---

## 📁 Structure du projet Docker

Le projet peut être organisé de la manière suivante :

```text
galerie-docker/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── galerie.py
```

Les quatre fichiers ont des rôles différents :

* `Dockerfile` : décrit la construction de l'image Docker.
* `docker-compose.yml` : configure et lance le conteneur.
* `requirements.txt` : indique les dépendances Python nécessaires.
* `galerie.py` : contient le serveur photo adapté pour fonctionner dans le conteneur.

---

# 1. Le fichier `Dockerfile`

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY galerie.py .

EXPOSE 9000

CMD ["python", "galerie.py"]
```

Voyons chaque instruction.

### `FROM python:3.13-slim`

```dockerfile
FROM python:3.13-slim
```

Cette instruction définit l'image de départ utilisée pour construire notre propre image.

Nous utilisons ici une image contenant **Python 3.13** dans sa variante `slim`.

Cette variante permet de disposer d'un environnement Python relativement léger sans installer dans l'image de nombreux éléments dont notre application n'a pas besoin.

---

### `WORKDIR /app`

```dockerfile
WORKDIR /app
```

Cette instruction définit `/app` comme répertoire de travail à l'intérieur du conteneur.

Les instructions suivantes travailleront donc à partir de ce répertoire.

---

### Copier `requirements.txt`

```dockerfile
COPY requirements.txt .
```

Cette instruction copie le fichier `requirements.txt` depuis le projet vers le répertoire `/app` de l'image.

Le point `.` représente ici le répertoire de travail courant, donc `/app`.

---

### Installer les dépendances Python

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

`pip` est le gestionnaire de paquets utilisé par Python.

L'option :

```text
--no-cache-dir
```

demande à `pip` de ne pas conserver son cache de téléchargement après l'installation.

Cela évite de conserver inutilement ces fichiers dans l'image Docker.

L'option :

```text
-r requirements.txt
```

demande à `pip` de lire le fichier `requirements.txt` et d'installer les paquets qui y sont indiqués.

---

### Copier le programme Python

```dockerfile
COPY galerie.py .
```

Cette instruction copie `galerie.py` dans `/app`.

Nous obtenons donc notamment :

```text
/app/galerie.py
```

à l'intérieur de l'image.

---

### `EXPOSE 9000`

```dockerfile
EXPOSE 9000
```

Cette instruction indique que l'application utilise le port `9000`.

Elle documente le port utilisé par le serveur.

La publication effective de ce port vers la machine hôte sera réalisée dans le fichier `docker-compose.yml`.

---

### Lancer le serveur

```dockerfile
CMD ["python", "galerie.py"]
```

Cette commande est exécutée lorsque le conteneur démarre.

Elle correspond à :

```bash
python galerie.py
```

Le serveur photo démarre donc automatiquement avec le conteneur.

---

# 2. Le fichier `requirements.txt`

Le serveur photo utilise Pillow pour le traitement des images et la création des miniatures.

Le fichier contient :

```text
Pillow==12.3.0
```

Le symbole :

```text
==
```

permet d'imposer précisément la version utilisée.

Cela rend la construction de l'image plus reproductible : une reconstruction ultérieure demandera toujours cette même version de Pillow plutôt que la dernière version disponible au moment de la reconstruction.

---

# 3. Le fichier `docker-compose.yml`

```yaml
services:
  galerie:
    build: .
    ports:
      - "9000:9000"
    volumes:
      - /home/philippe86220/Images/Photos:/photos
    environment:
      - DOSSIER_PHOTOS=/photos
    restart: unless-stopped
```

Ce fichier permet de définir la configuration nécessaire au lancement du conteneur.

---

## `build: .`

```yaml
build: .
```

Docker Compose doit construire l'image à partir du `Dockerfile` présent dans le répertoire courant.

---

## Publication du port

```yaml
ports:
  - "9000:9000"
```

Cette ligne relie :

```text
port 9000 de la machine Linux
            ↓
port 9000 du conteneur
```

Le serveur devient ainsi accessible depuis l'extérieur du conteneur.

---

# 4. Donner accès aux photos au conteneur

Les photographies ne sont pas copiées dans l'image Docker.

Elles restent stockées sur le disque de la machine Linux.

Le montage est réalisé avec :

```yaml
volumes:
  - /home/philippe86220/Images/Photos:/photos
```

Cette ligne établit la correspondance suivante :

```text
Machine Linux
/home/philippe86220/Images/Photos

              ↓

Conteneur Docker
/photos
```

Ainsi, lorsque le programme accède à :

```text
/photos
```

depuis le conteneur, il accède en réalité aux photographies présentes dans :

```text
/home/philippe86220/Images/Photos
```

sur la machine hôte.

Les milliers de photographies n'ont donc pas besoin d'être intégrées à l'image Docker.

---

# 5. Transmission du chemin à Python

Le fichier `docker-compose.yml` contient également :

```yaml
environment:
  - DOSSIER_PHOTOS=/photos
```

Cette instruction crée une variable d'environnement appelée :

```text
DOSSIER_PHOTOS
```

avec la valeur :

```text
/photos
```

Cette information peut alors être récupérée directement par Python.

---

# 6. Modification de `galerie.py`

Dans la version initiale du serveur, le chemin vers les photographies pouvait être directement défini dans le programme.

Pour rendre le programme compatible avec Docker et plus facilement portable, le début de `galerie.py` utilise maintenant :

```python
DOSSIER_PHOTOS = os.environ.get("DOSSIER_PHOTOS", "/photos")
PORT = int(os.environ.get("PORT", 9000))
```

Regardons la première ligne :

```python
os.environ.get("DOSSIER_PHOTOS", "/photos")
```

Python cherche la variable d'environnement :

```text
DOSSIER_PHOTOS
```

Si elle existe, il utilise sa valeur.

Dans notre conteneur, Docker Compose fournit :

```text
DOSSIER_PHOTOS=/photos
```

Python utilise donc :

```text
/photos
```

Si cette variable n'existe pas, Python utilisera également `/photos` comme valeur par défaut.

Le même principe est appliqué au port :

```python
PORT = int(os.environ.get("PORT", 9000))
```

La valeur par défaut est :

```text
9000
```

`os.environ.get()` renvoyant une chaîne de caractères, `int()` transforme cette valeur en nombre entier utilisable pour le numéro du port.

---

# 7. Création des miniatures

Le programme définit également :

```python
DOSSIER_MINIATURES = os.path.join(DOSSIER_PHOTOS, "miniatures")
```

Les miniatures sont donc enregistrées dans :

```text
/photos/miniatures
```

depuis le point de vue du conteneur.

Mais `/photos` étant monté sur le véritable dossier des photographies, elles se retrouvent réellement dans :

```text
/home/philippe86220/Images/Photos/miniatures
```

sur la machine Linux.

Pour cette raison, le volume est monté avec un accès en écriture : le programme doit pouvoir créer le dossier des miniatures et y enregistrer les images générées.

---

# 8. Démarrer le projet

Une fois les fichiers en place, la construction de l'image et le démarrage du conteneur peuvent être réalisés avec :

```bash
docker compose up -d --build
```

`--build` demande de construire ou reconstruire l'image si nécessaire.

`-d` signifie *detached* : le conteneur fonctionne en arrière-plan et le terminal reste disponible.

---

# 9. Arrêter le projet

Pour arrêter et supprimer les conteneurs créés par Compose :

```bash
docker compose down
```

Les photographies ne sont évidemment pas supprimées puisqu'elles restent stockées sur le disque de la machine Linux.

---

# 10. Principe général

Ce projet permet finalement de séparer trois éléments :

```text
APPLICATION
galerie.py
     │
     ▼
ENVIRONNEMENT
Python 3.13 + Pillow
     │
     ▼
DONNÉES
/photos
     │
     ▼
/home/philippe86220/Images/Photos
```

Docker fournit l'environnement d'exécution.

Python fournit l'application.

Le volume donne accès aux données réelles présentes sur la machine.

Le serveur photo devient ainsi plus facilement **portable, reproductible et déployable**, tout en conservant les photographies indépendamment du conteneur.

## Remerciements

La rédaction de ce README en français a été réalisée par **ChatGPT (OpenAI)** à partir de mon projet. J'ai relu, validé et adapté son contenu avant sa publication sur GitHub.

