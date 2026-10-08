# Face Auth — Authentification faciale multi-facteurs

<p align="center">
  <strong>Une plateforme de preuve de concept pour l'enrôlement, la reconnaissance faciale et la vérification par défis de pose.</strong>
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white">
  <img alt="OpenCV" src="https://img.shields.io/badge/OpenCV-vision-5C3EE8?logo=opencv&logoColor=white">
  <img alt="FastAPI" src="https://img.shields.io/badge/FastAPI-API-009688?logo=fastapi&logoColor=white">
  <img alt="AWS" src="https://img.shields.io/badge/AWS-S3%20%7C%20DynamoDB-FF9900?logo=amazonaws&logoColor=white">
</p>

## À propos

**Face Auth** combine détection faciale, descripteurs biométriques dlib et estimation
de l'orientation de la tête pour construire un parcours d'authentification vivant :

1. l'utilisateur est enrôlé avec cinq familles de poses ;
2. les images sont transformées en encodage facial et associées à des angles
   `yaw`, `pitch` et `roll` ;
3. l'identité est retrouvée par plus proche voisin ;
4. plusieurs défis de pose aléatoires doivent être réalisés dans un temps limité ;
5. l'interface graphique ou l'API suit l'avancement en temps réel.

Le projet contient également une couche expérimentale d'intégration AWS : suivi des
tâches dans DynamoDB, stockage S3 et génération de QR codes pour un second facteur.

> **Statut : prototype/PFE.** Le service de tâches est exécuté dans des threads et
> le code attend actuellement une caméra locale. Une revue de sécurité et une
> adaptation à la production sont indispensables avant tout usage réel.

## Fonctionnalités

- **Enrôlement guidé** : capture automatique lorsque le visage est détecté et stable.
- **Cinq poses** : `frontale`, `gauche`, `droite`, `haut` et `bas`.
- **Reconnaissance faciale** : descripteur dlib de 128 dimensions et seuil de
  distance configuré à `0.6`.
- **Détection de vivacité légère** : défis de pose maintenus pendant environ
  trois secondes.
- **Estimation de pose 3D** : calcul de `yaw`, `pitch` et `roll` avec OpenCV
  `solvePnP`.
- **Cache intelligent** : les encodages sont recalculés uniquement lorsque le
  contenu du dossier `dataset/` change.
- **Interface PyQt6** : aperçu caméra, progression et diagnostic utilisateur.
- **API FastAPI headless** : lancement de captures/authentifications en arrière-plan
  et interrogation de leur état.
- **Observabilité de démonstration** : persistance des états de tâche dans DynamoDB.
- **Protection expérimentale** : double chiffrement AES-CBC, HMAC-SHA-512 et
  dépôt S3 pour les données migrées.

## Architecture

```text
                    +----------------------+
                    |  Interface PyQt6     |
                    |  face_pose_estimation|
                    +----------+-----------+
                               |
                    +----------v-----------+
                    |  face_service.py    |
                    | détection / encodage |
                    | poses / authentif.   |
                    +----+------------+----+
                         |            |
                +--------v--+     +---v---------+
                | dataset/  |     | cache .pkl  |
                | images    |     | hash dataset|
                +-----------+     +-------------+

 Client HTTP -> FastAPI -> ThreadPoolExecutor -> face_service
                         |
                         +--> DynamoDB (état des tâches)
                         +--> S3 (migration chiffrée / QR)
```

## Arborescence

| Fichier | Rôle |
| --- | --- |
| `face_pose_estimation.py` | Application graphique PyQt6 et flux d'enrôlement/authentification |
| `face_service.py` | Logique métier réutilisable sans interface graphique |
| `api_server.py` | API FastAPI et exécution des tâches en arrière-plan |
| `pose_diagnostic.py` | Affichage des poses apprises et monitoring temps réel |
| `cache_manager.py` | Inspection, reconstruction et suppression du cache |
| `security_utils.py` | Chiffrement, signature HMAC, S3 et génération de QR code |
| `main.tf` | Ressources AWS Terraform de démonstration |
| `requirements.txt` | Dépendances de l'application graphique et du service facial |
| `requirements_api.txt` | Dépendances supplémentaires de l'API |
| `shape_predictor_68_face_landmarks.dat.bz2` | Archive locale du modèle de landmarks |

