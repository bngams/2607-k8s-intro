# 07 — Cas global : déployer PetClinic (Java + MySQL) sur Kubernetes

> **Scénario de synthèse à réaliser en autonomie.** Ce cas mobilise **tout** ce que vous avez vu dans la série : namespaces, Deployment, Service, Ingress, ConfigMap, Secret, StatefulSet + PVC, HPA. Les fichiers fournis dans `assets/attachments/k8s/petclinic/` contiennent des `# TODO` à compléter. Un `solution/` complet est là en dernier recours.

On déploie **Spring PetClinic** — une application Java (Spring Boot) adossée à une base **MySQL**. L'app est fournie **déjà buildée** en image Docker (`techiescamp/kube-petclinic-app:3.0.0`) : on se concentre entièrement sur Kubernetes, pas sur le build Java.

> **Prérequis cluster :** minikube démarré, addon **ingress** actif. On activera **metrics-server** à l'étape HPA. Le CNI par défaut suffit.

> **Vous ne connaissez pas Java / Spring ?** Pas d'inquiétude : tout ce qui est spécifique à Spring (la commande de lancement, le point de montage de la config) vous est **donné**. Vous appliquez uniquement des concepts Kubernetes déjà vus.

---

## ✨ Objectifs

- Assembler une application **multi-tiers** (app + base de données) sur Kubernetes
- Isoler app et DB dans **deux namespaces** distincts
- Relier l'app à la DB via le **FQDN DNS** d'un service cross-namespace ([lab 03](3-K8S-COMPLEMENTS-NAMESPACES.md))
- Configurer l'app par **ConfigMap** ([lab 04](4-K8S-CONFIGMAPS.md)) et **Secret** ([lab 05](5-K8S-SECRETS.md))
- Persister la DB avec un **StatefulSet + PVC** ([lab 06](6-K8S-STORAGE.md))
- Exposer par **Ingress** et mettre à l'échelle avec un **HPA** ([lab 01](1-K8S-INTRO.md))

---

## 🗺️ Architecture cible

```mermaid
graph TD
  User["navigateur<br/>petclinic.local"] --> Ing["Ingress"]
  Ing --> SVC["Service java-app :80"]
  SVC --> APP["Deployment java-app<br/>(PetClinic, :8080)"]
  APP -->|"jdbc mysql.pet-clinic-db.svc.cluster.local:3306"| DBSVC["Service mysql (headless)"]
  DBSVC --> STS["StatefulSet mysql"]
  STS --> PVC["PVC data-mysql-0"]
  CM["ConfigMap<br/>application.properties"] -.->|monté /opt/config| APP
  SEC["Secret mysql-secret"] -.->|env DB_USERNAME/PASSWORD| APP
  SEC -.->|env MYSQL_USER/PASSWORD| STS
  HPA["HPA 2→5 @cpu50%"] -.->|scale| APP
  subgraph ns_app["ns pet-clinic-app"]
    Ing; SVC; APP; CM; HPA
  end
  subgraph ns_db["ns pet-clinic-db"]
    DBSVC; STS; PVC
  end
  style ns_app fill:#dcfce7,color:#000
  style ns_db fill:#dbeafe,color:#000
```

## 📁 Arborescence des fichiers

```
petclinic/
├── namespaces.yaml         ← 2 namespaces (fourni)
├── secrets.yml             ← Secret MySQL, présent dans les 2 ns (à compléter)
├── mysql/
│   └── db.yml              ← Service headless + StatefulSet + PVC (fourni)
└── app/
    ├── app.configmap.yml   ← application.properties, URL JDBC (à compléter)
    ├── app.deploy.yml      ← Deployment (command fournie, branchements à compléter)
    ├── app.svc.yml         ← Service (à compléter)
    ├── app.ingress.yml     ← Ingress (fourni)
    └── app.hpa.yml         ← HPA (à compléter)
```

---

## 🧱 1 — Namespaces & Secret

On isole l'app et la DB dans deux namespaces. Le Secret des identifiants MySQL doit exister **dans les deux** : MySQL s'en sert pour créer l'utilisateur, l'app pour se connecter.

Appliquez les namespaces (fourni), puis complétez le Secret ([secrets.yml](assets/attachments/k8s/petclinic/secrets.yml)) :

```yaml
data:
  username: ____________     # TODO : base64 (ex: echo -n 'petclinic' | base64)
  password: ____________     # TODO : base64
```

> **Rappel base64 ≠ chiffrement** ([lab 05](5-K8S-SECRETS.md)) : `data` attend du base64. Pour éviter l'encodage manuel, vous pouvez utiliser `stringData:` (texte clair, Kubernetes encode). Les deux Secrets (app + db) doivent porter les **mêmes** valeurs.

