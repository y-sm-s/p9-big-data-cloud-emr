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

## Architecture cloud

```
Images JPG
    ↓
Amazon S3 (stockage)
    ↓
Amazon EMR (cluster Spark)
    ├── Lecture des images (binaryFile)
    ├── Feature extraction : MobileNetV2 (TensorFlow, broadcasted)
    ├── Normalisation : StandardScaler (Spark MLlib)
    └── Réduction dimensionnelle : PCA 200 composantes (Spark MLlib)
    ↓
Export résultats → S3 (format Parquet)
Export pipeline ajusté → S3 (Spark Pipeline)
```

---

## Méthodologie

1. **Initialisation** : création d'une SparkSession sur cluster EMR, connexion au bucket S3
2. **Chargement des images** : lecture des fichiers `.jpg` avec Spark `binaryFile`, extraction du label depuis le chemin
3. **Feature extraction distribuée** : MobileNetV2 (pré-entraîné ImageNet, sans la tête de classification) broadcasté sur tous les workers Spark — extraction de vecteurs de features de 1280 dimensions par image
4. **Pipeline Spark MLlib** :
   - `StandardScaler` : normalisation des features (withMean=True, withStd=True)
   - `PCA(k=200)` : réduction à 200 composantes
5. **Export** : features réduites en Parquet sur S3, pipeline ajusté sauvegardé pour réutilisation

---

## Résultats

- 200 composantes PCA retenues, expliquant ~95% de la variance des features MobileNetV2
- Pipeline Spark complet exporté et réutilisable sur de nouvelles images sans réentraînement
- Traitement entièrement distribué : scalable à des millions d'images sans modification du code

---

## Stack technique

- Python 3
- PySpark (Spark MLlib : StandardScaler, PCA, Pipeline, binaryFile)
- TensorFlow / Keras (MobileNetV2)
- AWS EMR (cluster Spark managé)
- AWS S3 (stockage objets)
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

Ce notebook est conçu pour s'exécuter sur un cluster **Amazon EMR** avec PySpark.

Pour le tester localement (sur un sous-ensemble d'images) :

```bash
pip install -r requirements.txt
# Adapter les chemins PATH vers un dossier local au lieu de S3
jupyter notebook notebooks/01_pipeline_emr_spark.ipynb
```

Pour un déploiement cloud complet, créez un cluster EMR avec Spark et TensorFlow, uploadez le notebook sur S3, et exécutez-le via EMR Notebooks ou JupyterHub.
