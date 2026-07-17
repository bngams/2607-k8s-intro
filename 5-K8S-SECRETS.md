# 05 — Secrets : gérer les données sensibles

> **Scénario à réaliser en autonomie.** Vous complétez vous-même les manifests à partir d'indices : les fichiers fournis dans `assets/attachments/k8s/secrets/` contiennent des `# TODO` à remplir. Un dossier `solution/` (à la fin) n'est là qu'en dernier recours.

Au [lab 04](4-K8S-CONFIGMAPS.md), on a externalisé la configuration *non sensible* avec une ConfigMap. Mais une URL de base de données contient un **mot de passe** — la stocker en clair dans une ConfigMap serait une faute. C'est le rôle du **Secret**. On va connecter une application à une base **MongoDB** via un Secret, puis lancer un **Job de sauvegarde** de cette base — ce qui nous mènera naturellement au besoin de **volumes persistants** (lab 06).

> **Prérequis cluster :** minikube démarré. Namespace dédié `secret-demo`.

---

## ✨ Objectifs

- Créer un **Secret** (3 méthodes) et comprendre que **base64 ≠ chiffrement**
- Consommer un Secret en **variable d'environnement** puis en **volume**
- Lancer un **Job** de backup MongoDB utilisant le Secret pour se connecter
- Comprendre pourquoi le backup est **perdu** sans volume persistant → transition lab 06

---

## 📁 Point de départ

```
secret-demo/ (votre dossier de travail)
├── mongo-secret.yaml   ← Secret via spec (à compléter, méthode 3)
├── mongo.yaml          ← MongoDB local + Service (fourni)
└── backup-job.yaml     ← Job mongodump (à compléter)
```

```bash
kubectl create namespace secret-demo
```

---

## 🔐 1 — Créer un Secret (3 méthodes)

Un **Secret** stocke des données sensibles. On va y mettre l'URL de connexion à notre MongoDB, sous la clé `mongo_url` :

```
mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin
```

> **D'où vient ce nom `mongo.secret-demo.svc.cluster.local` ?** C'est le FQDN DNS interne du Service `mongo` (même patron `<service>.<namespace>.svc.cluster.local` que vu au [lab 03](3-K8S-COMPLEMENTS-NAMESPACES.md)). Notre app et le Service mongo étant dans le même namespace, `mongo:27017` suffirait — on écrit le FQDN complet pour rester explicite.

Trois façons de créer le Secret `mongo` — **choisissez-en une** :

**Option 1 — depuis un fichier** (`--from-file`) :

```bash
echo -n "mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin" > mongo_url
kubectl create secret generic mongo --from-file=mongo_url -n secret-demo
```

**Option 2 — depuis une valeur littérale** (`--from-literal`) :

```bash
kubectl create secret generic mongo -n secret-demo \
  --from-literal=mongo_url='mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin'
```

**Option 3 — via un fichier de spécification.** Récupérez [assets/attachments/k8s/secrets/mongo-secret.yaml](assets/attachments/k8s/secrets/mongo-secret.yaml) et complétez le `# TODO`. Il faut d'abord encoder la valeur en base64 :

```bash
echo -n 'mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin' | base64
# => bW9uZ29kYjovL2s4c...  (collez ce résultat sous data.mongo_url)
kubectl apply -f mongo-secret.yaml
```

> **Astuce :** le champ `stringData` évite l'encodage manuel — Kubernetes encode pour vous. `data` attend du base64, `stringData` attend du texte clair. Les deux finissent stockés en base64.

### 🧪 Manip — base64 n'est PAS du chiffrement

Point de sécurité **fondamental** : un Secret n'est **pas chiffré**, juste encodé en base64 — trivialement réversible.

```bash
kubectl get secret mongo -n secret-demo -o jsonpath='{.data.mongo_url}'; echo
# => bW9uZ29kYjovL2s4c...   (base64, pas du chiffrement)

kubectl get secret mongo -n secret-demo -o jsonpath='{.data.mongo_url}' | base64 -d; echo
# => mongodb://k8sExercice:k8sExercice@...   (retrouvé en clair, instantanément)
```

