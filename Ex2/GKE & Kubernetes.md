## 🎯 Objectif
Comprendre les concepts Kubernetes sur GCP pour la certification Professional Cloud DevOps Engineer.
 
> ⚠️ **Note sandbox :** La contrainte `constraints/compute.vmExternalIpAccess` de la sandbox entreprise empêche la création de nœuds GKE (IP externe interdite). Cet exercice a été réalisé en mode prépa examen.
 
---
 
## 🛠️ Commandes réalisées en sandbox
 
### 1. Activer l'API GKE
```bash
gcloud services enable container.googleapis.com
```
 
### 2. Créer un réseau VPC
```bash
gcloud compute networks create mon-vpc \
  --subnet-mode=auto
```
> La sandbox n'a pas de réseau "default" — on doit créer le VPC manuellement.
 
### 3. Créer le cluster GKE
```bash
# Tentative finale (bloquée par contrainte sandbox)
gcloud container clusters create mon-cluster \
  --zone=europe-west1-b \
  --num-nodes=1 \
  --machine-type=e2-micro \
  --network=mon-vpc \
  --no-enable-basic-auth \
  --enable-ip-alias
```
 
> **En conditions normales (hors sandbox), la commande standard est :**
```bash
gcloud container clusters create mon-cluster \
  --zone=europe-west1-b \
  --num-nodes=3 \
  --machine-type=e2-standard-2 \
  --network=default
```
 
### 4. Se connecter au cluster
```bash
gcloud container clusters get-credentials mon-cluster \
  --zone=europe-west1-b
```
> Configure `kubectl` pour pointer sur le cluster.
 
### 5. Vérifier les nœuds
```bash
kubectl get nodes
```
 
### 6. Vérifier les pods
```bash
kubectl get pods
kubectl get pods -n kube-system     # pods système
kubectl get pods --all-namespaces   # tous les namespaces
```
 
---
 
## 🧹 Nettoyage (obligatoire après chaque exercice)
 
```bash
# Supprimer le cluster
gcloud container clusters delete mon-cluster \
  --zone=europe-west1-b \
  --quiet
 
# Supprimer le VPC
gcloud compute networks delete mon-vpc --quiet
```
 
---
 
## ⚠️ Problèmes rencontrés & solutions
 
| Problème | Cause | Solution |
|---|---|---|
| `No network named "default"` | Sandbox sans réseau par défaut | Créer un VPC manuellement |
| `vmExternalIpAccess violated` | Sandbox interdit les IPs externes sur les VMs | Contrainte permanente — GKE impossible dans cette sandbox |
| `Cannot specify --enable-private-nodes without --enable-ip-alias` | Option manquante | Ajouter `--enable-ip-alias` |
| `Already exists` | Cluster partiellement créé | Supprimer et recréer |
 
---
 
## 🏗️ Les objets Kubernetes fondamentaux
 
| Objet | Rôle | Analogie |
|---|---|---|
| **Pod** | Plus petite unité — contient 1 ou plusieurs conteneurs | Un appartement |
| **Deployment** | Gère les pods — assure le bon nombre de replicas | Le promoteur immobilier |
| **Service** | Expose les pods sur le réseau avec une IP stable | L'adresse postale |
 
> 💡 Un Pod peut mourir et être recréé avec une **nouvelle IP** à chaque fois. Le Service lui donne une **IP fixe**.
 
---
 
## ⚙️ Les composants du Control Plane
 
| Composant | Rôle |
|---|---|
| **Controller Manager** | Détecte les pods manquants et demande à en recréer |
| **Scheduler** | Décide sur quel nœud placer les pods selon les rules |
 
### Rules de scheduling
- `requests` / `limits` → ressources CPU/RAM demandées et maximales
- `taints` / `tolerations` → repousse ou attire les pods sur certains nœuds
- `affinity` / `anti-affinity` → règles de co-localisation des pods
---
 
## 🔄 Stratégies de déploiement
 
| Stratégie | Fonctionnement | Interruption |
|---|---|---|
| **RollingUpdate** (défaut) | Remplace les pods progressivement | ❌ Aucune |
| **Recreate** | Coupe tout puis recrée | ✅ Oui |
| **Blue/Green** | 2 environnements complets, bascule via le Service | ❌ Aucune |
| **Canary** | Envoie X% du trafic vers la nouvelle version | ❌ Aucune |
 
### Configuration Rolling Update
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # pods en PLUS autorisés pendant la mise à jour
    maxUnavailable: 0    # 0 = zéro interruption de service
```
 
---
 
## ⏪ Rollback
 
```bash
# Revenir à la version précédente
kubectl rollout undo deployment/mon-app
 
# Revenir à une version spécifique
kubectl rollout undo deployment/mon-app --to-revision=2
 
# Voir l'historique des déploiements
kubectl rollout history deployment/mon-app
```
 
> 💡 GKE conserve par défaut les **10 dernières révisions** — paramètre `revisionHistoryLimit`.
 
---
 
## 📈 Scaling automatique
 
| Objet | Rôle |
|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Scale le nombre de **pods** horizontalement |
| **VPA** (Vertical Pod Autoscaler) | Ajuste les **resources** (CPU/RAM) d'un pod |
| **Cluster Autoscaler** | Scale le nombre de **nœuds** du cluster |
 
### Exemple HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # scale si CPU > 70%
```
 
---
 
## 🔐 Workload Identity — Permissions sécurisées
 
> **Problème :** Comment donner des permissions GCP à un pod **sans clé JSON dans le code** ?
> **Solution : Workload Identity**
 
### Comment ça marche
```
Pod GKE
  └── Kubernetes Service Account (KSA)
          └── lié à un Google Service Account (GSA)
                  └── qui a les permissions IAM GCP
```
 
### Les 3 commandes clés
```bash
# 1. Créer un Google Service Account
gcloud iam service-accounts create mon-app-sa
 
# 2. Donner les permissions GCP
gcloud projects add-iam-policy-binding mon-projet \
  --member="serviceAccount:mon-app-sa@mon-projet.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
 
# 3. Lier KSA et GSA
gcloud iam service-accounts add-iam-policy-binding mon-app-sa@mon-projet.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:mon-projet.svc.id.goog[default/mon-app-ksa]"
```
 
---
 
## 🧠 Ce qu'il faut retenir pour l'examen
 
- **Pod** = unité de base, **Deployment** = gère les pods, **Service** = expose les pods
- En cas de panne nœud → **Controller Manager** détecte, **Scheduler** replace les pods
- **Rolling Update** = stratégie par défaut, zéro interruption avec `maxUnavailable: 0`
- **`kubectl rollout undo`** = rollback immédiat en production
- **HPA** scale les pods, **Cluster Autoscaler** scale les nœuds
- **Workload Identity** = jamais de clés JSON → lier KSA à GSA
- Toujours créer le cluster en **europe-west1** dans la sandbox entreprise