## Pré-requis

- Windows 10/11 (ou environnement compatible OpenCV/dlib).
- Python 3.10 ou plus récent.
- Une webcam accessible par OpenCV.
- Les deux modèles dlib suivants, placés à la racine du projet :
  - `shape_predictor_68_face_landmarks.dat`
  - `dlib_face_recognition_resnet_model_v1.dat`
- Pour le cloud : un compte AWS, des identifiants configurés et les permissions
  DynamoDB/S3 nécessaires.

Les modèles dlib ne sont pas des dépendances pip. L'archive
`shape_predictor_68_face_landmarks.dat.bz2` doit être décompressée ; le modèle de
reconnaissance faciale doit être téléchargé séparément depuis la distribution
officielle dlib, puis décompressé. Ne pas publier ces fichiers dans un dépôt public.

## Installation Windows

Depuis PowerShell :

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install -r requirements_api.txt
python -m pip install boto3 pycryptodome qrcode[pil]
```

Si PowerShell bloque l'activation :

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

Vérifier ensuite la présence des modèles :

```powershell
Test-Path .\shape_predictor_68_face_landmarks.dat
Test-Path .\dlib_face_recognition_resnet_model_v1.dat
```

## Démarrage rapide

### Interface graphique

```powershell
python .\face_pose_estimation.py
```

Autorisez l'accès à la webcam, puis utilisez l'interface pour charger les données,
enregistrer un utilisateur et lancer une authentification.

### API FastAPI

```powershell
python -m uvicorn api_server:app --host 127.0.0.1 --port 8000 --reload
```

Documentation interactive :

- Swagger UI : <http://127.0.0.1:8000/docs>
- ReDoc : <http://127.0.0.1:8000/redoc>
- Santé du service : <http://127.0.0.1:8000/health>

> L'API ouvre la caméra sur la machine qui exécute le serveur. Elle n'accède pas
> automatiquement à la webcam du navigateur ou du client distant.

## Utiliser l'API

### Vérifier le service

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

Réponse :

```json
{"status":"ok"}
```

### Enrôler un utilisateur

```powershell
$capture = Invoke-RestMethod `
  -Method Post `
  "http://127.0.0.1:8000/capture?user_name=alice&num_images_per_pose=40"

$capture
Invoke-RestMethod "http://127.0.0.1:8000/tasks/$($capture.task_id)"
```

La capture produit des fichiers semblables à :

```text
dataset/
└── alice/
    ├── alice_frontale_000.jpg
    ├── alice_gauche_000.jpg
    ├── alice_droite_000.jpg
    ├── alice_haut_000.jpg
    └── alice_bas_000.jpg
```

### Authentifier un utilisateur

```powershell
$auth = Invoke-RestMethod `
  -Method Post `
  "http://127.0.0.1:8000/authenticate?total_challenges=5&challenge_timeout=12"

