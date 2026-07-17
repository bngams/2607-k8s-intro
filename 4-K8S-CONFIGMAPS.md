# 04 — ConfigMaps : externaliser la configuration

> **Scénario à réaliser en autonomie.** Vous complétez vous-même les manifests à partir d'indices : les fichiers fournis dans `assets/attachments/k8s/configmaps/` contiennent des `# TODO` à remplir. Un dossier `solution/` (à la fin) n'est là qu'en dernier recours.

Jusqu'ici, la configuration de nos apps était **figée dans l'image** ou écrite en dur dans le manifest. On introduit maintenant l'objet qui permet de l'en sortir : la **ConfigMap**. On l'utilisera pour configurer un reverse-proxy nginx, puis on affrontera le piège classique : *que se passe-t-il quand on met la config à jour ?*

> **Prérequis cluster :** minikube démarré (le CNI par défaut suffit ici, pas besoin de Calico). On travaille dans un namespace dédié `cm-demo`.

---

## ✨ Objectifs

- Comprendre ce qu'est une **ConfigMap** et pourquoi découpler config et image
- Monter une ConfigMap **en volume** pour configurer un conteneur (nginx reverse-proxy)
- Découvrir le **piège** : mettre à jour une ConfigMap ne redéploie pas le Pod
- Appliquer la parade **`CONFIG_HASH`** pour forcer un rollout à chaque changement de config

---

## 📁 Point de départ

```
cm-demo/ (votre dossier de travail pour ce lab)
├── nginx.conf        ← config du reverse-proxy (fournie)
├── proxy.yaml        ← Pod nginx (à compléter)
├── www.conf          ← config d'un nginx statique (fournie)
└── deploy.yaml       ← Deployment nginx (à compléter)
```

Créez le namespace :

```bash
kubectl create namespace cm-demo
```

---

## 🧩 1 — Qu'est-ce qu'une ConfigMap ?

Une **ConfigMap** stocke des données de configuration **non sensibles** sous forme de paires clé/valeur (des chaînes, ou des fichiers entiers). Un Pod peut ensuite les consommer de deux façons :

| Mode de consommation | Quand l'utiliser |
|---|---|
| **Variables d'environnement** | quelques valeurs simples (`LOG_LEVEL`, `PORT`...) |
| **Fichiers montés en volume** | un **fichier de config complet** (nginx.conf, application.yaml...) |

> Pour des données **sensibles** (mots de passe, tokens, URL avec credentials), on utilise un **Secret** — c'est l'objet du [lab 05](5-K8S-SECRETS.md). Une ConfigMap, elle, est stockée en clair et lisible par quiconque a accès au namespace.

Dans ce lab on va monter un **fichier** (la config nginx) en volume — c'est le cas d'usage le plus courant et le plus parlant.

---

## 🔀 2 — Un reverse-proxy configuré par ConfigMap

On va mettre en place un proxy nginx qui route `/whoami` vers un service `whoami` interne. La config nginx ne sera pas dans l'image : elle vivra dans une ConfigMap, montée dans le conteneur.

### Déployer la cible `whoami`

D'abord la cible du proxy — un `traefik/whoami` (déjà croisé au lab 01) exposé par un Service `ClusterIP` :

```bash
kubectl run poddy --image=traefik/whoami -n cm-demo --labels=app=whoami
kubectl expose pod poddy --name=whoami --port=80 --target-port=80 -n cm-demo
kubectl -n cm-demo wait --for=condition=Ready pod/poddy --timeout=60s
```

### Créer la ConfigMap depuis un fichier

Récupérez [assets/attachments/k8s/configmaps/nginx.conf](assets/attachments/k8s/configmaps/nginx.conf) (fourni) dans votre dossier. Puis créez la ConfigMap **à partir de ce fichier** :

```bash
kubectl create configmap proxy-config --from-file=./nginx.conf -n cm-demo
```

> **D'où vient la clé de la ConfigMap ?** Avec `--from-file=./nginx.conf`, Kubernetes crée une entrée dont la **clé est le nom du fichier** (`nginx.conf`) et la valeur est son contenu. Inspectez-la : `kubectl get cm proxy-config -n cm-demo -o yaml`. C'est ce nom de clé qui deviendra le nom du fichier une fois monté.

### Monter la ConfigMap dans le Pod nginx

Pour que nginx lise cette config, on la **monte en volume** dans son conteneur. Récupérez [assets/attachments/k8s/configmaps/proxy.yaml](assets/attachments/k8s/configmaps/proxy.yaml) et complétez les `# TODO` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: proxy
  namespace: cm-demo
  labels:
    app: proxy
