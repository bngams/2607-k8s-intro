# 04 — Stockage, ConfigMaps & Secrets

> **⚠️ STUB — lab en cours de rédaction.** Ce fichier est un squelette structurel. Boris fournira ses **éléments de base** (manifests, exemples) qui seront intégrés ici en respectant le style « à construire soi-même » des labs 01→03 (why-tables, `# TODO`, liens doc, manips 🧪, `solution/`).
>
> **Sont déjà validés sur le cluster** (donc les sections ci-dessous tiendront la route) :
> - StorageClass par défaut `standard` (provisioner `k8s.io/minikube-hostpath`) → PVC à provisionnement dynamique OK
> - addon `storage-provisioner` actif

---

## ✨ Objectifs (prévisionnels)

- Externaliser la **configuration** hors de l'image (ConfigMap) — env vars & fichiers montés
- Gérer les **données sensibles** (Secret) et comprendre `base64 ≠ chiffrement`
- Comprendre la chaîne **PV → PVC → StorageClass** et le rôle du **CSI**
- Monter un **volume persistant** dans un pod et vérifier la persistance au redémarrage
- Situer `emptyDir` / `hostPath` / PVC et quand utiliser quoi

---

## 🗺️ Plan prévu (à remplir avec tes éléments)

### §1 — CSI : l'interface de stockage
- Symétrie CNI (réseau) / CRI (exécution) / **CSI** (stockage) — rappel du lab 02
- StorageClass, provisioning dynamique vs statique
- 📖 [kubernetes.io — Storage](https://kubernetes.io/docs/concepts/storage/)

### §2 — ConfigMap : externaliser la config
- Créer une ConfigMap (littéral, `--from-file`, YAML)
- Deux modes de consommation : **variables d'env** vs **fichiers montés** (volume)
- Why-table : quand env var, quand fichier monté
- `🚧 À compléter` : `whoami.configmap.yaml` + montage dans le Deployment
- 🧪 Manip : changer la ConfigMap → observer (et le piège : pas de rechargement auto des env vars)
- 📖 [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

### §3 — Secret : données sensibles
- Créer un Secret (`kubectl create secret generic ... --from-literal`)
- ⚠️ **`base64` n'est PAS du chiffrement** — montrer `kubectl get secret -o yaml` + `base64 -d`
- Consommation : env var (`secretKeyRef`) vs fichier monté
- Two epochs / alternatives : Secret natif → **Sealed Secrets** / **External Secrets** / **Vault**
- Sécurité : ne jamais committer un Secret en clair ; `.gitignore`
- 📖 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

### §4 — Volumes éphémères
- `emptyDir` (partage entre conteneurs d'un pod, perdu à la mort du pod)
- `hostPath` (⚠️ couplage au nœud, à éviter en prod — flag le shortcut)

### §5 — PV / PVC / StorageClass (persistance réelle)
- Chaîne PV ← PVC ← StorageClass, `accessModes`, `reclaimPolicy`
- `🚧 À compléter` : `whoami.pvc.yaml` + montage
- 🧪 Manip : écrire un fichier, `kubectl delete pod`, vérifier que la donnée survit
- Quota storage (lien lab 02 : `requests.storage`, `persistentvolumeclaims`)
- 📖 [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### 🎉 Challenge final
- App whoami (ou app avec état) : config via ConfigMap, credential via Secret, donnée persistée via PVC

---

## 📎 Éléments attendus de Boris

- [ ] Manifests de base (ConfigMap / Secret / PVC) ou app cible retenue
- [ ] App à état si différente de whoami (ex. postgres, redis, un compteur...) ?
- [ ] Cas d'usage concret à illustrer pour la persistance

➡️ **Précédent : [03 — Introduction à ArgoCD](3-K8S-INTRO-ARGO.md)**