> ⚖️ **Ce que "Secret" apporte vraiment (et pas).** Un Secret n'est pas plus confidentiel qu'une ConfigMap en soi : quiconque a les droits RBAC sur le namespace le lit. Ses vrais atouts : il n'apparaît pas dans les logs/`describe` par défaut, il peut être **chiffré au repos** (option `EncryptionConfiguration` de l'API server), et il ouvre la porte à des solutions dédiées — **Sealed Secrets** (chiffré dans git), **External Secrets** (tiré d'un coffre externe), **HashiCorp Vault** (serveur de secrets). Pour un vrai secret : ne jamais le committer en clair, `.gitignore` obligatoire.

---

## 🗄️ 2 — Déployer MongoDB (la base cible)

Récupérez [assets/attachments/k8s/secrets/mongo.yaml](assets/attachments/k8s/secrets/mongo.yaml) (fourni) et déployez la base + son Service :

```bash
kubectl apply -f mongo.yaml
kubectl -n secret-demo wait --for=condition=Ready pod/mongo --timeout=120s
```

> **Note honnête ⚖️ :** ce MongoDB stocke ses données dans le conteneur, **sans volume persistant** — si le pod meurt, la base repart de zéro. C'est volontairement imparfait ici (on n'a pas encore vu les volumes) ; on y reviendra au lab 06. Les identifiants sont aussi passés en clair via `env` — dans un vrai déploiement, ils viendraient d'un Secret.

---

## 🔌 3 — Consommer le Secret : env var puis volume

Un Secret se consomme comme une ConfigMap, de deux façons. On va prouver les deux dans un même Pod de test.

Décomposition des deux modes :

```yaml
    # --- Mode A : variable d'environnement ---
    env:
    - name: MONGODB_URL
      valueFrom:
        secretKeyRef:            # référence une clé d'un Secret
          name: mongo
          key: mongo_url

    # --- Mode B : fichier monté en volume ---
    volumeMounts:
    - name: mongo-creds
      mountPath: /run/secrets    # le fichier apparaîtra ici
      readOnly: true
  volumes:
  - name: mongo-creds
    secret:
      secretName: mongo
      items:
      - key: mongo_url           # la clé du Secret...
        path: MONGODB_URL        # ...montée sous ce nom de fichier (rename key -> path)
```

| Mode | Le secret apparaît comme | Quand le préférer |
|---|---|---|
| **env var** (`secretKeyRef`) | `$MONGODB_URL` | l'app lit une variable d'env |
| **volume** (`secret.items`) | fichier `/run/secrets/MONGODB_URL` | l'app lit un fichier (ex. `/run/secrets/...`) ; se met à jour sans redéploiement |

### 🧪 Manip — vérifier les deux modes

```bash
kubectl run consumer --image=busybox -n secret-demo --restart=Never --dry-run=client -o yaml > /dev/null
# Utilisez le Pod de test fourni dans solution/ (env + volume) OU construisez-le ;
# le point à observer : la même valeur, exposée des deux façons.
```

En pratique, un conteneur qui expose les deux affiche :

```
--- ENV ---
MONGODB_URL=mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin
--- VOLUME ---
mongodb://k8sExercice:k8sExercice@mongo.secret-demo.svc.cluster.local:27017/message?authSource=admin
```

> **Différence clé (env vs volume).** Une valeur injectée en **env var** est figée au démarrage du conteneur (comme la ConfigMap du lab 04 : changer le Secret ne la met pas à jour sans redéploiement). Un Secret monté en **volume**, lui, est **rafraîchi** par le kubelet quand le Secret change — l'app doit juste relire le fichier. C'est un argument fort pour le montage en volume des credentials rotés.

---

## 💾 4 — Un Job de backup qui utilise le Secret

On veut maintenant **sauvegarder** la base. Un backup est une tâche **ponctuelle** (pas un service qui tourne en continu) : c'est exactement le rôle d'un **Job**.

> **D'où vient l'objet `Job` ?** Un Deployment maintient N pods *en vie* en permanence. Un **Job** exécute un ou plusieurs pods **jusqu'à leur complétion réussie**, puis s'arrête. Idéal pour une migration, un import, ou ici un `mongodump`. Sa variante planifiée est le **CronJob** (voir plus bas).