```bash
kubectl apply -f namespaces.yaml
kubectl apply -f secrets.yml
```

---

## 🗄️ 2 — MySQL : StatefulSet + PVC

La base de données doit **persister** : on la déploie en **StatefulSet** avec un volume (via `volumeClaimTemplates`), pas en Pod nu.

> **Pourquoi un StatefulSet et pas un Deployment ?** Une base de données a besoin d'une **identité stable** et d'un **volume propre qui la suit**. Le StatefulSet garantit ça : un nom de pod stable (`mysql-0`) et un PVC dédié par réplica. Il exige un **Service headless** (`clusterIP: None`) pour l'adressage. (Le PVC seul, vu au [lab 06](6-K8S-STORAGE.md), suffirait pour une instance unique — le StatefulSet est la forme correcte en production.)

Le fichier [mysql/db.yml](assets/attachments/k8s/petclinic/mysql/db.yml) est **fourni** (Service headless + StatefulSet + PVC + branchement du Secret). Appliquez-le :

```bash
kubectl apply -f mysql/db.yml
kubectl -n pet-clinic-db rollout status statefulset/mysql --timeout=180s
kubectl -n pet-clinic-db get pvc      # data-mysql-0  Bound
```

> Le premier démarrage initialise la base `petclinic` sur un volume vide — laissez-lui le temps (1-2 min).

---

## ⚙️ 3 — ConfigMap de l'app : l'URL de la base

L'app lit sa configuration Spring dans un fichier `application.properties`, qu'on lui fournit via une **ConfigMap montée en volume** ([lab 04](4-K8S-CONFIGMAPS.md)). Le point central : **l'URL JDBC de la base**.

MySQL vit dans le namespace `pet-clinic-db`, l'app dans `pet-clinic-app`. Un nom court comme `mysql` ne suffit donc **pas** — il faut le **FQDN complet** du service, cross-namespace ([lab 03](3-K8S-COMPLEMENTS-NAMESPACES.md)) :

```
<service>.<namespace>.svc.cluster.local
```

Complétez [app/app.configmap.yml](assets/attachments/k8s/petclinic/app/app.configmap.yml) :

```properties
    # TODO : FQDN du service mysql (ns pet-clinic-db), port 3306, base "petclinic"
    spring.datasource.url=jdbc:mysql://____________:____________/____________
    spring.datasource.username=${DB_USERNAME}
    spring.datasource.password=${DB_PASSWORD}
```

> **`${DB_USERNAME}` / `${DB_PASSWORD}`** ne sont pas des placeholders Kubernetes : ce sont des variables que **Spring** substitue au démarrage, à partir des **variables d'environnement** du conteneur — qu'on alimentera depuis le Secret à l'étape suivante.

```bash
kubectl apply -f app/app.configmap.yml
```

---

## 🚀 4 — Deployment de l'app

C'est le cœur de l'assemblage. L'image et la **commande de lancement** sont fournies (spécifiques à Spring Boot — ne les modifiez pas) ; à vous de **brancher le Secret** (env) et **monter la ConfigMap** (volume).

> ⚠️ **Les deux pièges de ce conteneur** (vos notes de départ) :
> - **L'image** : `techiescamp/kube-petclinic-app:3.0.0`.
> - **La commande** : elle indique à Spring où lire la config (le fichier monté par la ConfigMap) et active le profil `mysql`. Elle est fournie telle quelle :
>   ```
>   command: ["java", "-jar", "/app/java.jar",
>             "--spring.config.location=/opt/config/application.properties",
>             "--spring.profiles.active=mysql"]
>   ```
>   Le `mountPath` de la ConfigMap **doit** donc être `/opt/config` — sinon Spring ne trouve pas sa config.

Complétez [app/app.deploy.yml](assets/attachments/k8s/petclinic/app/app.deploy.yml) — les `# TODO` : `secretKeyRef` (name/key) pour `DB_USERNAME`/`DB_PASSWORD`, le `mountPath` de la config, et le nom de la ConfigMap.

```bash
kubectl apply -f app/app.deploy.yml
kubectl -n pet-clinic-app rollout status deploy/java-app --timeout=240s
```

Vérifiez dans les logs que l'app a démarré **et s'est connectée à MySQL** :

```bash
kubectl -n pet-clinic-app logs deploy/java-app | grep -iE "HikariPool.*Added connection|Started PetClinicApplication"
# => HikariPool-1 - Added connection ... mysql.cj.jdbc...
# => Started PetClinicApplication in ... seconds
```

