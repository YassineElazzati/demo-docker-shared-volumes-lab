# demo-docker-shared-volumes-lab

## TP 2.1 : Architecture Sidecar - Partage de Volumes

Cet exercice démontre comment deux conteneurs peuvent communiquer de manière découplée via un volume partagé. Nous allons simuler un cas d'usage courant, inspiré par des architectures comme celle de Netflix, où une application principale produit des logs et un conteneur "sidecar" les collecte pour un traitement ultérieur.

Le **producteur** (notre application) écrira des logs dans un volume, et le **consommateur** (notre `log-shipper`) lira ces logs depuis le même volume, sans qu'ils aient besoin de se connaître directement.

---

### Étape 1 : Créer le Volume Partagé

Le volume est le "pont" qui va relier nos deux services. Il s'agit d'un espace de stockage géré par Docker, indépendant du cycle de vie des conteneurs.

```bash
docker volume create shared-logs
```

---

### Étape 2 : Lancer le Producteur de Logs (Conteneur Applicatif)

Nous lançons un conteneur `app-server` qui simule une application écrivant des logs d'accès toutes les deux secondes.

-   `-d` : Lance le conteneur en mode détaché (en arrière-plan).
-   `-v shared-logs:/var/log/app` : Monte notre volume `shared-logs` dans le répertoire `/var/log/app` du conteneur.

```bash
docker run -d --name app-server -v shared-logs:/var/log/app alpine sh -c "while true; do echo \"$(date) : User logged in\" >> /var/log/app/access.log; sleep 2; done"
```

---

### Étape 3 : Lancer le Consommateur (Log-Shipper) en Mode Lecture Seule

Maintenant, nous déployons notre sidecar `log-shipper`. Sa seule responsabilité est de lire les logs.

-   `-v shared-logs:/logs:ro` : Monte le même volume, mais avec le flag `:ro` (Read-Only). C'est une mesure de sécurité cruciale : le `log-shipper` ne doit en aucun cas pouvoir modifier les logs produits par l'application.

```bash
docker run -d --name log-shipper -v shared-logs:/logs:ro alpine tail -f /logs/access.log
```

---

### Étape 4 : Vérifier la Synchronisation des Logs

Observons en direct les logs capturés par notre `log-shipper`. Vous devriez voir les messages de "User logged in" apparaître au fur et à mesure que l'application les écrit.

```bash
docker logs -f log-shipper
```

---

### Étape 5 : Tester la Sécurité (Read-Only)

Prouvons que notre `log-shipper` ne peut pas altérer les fichiers de log. La commande suivante tentera de supprimer le fichier de log et échouera, car le volume est monté en lecture seule.

```bash
docker exec log-shipper rm /logs/access.log
```

La sortie attendue est une erreur de type `rm: can't remove '/logs/access.log': Read-only file system`.

---

### Nettoyage des Ressources

Cette étape est cruciale pour éviter de laisser des conteneurs ou des volumes inutilisés sur votre système.

1.  **Arrêter et supprimer les conteneurs :**
    Le flag `-f` (force) arrête le conteneur s'il est en cours d'exécution avant de le supprimer.

    ```bash
    docker rm -f app-server log-shipper
    ```

2.  **Supprimer le volume partagé :**
    Une fois les conteneurs supprimés, le volume peut être nettoyé.

    ```bash
    docker volume rm shared-logs
    ```