Récupérez [assets/attachments/k8s/secrets/backup-job.yaml](assets/attachments/k8s/secrets/backup-job.yaml) et complétez les `# TODO` (le Job lit l'URL depuis le Secret) :

```yaml
        env:
        - name: MONGODB_URL
          valueFrom:
            secretKeyRef:
              name: ____________       # TODO : mongo
              key: ____________         # TODO : mongo_url
```

> 📖 [Kubernetes — Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

Ajoutez d'abord une donnée à sauvegarder, puis lancez le backup :

```bash
# insérer un document de test
kubectl -n secret-demo exec mongo -- mongosh -u k8sExercice -p k8sExercice \
  --authenticationDatabase admin --quiet \
  --eval 'db.getSiblingDB("message").messages.insertOne({from:"boris",msg:"hello"})'

kubectl apply -f backup-job.yaml
kubectl -n secret-demo wait --for=condition=complete job/mongo-backup --timeout=90s
kubectl -n secret-demo logs job/mongo-backup
```

Vous devez voir `mongodump` se connecter (grâce au Secret) et dumper la base :

```
Backup vers /backup ...
writing `message.messages` to `/backup/message/messages.bson`
done dumping `message.messages` (1 document)
--- contenu du backup ---
/backup/message: messages.bson  messages.metadata.json  prelude.json
```

**Le Secret a fait son travail** : le Job s'est authentifié sans qu'aucun mot de passe n'apparaisse dans son manifest.

---

## 🔗 5 — Le problème qui reste : où est le backup ?

Le backup existe... dans le système de fichiers du conteneur du Job. Or ce Job est **terminé** :

```bash
kubectl -n secret-demo get pods -l job-name=mongo-backup
# => mongo-backup-xxxxx   0/1   Completed
kubectl -n secret-demo get pvc
# => No resources found      <- rien n'a été persisté nulle part
```

Le `/backup` vivait dans le conteneur éphémère. Le pod terminé, **le backup est perdu**. Même problème pour les données de MongoDB lui-même (§2) : sans stockage durable, un redémarrage les efface.

> **Et si on voulait le faire tous les jours ?** On remplacerait le `Job` par un **CronJob** (`kind: CronJob`, avec un `schedule: "0 2 * * *"`), qui crée un Job à intervalle régulier. Mais que le backup soit ponctuel ou quotidien, **le résultat doit atterrir quelque part de durable**.

> **Il nous manque donc un VOLUME PERSISTANT** — capable de survivre à la mort d'un pod — pour y écrire les backups (et pour stocker les données de MongoDB). C'est précisément l'objet du **lab 06 : Volumes & stockage persistant** (PV, PVC, StorageClass, CSI).

---

## 🎉 Challenge final

- [ ] Créer un Secret par les 3 méthodes et prouver que base64 n'est pas du chiffrement
- [ ] Consommer un Secret en env var **et** en volume
- [ ] Lancer un Job qui se connecte à MongoDB via le Secret
- [ ] Expliquer pourquoi le backup ne survit pas, et ce qu'il faudrait pour y remédier

---

## ✅ Bonus

- **CronJob** : transformer le Job en sauvegarde quotidienne (`schedule`, `successfulJobsHistoryLimit`).
- **Secret typé** : `kubernetes.io/dockerconfigjson` pour tirer d'un registre privé, `kubernetes.io/tls` pour un certificat.
- **Chiffrement au repos** : activer l'`EncryptionConfiguration` de l'API server (secrets chiffrés dans etcd).

---

## 🧹 Nettoyage

```bash
kubectl delete ns secret-demo
```

---

## Récap

| Notion | Rôle |
|---|---|
| **Secret** | données sensibles, encodées base64 (**pas chiffrées**) |
| **`secretKeyRef`** | consommer en variable d'environnement (figée au démarrage) |
| **`secret.items`** | monter en volume (rafraîchi si le Secret change) |
| **Job** | tâche ponctuelle jusqu'à complétion (vs Deployment continu) |
| **CronJob** | Job planifié à intervalle régulier |
| **Le manque** | rien n'est persisté → besoin d'un volume (lab 06) |

➡️ **Suite : [06 — Volumes & stockage persistant](6-K8S-STORAGE.md)**