> **Symptôme → Cause → Correctif.** Si le pod `CrashLoopBackOff` avec une erreur de connexion JDBC : (1) l'URL de la ConfigMap ne pointe pas sur le bon FQDN cross-namespace (étape 3) ; (2) le Secret n'est pas branché en env (les `${DB_...}` restent vides) ; (3) MySQL n'est pas encore prêt. Vérifiez dans cet ordre.

---

## 🌐 5 — Service & Ingress

Exposez l'app : d'abord un **Service** (l'app écoute sur `8080`, on l'expose sur `80`), puis un **Ingress** sur `petclinic.local`.

Complétez [app/app.svc.yml](assets/attachments/k8s/petclinic/app/app.svc.yml) (`selector` + `targetPort`), l'[Ingress](assets/attachments/k8s/petclinic/app/app.ingress.yml) est fourni :

```bash
kubectl apply -f app/app.svc.yml
kubectl apply -f app/app.ingress.yml

# résoudre petclinic.local en local (comme au lab 01)
echo "$(minikube ip)  petclinic.local" | sudo tee -a /etc/hosts
# ou avec `minikube tunnel` actif :  echo "127.0.0.1  petclinic.local" | sudo tee -a /etc/hosts
```

Ouvrez [http://petclinic.local](http://petclinic.local) → la page **PetClinic** s'affiche. 🎉

```bash
# test en ligne de commande
curl -s http://petclinic.local | grep -o '<title>[^<]*</title>'
# => <title>PetClinic :: a Spring Framework demonstration</title>
```

---

## 📈 6 — Autoscaling (HPA)

Enfin, on ajoute un **Horizontal Pod Autoscaler** pour mettre l'app à l'échelle selon la charge CPU ([lab 01](1-K8S-INTRO.md)).

Activez d'abord le **metrics-server** (le HPA en a besoin pour lire les métriques) :

```bash
minikube addons enable metrics-server
```

Complétez [app/app.hpa.yml](assets/attachments/k8s/petclinic/app/app.hpa.yml) (nom du Deployment cible + seuil) et appliquez :

```bash
kubectl apply -f app/app.hpa.yml
kubectl -n pet-clinic-app get hpa java-app-hpa
# NAME           REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS
# java-app-hpa   Deployment/java-app   cpu: 18%/50%  2         5         2
```

> Le HPA passe à **2 replicas** (son `minReplicas`) dès qu'il lit les métriques. Le `cpu: <unknown>` initial disparaît une fois metrics-server prêt (~30s). Les `resources.requests.cpu` du Deployment sont **indispensables** — sans eux, pas de pourcentage à calculer.

---

## 🎉 Challenge final

Vous avez un déploiement multi-tiers complet. Vérifiez :

- [ ] `kubectl get all -n pet-clinic-app` et `-n pet-clinic-db` : tout est `Running`
- [ ] PetClinic répond sur `http://petclinic.local`
- [ ] Les données MySQL survivent à un `kubectl delete pod mysql-0 -n pet-clinic-db` (grâce au PVC)
- [ ] Le HPA affiche `2/5` replicas et un `%` de CPU
- [ ] Vous savez tracer le chemin d'une requête : Ingress → Service → app → (FQDN) → MySQL

---

## ✅ Bonus

- **Sondes de santé** : ajouter `livenessProbe` / `readinessProbe` sur `/actuator/health` (exposé par l'app).
- **Charge + scaling** : générer du trafic (`hey`, `ab`, ou une boucle `curl`) et observer le HPA monter les replicas.
- **Secret pour root** : `MYSQL_ROOT_PASSWORD` est en clair dans `db.yml` — le passer par le Secret.
- **GitOps** : faire déployer tout ce dossier par ArgoCD ([lab 02](2-K8S-INTRO-ARGO.md)).

---

## 🧹 Nettoyage

```bash
kubectl delete namespace pet-clinic-app pet-clinic-db
# retirez aussi la ligne petclinic.local de /etc/hosts
```

---

## Récap — ce que ce cas a mobilisé

| Brique | Ressource | Lab d'origine |
|---|---|---|
| Isolation | 2 Namespaces | 01 / 03 |
| Config app | ConfigMap montée en volume | 04 |
| Identifiants | Secret (env var) | 05 |
| Base de données | StatefulSet + PVC | 06 |
| Résolution app→DB | FQDN DNS cross-namespace | 03 |
| Exécution app | Deployment | 01 |
| Exposition | Service + Ingress | 01 |
| Mise à l'échelle | HPA + metrics-server | 01 |

➡️ **Précédent : [06 — Volumes & stockage persistant](6-K8S-STORAGE.md)**