spec:
  containers:
  - name: proxy
    image: nginx:1.27-alpine
    volumeMounts:
    - name: config
      mountPath: "____________"     # TODO : /etc/nginx/ (nginx y cherche nginx.conf)
  volumes:
  - name: config
    configMap:
      name: ____________           # TODO : proxy-config
```

> 📖 [ConfigMaps as files from a Pod](https://kubernetes.io/docs/concepts/configuration/configmap/#using-configmaps-as-files-from-a-pod)

Appliquez et testez le routage :

```bash
kubectl apply -f proxy.yaml
kubectl -n cm-demo wait --for=condition=Ready pod/proxy --timeout=60s

# tester le proxy : /whoami doit être forwardé vers le service whoami
kubectl -n cm-demo exec proxy -- wget -qO- http://127.0.0.1/whoami
# => Hostname: poddy
#    IP: ...
```

> ⚠️ **Testez sur `127.0.0.1`, pas `localhost`.** Dans le conteneur, `localhost` peut résoudre en IPv6 (`::1`) où nginx n'écoute pas forcément tout de suite — vous obtiendriez un `Connection refused` trompeur. `127.0.0.1` force l'IPv4. (En dehors du conteneur, préférez un `kubectl port-forward proxy 8080:80 -n cm-demo` puis `curl 127.0.0.1:8080/whoami`.)

La réponse `Hostname: poddy` prouve que le proxy a bien lu sa config depuis la ConfigMap et route vers whoami. **On a découplé la configuration de l'image** : changer le routage ne demande plus de rebuild.

---

## ⚠️ 3 — Le piège : mettre à jour une ConfigMap

Puisque la config est externalisée, on devrait pouvoir la changer sans rebuild. Voyons ce qui se passe réellement. On repart sur un cas plus simple à observer : un nginx statique dont on change juste le **port d'écoute**.

Récupérez [www.conf](assets/attachments/k8s/configmaps/www.conf) et [deploy.yaml](assets/attachments/k8s/configmaps/deploy.yaml). Créez la ConfigMap et le Deployment :

```bash
# la clé dans la ConfigMap doit s'appeler nginx.conf (nginx la cherche sous ce nom)
kubectl create configmap www-config --from-file=nginx.conf=./www.conf -n cm-demo
kubectl apply -f deploy.yaml      # après avoir complété le TODO (name: www-config)
kubectl -n cm-demo rollout status deploy/www
```

> **`--from-file=nginx.conf=./www.conf`** : la syntaxe `<clé>=<fichier>` force le nom de la clé (`nginx.conf`) indépendamment du nom du fichier source (`www.conf`). Sans le préfixe, la clé serait `www.conf` et nginx ne trouverait pas sa config.

Vérifiez que nginx écoute sur `8080` :

```bash
POD=$(kubectl -n cm-demo get po -l app=www -o jsonpath='{.items[0].metadata.name}')
kubectl -n cm-demo exec $POD -- grep listen /etc/nginx/nginx.conf
# =>     listen 8080;
```

### 🧪 Manip — changer le port et observer

On met la ConfigMap à jour (`8080` → `9090`) et on regarde ce qui se passe **sans rien faire d'autre** :

```bash
# éditer www.conf : remplacer 8080 par 9090, puis re-générer la ConfigMap
sed -i '' 's/8080/9090/' www.conf     # macOS  (Linux : sed -i 's/8080/9090/' www.conf)
kubectl create configmap www-config --from-file=nginx.conf=./www.conf -n cm-demo \
  --dry-run=client -o yaml | kubectl apply -f -

# le Pod a-t-il été recréé ?
kubectl -n cm-demo get po -l app=www
# => MÊME pod, même AGE qui continue de courir — AUCUN redéploiement
```

Attendez ~1 minute (le kubelet resynchronise le volume), puis regardez le fichier **monté** vs le port **réellement servi** :

```bash
kubectl -n cm-demo exec $POD -- grep listen /etc/nginx/nginx.conf
# =>     listen 9090;      <- le FICHIER monté est bien à jour...

