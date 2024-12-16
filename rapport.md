Voici ce à quoi nous devons nous attendre pour chaque partie du TD concernant l'intégration des principes **Green I.T.** dans la CI/CD avec le projet **Node.js Integration Tests Best Practices**.

---

## **1. Optimisation du Workflow**

### **a. Cache des `node_modules`**

- **Objectif :** Réduire le temps d'exécution en mettant en cache les dépendances.
- **Procédure :**
    - Ajouter une étape pour utiliser et sauvegarder le cache :
      ```yaml
      - name: Cache node_modules
        uses: actions/cache@v2
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      ```
    - Comparer :
        1. Le temps d'exécution initial sans cache.
        2. Le temps après l'activation du cache.

- **Résultat attendu :**
    - Une réduction significative du temps pour `npm ci`, grâce au cache.

---

### **b. Exécution parallèle des tests**

- **Objectif :** Réduire le temps global en exécutant les tests en parallèle.
- **Modification du Workflow :**
    - Créer des jobs séparés pour chaque type de test :
      ```yaml
      jobs:
        build:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v2
            - name: Install dependencies
              run: npm ci
 
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

- **Résultat attendu :**
    - **Temps total d'exécution réduit**, car les tests sont parallélisés.
    - Le temps pour chaque job sera mesuré et comparé.

---

### **c. Tests conditionnels**

- **Objectif :** Exécuter uniquement les jobs nécessaires en fonction des changements de fichiers.
- **Ajout d'une condition pour chaque job :**
   ```yaml
   test-jest:
     if: contains(github.event.head_commit.message, 'backend')
   ```

- **Résultat attendu :**
    - Les jobs inutiles ne s’exécutent pas si aucun fichier pertinent n’a été modifié.
    - Réduction des temps d’exécution pour les commits mineurs.

---

### **d. Tests progressifs**

- **Objectif :** Arrêter la pipeline au premier échec.
- **Modification du workflow :**
    - Faire dépendre chaque job du précédent :
      ```yaml
      test-jest:
        needs: build
      test-mocha:
        needs: test-jest
      ```

- **Résultat attendu :**
    - La pipeline s'arrête plus rapidement en cas d’échec, réduisant le temps de traitement.

---

## **2. Monitoring**

### **Objectif :**
- Surveiller la consommation de ressources (CPU, RAM, etc.) pendant l’exécution des workflows.

### **Procédure :**
1. **Exécution locale :**
    - Utilisation de `act` pour lancer les workflows localement :
      ```bash
      act -j build
      ```
2. **Mesure des performances :**
    - Intégrer **node-exporter** pour collecter les métriques.
    - Ajouter un scraper **Prometheus**.
    - Visualiser les résultats dans **Grafana**.

### **Résultat attendu :**
- **Visualisation des métriques :**
    - Graphiques de l'utilisation CPU, RAM et temps d'exécution des jobs.
    - Identification des étapes les plus coûteuses en ressources.

---

## **3. Comparaison des Temps et des Ressources**

- Créez un tableau comparatif avant et après optimisation :

| **Étape**                  | **Temps Initial (s)** | **Après Cache** | **Parallèle** | **Progressif** |
|----------------------------|------------------------|-----------------|---------------|----------------|
| Installation des dépendances | 120                    | 45              | 45            | 45             |
| Tests Jest                 | 60                     | 60              | 30            | Arrêt immédiat |
| Tests Mocha                | 60                     | 60              | 30            | Arrêt immédiat |
| **Total**                  | 240                    | 165             | 105           | **Variable**   |

- **Conclusion attendue :**
    - Temps global réduit grâce à la mise en cache et l’exécution parallèle.
    - Le mode progressif arrête la pipeline en cas d’erreur, optimisant encore le temps.

---

## **4. Monitoring en Local avec `act`**

- **Avantage :**
    - Simulation locale des workflows sans coût.
    - Intégration d’outils de monitoring pour collecter des métriques.

- **Procédure :**
    1. Configurer `act` pour exécuter les jobs :
       ```bash
       act -j build
       ```
    2. Lancer **node-exporter** et **Prometheus** pour collecter les données.
    3. Visualiser dans **Grafana**.

- **Résultat attendu :**
    - Mesure des performances (CPU, RAM, I/O) des jobs locaux.
    - Identification des points d'amélioration pour optimiser la consommation.

---

## **Conclusion Générale**

Ce TD met en évidence plusieurs techniques d'optimisation CI/CD :
1. **Cache des dépendances** pour des exécutions plus rapides.
2. **Parallélisation des tests** pour maximiser l’efficacité.
3. **Tests conditionnels et progressifs** pour réduire le temps global.
4. **Monitoring des workflows** pour analyser et réduire la consommation de ressources.

Les résultats finaux sont attendus sous forme de **temps optimisés** et de **visualisations des performances** via Grafana.