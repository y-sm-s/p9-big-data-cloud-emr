# Pipeline Big Data de classification d'images dans le cloud

Projet 9 — Formation Data Scientist, OpenClassrooms

---

## Contexte métier

Dans la continuité du Projet 6 (classification de biens de consommation), l'objectif est de mettre en place un pipeline de traitement d'images à l'échelle Big Data, déployé sur le cloud AWS. Le pipeline doit être capable de traiter un volume important d'images de fruits, extraire des features visuelles, et réduire leur dimensionnalité — le tout dans un environnement distribué.

---

## Données

Source : [Fruits 360 Dataset — Kaggle](https://www.kaggle.com/datasets/moltean/fruits)

Les images ne sont pas incluses dans ce dépôt. Le pipeline s'attend à trouver les images dans un bucket S3 (`s3://p9-fruits-project/data/Test/`).

---

## Approche en deux phases

### Phase 1 — Reproduction locale de l'architecture AWS avec Docker

Avant tout déploiement sur AWS, j'ai reproduit l'architecture cible entièrement en local à l'aide de **Docker**, pour simuler un cluster Spark multi-nœuds sur ma machine.

Cette décision était délibérément économique : exécuter toutes les itérations de développement et de debugging directement sur EMR aurait engendré des coûts significatifs (facturation au temps de cluster, transferts S3, erreurs de configuration coûteuses à corriger en live).

La phase locale a permis de :
- **Tester et stresser l'architecture** Spark sans frais cloud — volumes de données, partitionnement, gestion mémoire des workers
- **Résoudre tous les problèmes de configuration** liés à la mise en place d'un environnement Big Data : dépendances entre PySpark et TensorFlow, sérialisation du modèle MobileNetV2, broadcast des poids du réseau sur les workers, compatibilité des versions Spark/Java
- **Valider le pipeline de bout en bout** (lecture images → feature extraction → PCA → export Parquet) avant de toucher à l'infrastructure cloud
- **Réduire drastiquement les coûts AWS** : le cluster EMR n'a été démarré qu'une fois le pipeline stabilisé localement, limitant le temps de facturation au strict nécessaire

### Phase 2 — Déploiement sur Amazon EMR

Une fois le pipeline validé localement, le déploiement sur EMR s'est fait sans surprise grâce au travail effectué en phase 1. Les seuls ajustements ont porté sur les chemins (local → S3) et la configuration du cluster.

---

## Architecture

```
Phase 1 — Local (Docker)              Phase 2 — AWS Cloud
─────────────────────────             ─────────────────────────
Docker Spark cluster                  Amazon S3
  ├── Master node                       └── Images JPG
  └── Worker nodes                            ↓
        ↓                             Amazon EMR (Spark)
  Images locales                        ├── Lecture binaryFile (S3)
        ↓                               ├── Feature extraction MobileNetV2
  Pipeline identique                    │   (broadcasted sur workers)
  (test & troubleshooting)              ├── StandardScaler
                                        └── PCA 200 composantes
                                              ↓
                                      Export S3 (Parquet + Pipeline)
```

---

## Méthodologie

1. **Containerisation locale** : mise en place d'un cluster Spark via Docker Compose, reproduction des contraintes d'un environnement distribué (réseau entre nœuds, gestion des ressources, sérialisation)
2. **Troubleshooting Big Data** : résolution des incompatibilités PySpark/TensorFlow, optimisation de la stratégie de broadcast du modèle, gestion de la mémoire des workers
3. **Chargement des images** : lecture des fichiers `.jpg` avec Spark `binaryFile`, extraction du label depuis le chemin de fichier
4. **Feature extraction distribuée** : MobileNetV2 (pré-entraîné ImageNet, couche avant classification) broadcasté sur tous les workers — vecteurs de 1280 dimensions par image
5. **Pipeline Spark MLlib** :
   - `StandardScaler` : normalisation des features (withMean=True, withStd=True)
   - `PCA(k=200)` : réduction dimensionnelle
6. **Export** : features réduites en Parquet sur S3, pipeline ajusté sauvegardé pour réutilisation sur de nouvelles images

---

## Résultats

- 200 composantes PCA retenues, expliquant **~95% de la variance** des features MobileNetV2
- Pipeline Spark complet exporté et réutilisable sans réentraînement
- Traitement entièrement distribué : scalable à des millions d'images sans modification du code
- **Coûts AWS maîtrisés** grâce à la phase de validation locale : zéro surprise au déploiement

---

## Stack technique

- Python 3
- **Docker** (reproduction locale du cluster Spark)
- **PySpark** (Spark MLlib : StandardScaler, PCA, Pipeline, binaryFile)
- **TensorFlow / Keras** (MobileNetV2)
- **AWS EMR** (cluster Spark managé)
- **AWS S3** (stockage objets)
- pandas, numpy, Pillow, matplotlib

---

## Structure du projet

```
p9-big-data-cloud-emr/
├── notebooks/
│   └── 01_pipeline_emr_spark.ipynb   # Pipeline complet : lecture S3, feature extraction, PCA, export
├── reports/
│   └── presentation.pdf               # Présentation de soutenance
├── requirements.txt
└── README.md
```

---

## Lancer le notebook

**En local (Docker)** — reproduire l'architecture avant tout déploiement cloud :

```bash
# Adapter les chemins PATH vers un dossier local au lieu de S3
pip install -r requirements.txt
jupyter notebook notebooks/01_pipeline_emr_spark.ipynb
```

**Sur AWS EMR** — une fois le pipeline validé localement :

Créez un cluster EMR avec Spark et TensorFlow, uploadez le notebook sur S3, et exécutez-le via EMR Notebooks ou JupyterHub. Les chemins S3 sont déjà configurés dans le notebook.
