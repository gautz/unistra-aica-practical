## TP de manipulation avancée à l'IUT Haguenau - déploiement Cloud de AICA

On déploie les TP avec une architecture client-serveur.

Avantages :

- Fonctionne dans le navigateur sur postes Windows même peu puissants (WebGL)
- Pas besoin d'installer Docker, WSL, VirtualBox ou d'avoir des droits admin
- Déploiement d'autant d'environnements AICA qu'on veut sur une VM en un script

### Prérequis
#### Côté serveur

- Une VM (machine virtuelle ou serveur physique) Linux
    - Avec un accès SSH pour un utilisateur dans le groupe `sudo`
    - Configuration minimum (pour les tests) : 30G de SSD, 2G de RAM, 2 vCPU
    - Configuration recommandée : 2G de RAM et 1 vCPU par étudiant/conteneur, 60G de SSD
    - Avec `docker.io` installé
    - De préférence avec GPU NVidia, les drivers propriétaires et `nvidia-container-toolkit` installés
- Pour la configuration SSH, sudo, firewall : installation de yunohost.org recommandée
- Serveur public avec 3 ports accessibles par étudiant OU serveur accessible dans le réseau local des PC de TP

#### Côté client

- Un PC avec une connexion rapide à la VM
    - Pour l'utilisation de AICA Studio : Chrome/Chromium installé avec support de WebGL
    - Pour donner accès à la configuration de AICA : VSCode/Codium installé avec l'extension `ms-vscode-remote.remote-ssh` (sur Codium `jeanp413.open-remote-ssh`, l'arborescence de fichier n'est accessible qu'en lecture-seule)

