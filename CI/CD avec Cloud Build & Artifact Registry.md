# Exercice 1 — CI/CD avec Cloud Build & Artifact Registry
 
## 🎯 Objectif
Créer un pipeline CI/CD qui build une image Docker et la pousse dans Artifact Registry.
 
---
 
## 🏗️ Architecture mise en place
 
```
Code source (Cloud Shell)
        ↓
   cloudbuild.yaml        ← définit le pipeline
        ↓
   Cloud Build            ← exécute le pipeline
        ↓
 Artifact Registry        ← stocke l'image Docker
```
 
---
 
## 📋 Étapes réalisées
 
### 1. Initialisation du projet
```bash
gcloud config set project sbx-31371-208ad60900fb   # sélectionner le projet
gcloud config list                                   # vérifier la config
```
 
### 2. Activer les APIs
```bash
gcloud services enable cloudbuild.googleapis.com artifactregistry.googleapis.com
```
> Sur GCP, chaque service est désactivé par défaut. Il faut les "allumer" avant de les utiliser.
 
### 3. Créer un dépôt Artifact Registry
```bash
gcloud artifacts repositories create mon-premier-repo \
  --repository-format=docker \
  --location=europe-west1 \
  --description="Mon premier repo Docker"
```
> Un dépôt privé en Europe pour stocker nos images Docker.
 
### 4. Créer l'application
**`main.py`**
```python
print("Hello depuis mon pipeline CI/CD GCP !")
```
 
**`Dockerfile`**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY main.py .
CMD ["python", "main.py"]
```
 
### 5. Créer le pipeline CI/CD
**`cloudbuild.yaml`**
```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'europe-west1-docker.pkg.dev/sbx-31371-208ad60900fb/mon-premier-repo/mon-app:v1', '.']
 
images:
  - 'europe-west1-docker.pkg.dev/sbx-31371-208ad60900fb/mon-premier-repo/mon-app:v1'
```
> `steps` → étapes du pipeline | `images` → pousse l'image dans Artifact Registry
 
### 6. Lancer le build
```bash
# Créer un bucket de logs en Europe (contrainte sandbox entreprise)
gsutil mb -l europe-west1 gs://cloudbuild-logs-sbx-31371-208ad60900fb
 
# Donner les permissions à Cloud Build
gsutil iam ch serviceAccount:905860256274-compute@developer.gserviceaccount.com:roles/storage.admin \
  gs://cloudbuild-logs-sbx-31371-208ad60900fb
 
gcloud projects add-iam-policy-binding sbx-31371-208ad60900fb \
  --member="serviceAccount:905860256274-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
 
# Lancer le build
gcloud builds submit \
  --config cloudbuild.yaml \
  --region=europe-west1 \
  --gcs-log-dir=gs://cloudbuild-logs-sbx-31371-208ad60900fb \
  --gcs-source-staging-dir=gs://cloudbuild-logs-sbx-31371-208ad60900fb/staging \
  gs://cloudbuild-logs-sbx-31371-208ad60900fb/source.zip
```
 
### 7. Vérifier l'image
```bash
gcloud artifacts docker images list \
  europe-west1-docker.pkg.dev/sbx-31371-208ad60900fb/mon-premier-repo
```
 
---
 
## ⚠️ Problèmes rencontrés & solutions
 
| Problème | Cause | Solution |
|---|---|---|
| `'us' violates constraint` | Sandbox interdit les ressources hors Europe | Créer un bucket de logs en `europe-west1` |
| `Permission denied` sur le bucket | Cloud Build n'a pas accès au bucket | Donner le rôle `storage.admin` au SA Cloud Build |
| `uploadArtifacts denied` | Cloud Build ne peut pas écrire dans Artifact Registry | Donner le rôle `artifactregistry.writer` au SA Cloud Build |
 
---
 
## 🧹 Nettoyage (obligatoire après chaque exercice)
```bash
gcloud artifacts docker images delete \
  europe-west1-docker.pkg.dev/sbx-31371-208ad60900fb/mon-premier-repo/mon-app:v1 --quiet
 
gcloud artifacts repositories delete mon-premier-repo \
  --location=europe-west1 --quiet
 
gcloud storage rm -r gs://cloudbuild-logs-sbx-31371-208ad60900fb
 
cd .. && rm -rf mon-app
```
 
---
 
## 💰 Coût estimé
| Service | Coût |
|---|---|
| Cloud Build (< 120 min/jour) | ✅ Gratuit |
| Artifact Registry (quelques MB) | ~0.001$ |
| Cloud Storage (logs) | ~0.00$ |
 
---
 
## 🧠 Ce qu'il faut retenir pour l'examen
 
- **Cloud Build** exécute des pipelines définis dans `cloudbuild.yaml`
- **Artifact Registry** remplace Container Registry (GCR) — c'est le service moderne pour stocker les images
- Les **Service Accounts** ont besoin de permissions explicites sur chaque ressource GCP
- Les **contraintes d'organisation** (`constraints/gcp.resourceLocations`) peuvent bloquer la création de ressources hors d'une région autorisée
- `--region=europe-west1` force Cloud Build à s'exécuter en Europe