Invoke-RestMethod "http://127.0.0.1:8000/tasks/$($auth.task_id)"
```

### Lister le dataset chargé

```powershell
Invoke-RestMethod http://127.0.0.1:8000/dataset
```

Les tâches sont asynchrones : conservez le `task_id` renvoyé par `POST /capture`
ou `POST /authenticate`, puis interrogez `GET /tasks/{task_id}` jusqu'à obtenir
`status: "finished"` ou `status: "error"`. La lecture de
`GET /tasks/{task_id}` dépend toutefois de la table DynamoDB configurée dans
`api_server.py` ; pour un mode strictement local, il faut remplacer cette lecture
cloud par le dictionnaire `TASKS`.

## Gestion du cache et diagnostic

Le cache accélère le démarrage en conservant les encodages et les poses moyennes
dans `face_encodings_cache.pkl`.

```powershell
python .\cache_manager.py status
python .\cache_manager.py rebuild
python .\cache_manager.py clear
```

Pour inspecter les angles appris et les tolérances utilisées pendant la validation :

```powershell
python .\pose_diagnostic.py
```

Le mode interactif permet de changer de pose avec `f` (frontale), `g` (gauche),
`d` (droite), `u` (haut), `b` (bas) et de quitter avec `q`.

## AWS et Terraform

`main.tf` décrit deux ressources de démonstration dans `us-east-1` :

- une table DynamoDB `FaceAuthTasks`, partitionnée par `task_id` ;
- un bucket S3 destiné au stockage du dataset.

Avant tout déploiement, remplacez le nom de bucket d'exemple dans `main.tf` et
dans les constantes applicatives. Un nom S3 doit être unique mondialement.

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

L'application utilise actuellement les noms `FaceAuthTasks` et
`pfe-face-auth-storage-votre-nom`/`ton-nom-de-bucket-ici` à plusieurs endroits.
Ils doivent être harmonisés avant une exécution cloud. Configurez les identifiants
AWS via le mécanisme officiel AWS CLI ou les variables d'environnement ; ne les
écrivez jamais dans le code.

## Sécurité et limites connues

Cette implémentation est destinée à la démonstration et non à la production :

- la clé maître est codée en dur dans `security_utils.py` : elle doit être remplacée
  par un secret géré (par exemple AWS Secrets Manager ou KMS) ;
- AES-CBC est utilisé avec HMAC, mais une construction AEAD moderne
  (AES-GCM ou ChaCha20-Poly1305) serait préférable pour une nouvelle version ;
- certains appels AWS et certaines erreurs sont gérés de façon permissive ;
- l'API n'a ni authentification HTTP, ni autorisation, ni limitation de débit ;
- les tâches sont gardées en mémoire par l'API, même si leurs statuts sont écrits
  dans DynamoDB ;
- les données biométriques et les images doivent avoir une politique de rétention,
  un contrôle d'accès et un consentement explicites ;
- les modèles et la webcam doivent être protégés contre les accès non autorisés.

Ne déployez donc pas cette API sur Internet sans ajouter TLS, authentification,
validation stricte des entrées, journalisation maîtrisée, IAM least privilege et
chiffrement des secrets.

## Dépannage

| Symptôme | Vérification |
| --- | --- |
| `shape predictor not found` | Décompresser le fichier `.bz2` et placer le `.dat` à la racine |
| `face recognition model not found` | Télécharger et décompresser le second modèle dlib |
| `Cannot open camera` | Tester `camera_index=1` ou fermer une autre application utilisant la webcam |
| Aucun utilisateur reconnu | Vérifier `dataset/<nom>/`, reconstruire le cache et améliorer l'éclairage |
| `Task not found on AWS` | Vérifier les credentials AWS, la région et l'existence de `FaceAuthTasks` |
| Capture très lente | Utiliser `cache_manager.py rebuild` après modification du dataset |

## Feuille de route

- déplacer tous les paramètres (bucket, table, région, clé et seuils) dans une
  configuration validée ;
- remplacer le polling par WebSocket ou Server-Sent Events ;
- ajouter une authentification API et des tests automatisés ;
- isoler le stockage biométrique et chiffrer avec KMS/AEAD ;
- ajouter une vraie détection anti-spoofing (photo/vidéo) ;
- conteneuriser le service et remplacer l'exécuteur local par une file de tâches ;
- documenter une politique de consentement, de suppression et de conservation.

## Licence

Aucune licence open source n'est actuellement déclarée dans le dépôt. Ajoutez une
licence explicite avant toute redistribution.
