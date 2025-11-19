# Résolution d'erreur : Docker sur WSL2 Windows

## 🔴 Problème rencontré

### Erreur
```
Cannot connect to the Docker daemon at unix:///mnt/wsl/docker-desktop/shared-sockets/docker.sock. 
Is the docker daemon running?
```

### Contexte
- Système : Windows avec WSL2 (Ubuntu)
- Docker Desktop installé et configuré
- Utilisateur : `ines`
- Le socket Docker existe bien dans `/var/run/docker.sock`

## 🔍 Diagnostic

### Vérifications effectuées

1. **Socket Docker présent**
   ```bash
   ls -la /var/run/docker.sock
   # Résultat : srw-rw---- 1 root docker 0 Nov 19 16:56 /var/run/docker.sock
   ```

2. **Groupes de l'utilisateur**
   ```bash
   groups ines
   # Avant : ines sudo users
   # Après : ines sudo users docker
   ```

3. **Variable d'environnement Docker**
   ```bash
   echo $DOCKER_HOST
   # Vide, mais Docker cherchait dans /mnt/wsl/docker-desktop/shared-sockets/
   ```

## ✅ Solutions appliquées

### Étape 1 : Ajouter l'utilisateur au groupe docker

```bash
# En tant que root
usermod -aG docker ines
```

### Étape 2 : Redémarrer WSL pour appliquer les changements

```powershell
# Dans PowerShell Windows
wsl --shutdown
```

Puis rouvrir le terminal WSL.

### Étape 3 : Définir la variable DOCKER_HOST

```bash
# En tant que ines
export DOCKER_HOST=unix:///var/run/docker.sock
```

### Étape 4 : Rendre le changement permanent

```bash
# Ajouter au fichier .bashrc
echo 'export DOCKER_HOST=unix:///var/run/docker.sock' >> ~/.bashrc

# Recharger la configuration
source ~/.bashrc
```

## ✅ Vérification finale

```bash
docker version
```

**Résultat attendu :**
```
Client:
 Version:           26.1.4
 API version:       1.45
 ...

Server: Docker Desktop
 Engine:
  Version:          26.1.4
  API version:      1.45
  ...
```

Les sections **Client** ET **Server** doivent apparaître.

## 📝 Résumé

### Causes du problème
1. **Permissions insuffisantes** : L'utilisateur n'était pas dans le groupe `docker`
2. **Mauvais chemin du socket** : Docker cherchait le socket dans `/mnt/wsl/docker-desktop/shared-sockets/` au lieu de `/var/run/docker.sock`

### Solutions
1. Ajouter l'utilisateur au groupe `docker`
2. Redémarrer WSL pour appliquer les changements de groupe
3. Définir `DOCKER_HOST=unix:///var/run/docker.sock`
4. Rendre cette configuration permanente dans `.bashrc`

## 🚀 Lancement de Jenkins

Une fois Docker fonctionnel :

```bash
# Avec une image personnalisée
docker run --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home jenkins-kube-helm:lts

# Ou avec l'image officielle
docker run --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

## 💡 Notes importantes

- ⚠️ Ne pas utiliser `chmod 777` sur `/var/run/docker.sock` avec Docker Desktop
- ⚠️ Le redémarrage de WSL (`wsl --shutdown`) est **obligatoire** après l'ajout au groupe docker
- ⚠️ Un simple `exit` et `su -` ne suffit pas pour recharger les groupes

## 🔧 Configuration Docker Desktop

Assurez-vous dans **Docker Desktop** → **Settings** → **Resources** → **WSL Integration** :
- ✅ "Enable integration with my default WSL distro" est coché
- ✅ Votre distribution Ubuntu est activée

---

**Date de résolution :** 19 Novembre 2025  
**Système :** Windows + WSL2 (Ubuntu) + Docker Desktop