<p class="callout info">Si notre client est un PC Linux on peut avoir une expérience AICA complète en affichant RViz et AICA Launcher sur le PC : [https://docs.aica.tech/docs/reference/manual-installation-launch#display-sharing](https://docs.aica.tech/docs/reference/manual-installation-launch#display-sharing)</p>

### Installation de AICA sur le serveur via Terminal, Git et SSH

[https://docs.aica.tech/docs/reference/manual-installation-launch](https://docs.aica.tech/docs/reference/manual-installation-launch)

- Ouvrir un Terminal
- Se connecter à la VM via SSH
- Se placer dans votre espace utilisateur `~` / `/home/USERNAME/`
- Cloner le dépôt des TPs AICA `git clone https://github.com/gautz/unistra-aica-practical aica`
- Se placer dans le dossier `cd ~/aica`
- Envoyer la licence au serveur via SSH : `scp ./aica-license.toml USERNAME@server.tld:aica/aica-license.toml`
- Connecter la VM au dépôt Docker de AICA
    `cat /home/USERNAME/aica/aica-license.toml | sudo docker login registry.licensing.aica.tech -u USERNAME --password-stdin`
- Build le ou les environnements Docker nécessaires pour vos TPs depuis le fichier `aica-launcher-....toml` en donnant un nom différent à chaque environnement nécessaire, par ex. `aica-vision-web` :  
    ```bash
    cd ~/aica
    sudo docker build -f /home/USERNAME/aica/aica-launcher-vision-web.toml -t aica-vision-web .
    ```

### Installation de AICA sur le serveur via VSCode et Remote SSH

[https://docs.aica.tech/docs/reference/manual-installation-launch](https://docs.aica.tech/docs/reference/manual-installation-launch)

- Ouvrir VSCode avec l'extension `ms-vscode-remote.remote-ssh` (Codium ne permet pas l'envoi graphique de fichiers)
- Se connecter à la VM via Remote SSH
- File > Open Folder > `/home/USERNAME/`
- Copier le dossier `aica` (téléchargeable sur https://github.com/gautz/unistra-aica-practical) dans votre espace utilisateur `~` / `/home/USERNAME/`
- Copier la licence `aica-license.toml` dans `~/aica/`
- Ouvrir un Terminal sur le serveur
- Connecter la VM au dépôt Docker de AICA
    `cat /home/USERNAME/aica/aica-license.toml | sudo docker login registry.licensing.aica.tech -u USERNAME --password-stdin`
- Build le ou les environnements Docker nécessaires pour vos TPs depuis le fichier `aica-launcher-....toml` en donnant un nom différent à chaque environnement nécessaire, par ex. `aica-vision-web` :  
    ```bash
    cd ~/aica
    sudo docker build -f /home/USERNAME/aica/aica-launcher-vision-web.toml -t aica-vision-web .
    ```
### Exemple de ficher launcher pour un TP de vision avec Yolo

Pour déploiement sur un PC avec GPU NVidia, les drivers propriétaires et `nvidia-container-toolkit` installés :

- Voir `4_aica_ur_yolo_gpu/aica-launcher-vision-gpu.toml`

Pour déploiement sur un serveur sans GPU :

- Voir `2_aica_yolo_web/aica-launcher-vision-web.toml`

### Démarrer, tester et initialiser AICA Studio

- Démarrer le conteneur depuis l'utilisateur administrateur en montant le dossier `/persistent` dans `/persistent` 

```bash
sudo docker run -it --rm --privileged --net=host -v /home/USERNAME/aica/aica-license.toml:/license:ro -v /home/USERNAME/aica/persistent/:/persistent:rw -e AICA_SUPER_ADMIN_PASSWORD=12345678 --name aica aica-vision-web
```

- L'argument `-e AICA_SUPER_ADMIN_PASSWORD=12345678` n'est nécessaire qu'au premier démarrage pour pouvoir créer un compte Admin.
- ouvrir AICA Studio dans Chrome `server.tld:8080`
- créer des comptes avec différentes permissions depuis l'interface super-admin
- **si votre serveur est public, mettre une phrase-de-passe** forte car AICA Studio sera accessible tant que le conteneur est actif
- si nécessaire, pré-enregistrer des Applications AICA 
- noter que dans un conteneur donné, tous les utilisateurs ont accès à toutes les Applications AICA. Pour isoler les utilisateurs, il faut qu'ils travaillent dans un conteneur avec un dossier persistant dédié
- les comptes et applications sont sauvegardées dans une base de données SQL dans `/data` dans `aica.sqlite`, `aica.sqlite-shm` et `aica.sqlite-wal`
- pour garder et dupliquer la config, on copie les fichiers `.sqlite` dans le dossier `persistent/`
- ouvrir un Terminal dans le conteneur : `sudo docker container exec -it -u ros2 aica /bin/bash`
- Copier la base SQL dans le dossier persistant : `cp -r /data /persistent`
- Stoper le conteneur : `docker container ps` puis `docker container stop <container_name>`.
- Démarrer le conteneur depuis l'utilisateur administrateur en montant le dossier `/persistent` dans `/data`

```bash
sudo docker run -it --rm --privileged --net=host -v /home/USERNAME/aica/aica-license.toml:/license:ro -v /home/USERNAME/aica/persistent/:/data:rw --name aica aica-vision-web
```

- Ouvrir AICA Studio et vérifier que les comptes et applications sont bien présents
- Ou attacher un Terminal à l'intérieur l'instance `aica` de `aica-vision-web` :

```bash
sudo docker container exec -it -u ros2 aica /bin/bash
ros2 node list
```

- On voit bien que le contenu du dossier de la VM `aica-vision-web/persistent/` est dispo dans le conteneur dans le dossier `/data` :

```bash
ls -l /data
aica.sqlite aica.sqlite-shm aica.sqlite-wal
```

- Détacher le Terminal du conteneur avec `CTRL+D` ou en tapant `exit`.

### TP3 - Vision par IA avec Yolo - étudiants sans accès SSH

Si les étudiants n'ont pas accès à la VM, ils ne pourront pas réaliser le Setup de Yolo et du composant générateur de Twist https://docs.aica.tech/core/examples/guides/yolo-example#setup . Il faut donc le réaliser en avance sur la VM.

#### Précompiler les composants locaux

- Si ils ne sont pas déjà dispo `sudo docker images`, télécharger et compiler les composants dépendants https://docs.aica.tech/core/examples/guides/yolo-example#creating-a-custom-twist-generator-component , par ex. :

```bash
cd ~/aica/package-template
sudo docker build -f aica-package.toml -t object-detection-utils .
```

- Inclure les composants locaux dans votre `aica-launcher-....toml`, par ex. :

```
# other extensions
"object-detection-utils" = "docker-image://object-detection-utils" # self-built component to feed goal frame extracted from yolo to robot velocity controller
# launcher syntax: object-detection-utils
```

#### Ajouter les modèles et vidéos de test Yolo

Ce dépôt fournit déjà les éléments nécessaires dans `~/yolo-example-data`. Pour les reproduire, suivez le tutoriel : https://docs.aica.tech/core/examples/guides/yolo-example#setup

#### Démarrer un conteneur par étudiant avec tmuxinator

On utilise tmuxinator pour initialiser les conteneurs et les gérer pendant un TP :
- Se connecter au serveur avec SSH
- `sudo apt install tmuxinator`
- Lancer tmuxinator pour créer les utilisateurs Linux, ajouter leur dossier persistant :

```bash
cd ~/aica
tmuxinator start -t /tmuxinator/aica-init.yml
```

- Une fenêtre s'ouvre avec autant de terminaux que de conteneurs nécessaires
- Dans chaque Terminal qui s'ouvre, définir la phrase-de-passe de l'utilisateur Linux, puis la license pour un conteneur donné
- choisir des phrases-de-passe fortes et/ou retirer les accès SSH et ne pas ajouter au groupe sudo

Pour l'étudiant `1` du groupe de TP `1`, on démarre une instance du conteneur `aica-vision-web` nommée `robot1` en lui passant le dossier persistant `/home/robot1/aica/persistent-tp1/` qui sera disponible dans le dossier `/data` du conteneur :

```bash
sudo docker run -it --rm --privileged --net=robot1 -p public_ports:8080-8081 -v /home/robot1/aica/aica-license-robot1.toml:/license:ro -v /home/robot1/aica/persistent-tp1/:/data:rw -v /home/robot1/aica/yolo-example-data/:/yolo-example-data:rw --name robot1 aica-vision-web
```

Il peut y avoir des Warnings qui sont à ignorer et apparaissent si Cloud Storage n'est pas configuré ou si la vérification de license prend plus que quelques secondes :

```
[2024-11-18 13:38:16 +0000] [135] [INFO] Starting sync of cloud applications
[2024-11-18 13:38:16 +0000] [135] [WARNING] Sync failed
[2024-11-18 13:08:42 +0000] [151] [INFO] Waiting for licensing status... 5
[WARN] [1731935323.407252919] [EventEngine.ServiceHandler]: (404): Could not determine any license status
```

### TP3 - Vision par IA avec Yolo - étudiants admin du serveur

