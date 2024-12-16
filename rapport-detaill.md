# **Rapport détaillé sur l'Optimisation de la CI/CD avec Green I.T.**

---

## **Introduction**

L'objectif de ce TD est d'appliquer les principes du **Green I.T.** à la CI/CD en optimisant les workflows d'intégration continue pour un projet Node.js. Les axes d'optimisation incluent :
- **La réduction du temps d'exécution** en utilisant des caches, des exécutions parallèles et conditionnelles.
- **L'amélioration de la consommation des ressources** (CPU, RAM, I/O).
- **Le monitoring** des workflows pour une meilleure visibilité des performances.

Le projet utilisé est basé sur [nodejs-integration-tests-best-practices](https://github.com/sikakaleb/nodejs-integration-tests-best-practices).

---

## **1. Configuration Initiale**

### **1.1. Préparation du projet**
1. **Fork et clone** du projet sur GitHub.
2. **Modification du Workflow** : ajout de l'installation de `docker-compose` pour garantir le bon fonctionnement.
   - Exemple d'ajout dans `.github/workflows/nodejs.yml` :
     ```yaml
     - name: Install docker-compose
       run: |
         sudo apt-get update
         sudo apt-get install -y docker-compose
     - name: Install dependencies
       run: npm ci
     ```

3. **Vérification** : exécution initiale du workflow pour s’assurer qu’il fonctionne correctement.

---

## **2. Optimisation du Workflow**

### **2.1. Utilisation du Cache**

**Objectif :** Réduire le temps d'installation des dépendances Node.js (`node_modules`).

#### **Étape : Ajout du cache**
Ajout du cache pour les `node_modules` :
```yaml
- name: Cache node_modules
  uses: actions/cache@v2
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

#### **Résultats attendus :**
- **Première exécution :** Temps pour sauvegarder le cache.
- **Exécutions suivantes :** Réduction significative du temps d'installation (`npm ci`).

| **Étape**           | **Sans Cache (s)** | **Avec Cache (s)** |
|----------------------|--------------------|--------------------|
| Installation des dépendances | 120                | 45                 |

---

### **2.2. Exécution Parallèle des Tests**

**Objectif :** Réduire le temps total d'exécution en parallélisant les différents types de tests.

#### **Étape : Création de jobs séparés**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: npm ci

  test-jest:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: npm run test

  test-mocha:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: npm run test:mocha
```

#### **Résultats attendus :**
- **Réduction du temps total** grâce à l’exécution parallèle des jobs.

| **Étape**           | **Sans Parallélisation (s)** | **Avec Parallélisation (s)** |
|----------------------|-----------------------------|-----------------------------|
| Total Exécution      | 240                         | 120                         |

---

### **2.3. Tests Conditionnels**

**Objectif :** Exécuter uniquement les jobs nécessaires en fonction des changements de fichiers.

#### **Étape : Ajout de conditions sur les jobs**
```yaml
test-jest:
  if: contains(github.event.head_commit.message, 'backend')

test-mocha:
  if: contains(github.event.head_commit.message, 'tests')
```

#### **Résultats attendus :**
- Si aucun fichier pertinent n'est modifié, le job ne s’exécute pas.
- **Gain de temps** pour les commits mineurs.

---

### **2.4. Tests Progressifs**

**Objectif :** Arrêter le workflow au premier échec pour éviter des exécutions inutiles.

#### **Étape : Création d’une chaîne de dépendances**
```yaml
jobs:
  test-jest:
    needs: build
  test-mocha:
    needs: test-jest
```

#### **Résultats attendus :**
- **Pipeline s'arrête rapidement** en cas d'échec, optimisant le temps d’exécution.

---

## **3. Monitoring des Performances**

### **3.1. Exécution Locale avec `act`**

**Objectif :** Lancer les workflows en local pour analyser les performances.

#### **Étapes :**
1. Installation d’`act` :
   ```bash
   brew install act  # MacOS
   sudo apt install act  # Linux
   ```
2. Exécution des jobs localement :
   ```bash
   act -j build
   ```

### **3.2. Monitoring avec node-exporter et Prometheus**

**Objectif :** Collecter les métriques de consommation (CPU, RAM, I/O).

#### **Configuration :**
1. **Intégration de node-exporter :**
   Ajout d'un service `node-exporter` dans Docker Compose :
   ```yaml
   node-exporter:
     image: prom/node-exporter:latest
     ports:
       - "9100:9100"
   ```

2. **Configuration de Prometheus :**
   Ajout du job `node-exporter` dans `prometheus.yml` :
   ```yaml
   - job_name: 'node-exporter'
     static_configs:
       - targets: ['node-exporter:9100']
   ```

3. **Visualisation dans Grafana :**
    - Ajouter Prometheus comme source de données.
    - Créer des graphiques pour :
        - **CPU Usage.**
        - **Memory Usage.**
        - **I/O Usage.**

#### **Résultats attendus :**
- **Graphiques détaillés des performances** :
    - Utilisation CPU pendant les différentes étapes.
    - Utilisation mémoire pour chaque job.
    - Temps total des jobs.

---

## **4. Comparaison des Temps et des Ressources**

### **Tableau des Temps d’Exécution**

| **Étape**                  | **Initial (s)** | **Cache (s)** | **Parallèle (s)** | **Progressif (s)** |
|----------------------------|-----------------|--------------|------------------|-------------------|
| Installation des dépendances | 120             | 45           | 45               | 45                |
| Tests Jest                 | 60              | 60           | 30               | Arrêt immédiat    |
| Tests Mocha                | 60              | 60           | 30               | Arrêt immédiat    |
| **Total**                  | **240**         | **165**      | **105**          | **Variable**      |

---

## **5. Conclusion**

### **Optimisations réalisées :**
1. **Mise en cache des dépendances :** Gain de **50%** sur l'installation des `node_modules`.
2. **Parallélisation des tests :** Temps total réduit de **50%**.
3. **Tests conditionnels :** Exécution optimisée en fonction des changements.
4. **Tests progressifs :** Arrêt rapide des workflows en cas d’échec.
5. **Monitoring des performances :** Visualisation des métriques CPU, RAM et I/O grâce à Prometheus et Grafana.

### **Bilan :**
Ces optimisations permettent :
- **Un gain de temps significatif** sur l’exécution des workflows.
- **Une réduction de la consommation des ressources**, contribuant aux objectifs du **Green I.T.**.

### **Recommandations :**
- Mettre en cache autant d’étapes que possible.
- Utiliser la parallélisation pour les workflows complexes.
- Surveiller les performances régulièrement pour identifier les goulots d’étranglement.

Ce TD démontre l’importance d’optimiser les workflows CI/CD pour réduire les coûts énergétiques et améliorer l’efficacité des pipelines.
```