kubectl apply -f deploy.yaml
# => deployment.apps/www unchanged      <- ...mais Kubernetes ne voit AUCUN changement
```

**Le constat, en deux temps :**

- Le **fichier monté** finit par refléter la nouvelle valeur (`9090`) — la ConfigMap se propage au volume.
- Mais **nginx tourne toujours avec l'ancienne config** (il l'a lue une seule fois au démarrage) et **le Pod n'a pas été recréé**.

> **Pourquoi ?** Un Deployment ne se redéploie que si la **spec de son template de Pod** (`.spec.template`) change. Or modifier une ConfigMap ne touche pas ce template — la référence `configMap: {name: www-config}` est identique. Kubernetes ne voit donc aucune raison de recréer le Pod. Le contenu du volume change, mais le processus déjà démarré, lui, ne le relit pas.

---

## ✅ 4 — La parade : `CONFIG_HASH`

Il vaut mieux rendre le redéploiement **explicite** : on injecte dans le template de Pod une variable d'environnement contenant un **hash de la ConfigMap**. Quand la config change, le hash change → le template change → Kubernetes redéploie. Le hash ne sert à rien fonctionnellement : sa seule utilité est de **faire bouger la spec**.

Dé-commentez le bloc `env` dans votre `deploy.yaml` :

```yaml
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        volumeMounts:
        - name: config
          mountPath: "/etc/nginx/"
        env:
        - name: CONFIG_HASH
          value: ${CONFIG_HASH}       # sera substitué par envsubst
```

Puis calculez le hash et appliquez en substituant la variable :

```bash
export CONFIG_HASH=$(kubectl -n cm-demo get cm www-config -o yaml | sha256sum | cut -d' ' -f1)
echo "hash = $CONFIG_HASH"

envsubst '${CONFIG_HASH}' < deploy.yaml | kubectl apply -f -
# => deployment.apps/www configured      <- cette fois, un VRAI changement
kubectl -n cm-demo rollout status deploy/www
```

> **D'où vient `envsubst` ?** C'est un utilitaire GNU (`gettext`) qui remplace les `${VAR}` d'un fichier par les variables d'environnement correspondantes. Ici il transforme `value: ${CONFIG_HASH}` en `value: <le sha256>` juste avant le `kubectl apply`, sans modifier le fichier sur disque. Sur macOS : `brew install gettext` s'il manque.

Vérifiez qu'un **nouveau** Pod a été créé et qu'il sert enfin la bonne config :

```bash
kubectl -n cm-demo get po -l app=www
# => un NOUVEAU nom de pod (le hash a déclenché le rollout)
NEWPOD=$(kubectl -n cm-demo get po -l app=www -o jsonpath='{.items[0].metadata.name}')
kubectl -n cm-demo exec $NEWPOD -- grep listen /etc/nginx/nginx.conf
# =>     listen 9090;      ✅ nginx écoute enfin sur le nouveau port
```

> ⚖️ **Remarque de conception.** Ce pattern `CONFIG_HASH` + `envsubst` est artisanal (on pilote le hash à la main). En production, on laisse souvent un outil s'en charger : **Kustomize** génère des ConfigMaps *avec un suffixe de hash* (`configMapGenerator`), **Helm** peut poser une annotation `checksum/config`, et les outils GitOps (ArgoCD du [lab 02](2-K8S-INTRO-ARGO.md)) rejouent le tout. Le mécanisme sous-jacent reste identique : **faire changer la spec du Pod pour déclencher un rollout**.

---

## 🎉 Challenge final

Vous savez répondre par une manip :

- [ ] Configurer un conteneur via une ConfigMap montée en volume
- [ ] Expliquer pourquoi éditer une ConfigMap ne redéploie pas le Pod
- [ ] Forcer un redéploiement propre quand la config change (`CONFIG_HASH`)
- [ ] Dire quand préférer une variable d'env vs un fichier monté

---

## ✅ Bonus

- **ConfigMap en variables d'env** : `envFrom.configMapRef` pour injecter toutes les clés d'un coup.
- **Sous-ensemble de clés** : `configMap.items` pour ne monter que certaines clés (comme on le fera avec les Secrets au lab 05).
- **`immutable: true`** : rendre une ConfigMap immuable (perf + sécurité) — toute modif impose alors de la recréer.

---

## 🧹 Nettoyage

```bash
kubectl delete ns cm-demo
```

---

## Récap

| Notion | Rôle |
|---|---|
| **ConfigMap** | données de config **non sensibles**, en clair |
| **Montée en volume** | un fichier de config complet dans le conteneur |
| **Montée en env var** | quelques valeurs simples |
| **Le piège** | éditer la ConfigMap ne redéploie pas le Pod (le template ne change pas) |
| **`CONFIG_HASH`** | faire changer la spec du Pod pour forcer le rollout |

➡️ **Suite : [05 — Secrets](5-K8S-SECRETS.md)**
