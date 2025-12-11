# Projet : Comparaison mono-thread / multi-thread pour la multiplication de matrices

## 1. Objectif pédagogique

Ce projet a pour but d’illustrer, sur un cas concret de calcul scientifique, les différences entre :

- une exécution **mono-thread** (séquentielle),
- une exécution **multi-thread** (parallèle, avec `pthreads`).

Les objectifs (alignés avec le cahier des charges) sont :

- **Mesurer les performances** : temps d’exécution, GFLOP/s, speedup.
- **Évaluer la réactivité et la montée en charge** : impact de la taille des matrices et du nombre de threads.
- **Observer l’utilisation du matériel** : exploitation des cœurs CPU de la VM via le multi-threading.

Le cas d’étude choisi est la **multiplication de matrices carrées denses** `C = A × B` avec un algorithme naïf en `O(n³)`.

---

## 2. Organisation du dépôt

Arborescence principale :

```text
projet-matmul/
├── src/
│   └── matmul_pthreads.c      # Implémentation C : mono-thread + multi-thread (pthreads)
├── scripts/
│   ├── run_bench.sh           # Script Bash : lance les séries de mesures
│   └── analyse_results.py     # Script Python : analyse les mesures et génère les figures
├── results/
│   ├── mesures.csv            # Fichier CSV des mesures brutes
│   └── figures/               # Figures générées automatiquement
│       ├── gflops_parallel_n500.png
│       ├── gflops_parallel_n800.png
│       ├── gflops_parallel_n1000.png
│       ├── gflops_parallel_n1500.png
│       ├── gflops_parallel_n2000.png
│       ├── speedup_n500.png
│       ├── speedup_n800.png
│       ├── speedup_n1000.png
│       ├── speedup_n1500.png
│       ├── speedup_n2000.png
│       ├── time_parallel_threads1.png
│       ├── time_parallel_threads2.png
│       ├── time_parallel_threads4.png
│       ├── time_parallel_threads8.png
│       └── time_serial_vs_best_parallel.png
└── README.md


   Résumé des fichiers :

src/matmul_pthreads.c
Contient le programme C principal :

version mono-thread : matmul_serial ;

version multi-thread : matmul_parallel basée sur pthread.

scripts/run_bench.sh
Script Bash qui :

exécute le binaire pour plusieurs tailles de matrices (n = 500, 800, 1000, 1500, 2000) ;

teste différents nombres de threads (1, 2, 4, 8) ;

récupère les lignes « Serial time », « Parallel time » et « Speedup » ;

écrit les mesures dans results/mesures.csv au format :

n,threads,time_serial_s,time_parallel_s,speedup,gflops_serial,gflops_parallel


scripts/analyse_results.py
Script Python qui :

lit results/mesures.csv ;

calcule éventuellement des métriques dérivées ;

génère automatiquement les figures dans results/figures/ :

GFLOP/s en fonction du nombre de threads,

speedup en fonction du nombre de threads,

temps d’exécution parallèle en fonction de n pour un nombre de threads fixé,

comparaison mono vs meilleur multi.

results/mesures.csv
Fichier de résultats bruts. Peut être regénéré à tout moment en relançant run_bench.sh.

results/figures/
Figures prêtes à être intégrées dans le rapport ou la présentation.

3. Description du code C (src/matmul_pthreads.c)
3.1. Représentation des matrices

Les matrices sont carrées de taille n × n et stockées en row-major dans un tableau 1D de double :

M[i, j]  ↦  M[i * n + j]


Ainsi :

la ligne i commence à l’indice i * n ;

l’élément (i, j) se trouve à l’indice i * n + j.

Ce choix est cohérent avec la manière dont C stocke les tableaux, et il facilite les boucles imbriquées i / j / k de la multiplication.

Deux fonctions utilitaires sont fournies :

fill_random(...)
Remplit une matrice avec des valeurs pseudo-aléatoires :

void fill_random(int n, double *M) {
    for (int i = 0; i < n * n; i++) {
        M[i] = (double)rand() / RAND_MAX * 100.0;
    }
}


zero_matrix(...)
Initialise une matrice à zéro (utilise memset) :

void zero_matrix(int n, double *M) {
    memset(M, 0, n * n * sizeof(double));
}


Une fonction elapsed_seconds(...) encapsule le calcul de durée à partir de deux struct timespec :

static inline double elapsed_seconds(struct timespec start,
                                     struct timespec end) {
    return (end.tv_sec - start.tv_sec)
           + (end.tv_nsec - start.tv_nsec) / 1e9;
}

3.2. Structure de données pour les threads

Pour passer proprement les paramètres aux threads, on utilise une structure :

typedef struct {
    int n;                 // Taille de la matrice (n x n)
    const double *A;       // Pointeur sur la matrice A (en lecture seule)
    const double *B;       // Pointeur sur la matrice B (en lecture seule)
    double *C;             // Pointeur sur la matrice résultat C (en écriture)
    int thread_id;         // Identifiant du thread [0 .. nthreads-1]
    int nthreads;          // Nombre total de threads
    int row_start;         // Première ligne de C à calculer (incluse)
    int row_end;           // Dernière ligne de C à calculer (exclue)
    double elapsed_sec;    // Temps passé par ce thread (optionnel)
} ThreadData;


Chaque thread connaît :

la taille globale n ;

les pointeurs sur A, B, C (partagés entre tous les threads) ;

la plage de lignes de C qu’il doit calculer ([row_start, row_end)) ;

son identifiant thread_id et le nombre total de threads nthreads.

Cette struct permet de rester dans une approche multi-thread classique, sans utiliser de variables globales.

3.3. Version séquentielle : matmul_serial

La fonction séquentielle sert à la fois de référence de correction et de baseline pour les mesures :

void matmul_serial(int n, const double *A, const double *B, double *C) {
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            double sum = 0.0;
            for (int k = 0; k < n; k++) {
                sum += A[i * n + k] * B[k * n + j];
            }
            C[i * n + j] = sum;
        }
    }
}


Caractéristiques :

algorithme naïf en O(n³) (triple boucle) ;

accès mémoire simple, directement sur les indices i * n + j ;

sert de point de comparaison pour le speedup : T_serial / T_parallel.

3.4. Version parallèle : matmul_parallel (pthreads)
3.4.1. Fonction exécutée par chaque thread

Chaque thread exécute la fonction matmul_worker, qui calcule uniquement les lignes qui lui sont attribuées :

void* matmul_worker(void *arg) {
    ThreadData *data = (ThreadData*) arg;
    int n = data->n;
    const double *A = data->A;
    const double *B = data->B;
    double *C = data->C;

    int row_start = data->row_start;
    int row_end   = data->row_end;

    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);

    for (int i = row_start; i < row_end; i++) {
        for (int j = 0; j < n; j++) {
            double sum = 0.0;
            for (int k = 0; k < n; k++) {
                sum += A[i * n + k] * B[k * n + j];
            }
            C[i * n + j] = sum;
        }
    }

    clock_gettime(CLOCK_MONOTONIC, &t1);
    data->elapsed_sec = elapsed_seconds(t0, t1);

    pthread_exit(NULL);
}


Remarque importante :

chaque thread écrit dans des lignes distinctes de C ;

il n’y a donc aucune zone critique partagée en écriture → pas besoin de mutex ni de sémaphores.

3.4.2. Création et synchronisation des threads

La fonction matmul_parallel se charge de :

allouer les tableaux de pthread_t et de ThreadData ;

découper les lignes de C entre les threads (répartition homogène) ;

créer les threads avec pthread_create ;

synchroniser la fin du calcul avec pthread_join.

Extrait simplifié :

void matmul_parallel(int n,
                     const double *A,
                     const double *B,
                     double *C,
                     int nthreads)
{
    pthread_t  *threads     = malloc(sizeof(pthread_t)  * nthreads);
    ThreadData *thread_data = malloc(sizeof(ThreadData) * nthreads);

    int rows_per_thread  = n / nthreads;
    int remaining_rows   = n % nthreads;
    int current_row = 0;

    // Préparation des ThreadData et création des threads
    for (int t = 0; t < nthreads; t++) {
        int rows = rows_per_thread + (t < remaining_rows ? 1 : 0);

        thread_data[t].n         = n;
        thread_data[t].A         = A;
        thread_data[t].B         = B;
        thread_data[t].C         = C;
        thread_data[t].thread_id = t;
        thread_data[t].nthreads  = nthreads;
        thread_data[t].row_start = current_row;
        thread_data[t].row_end   = current_row + rows;
        thread_data[t].elapsed_sec = 0.0;

        current_row += rows;

        pthread_create(&threads[t], NULL,
                       matmul_worker, &thread_data[t]);
    }

    // Attente de la fin de tous les threads
    for (int t = 0; t < nthreads; t++) {
        pthread_join(threads[t], NULL);
    }

    free(threads);
    free(thread_data);
}


Cette fonction est appelée entre deux clock_gettime dans le main pour mesurer le temps global parallèle (T_parallel).

3.5. Fonction main et calcul des métriques

La fonction main :

lit les paramètres en ligne de commande : N (taille de la matrice), NTHREADS (nombre de threads) ;

alloue dynamiquement les matrices A, B, C_serial, C_parallel ;

initialise A et B par des valeurs aléatoires ;

exécute la version mono-thread et mesure t_serial ;

exécute la version multi-thread et mesure t_parallel ;

vérifie que les résultats sont cohérents (comparaison des matrices) ;

calcule les métriques de performance.

Extraits clés :

// 2 * n^3 FLOPs : n^2 éléments, chacun somme de n produits
double ops             = 2.0 * (double)n * n * n;
double gflops_serial   = ops / (t_serial   * 1e9);
double gflops_parallel = ops / (t_parallel * 1e9);

printf("Serial   time: %.3f s, %.2f GFLOP/s\n",
       t_serial,   gflops_serial);
printf("Parallel time: %.3f s, %.2f GFLOP/s\n",
       t_parallel, gflops_parallel);
printf("Speedup (serial/parallel): %.2f x\n",
       t_serial / t_parallel);


Ces lignes sont ensuite parsées par run_bench.sh pour remplir le CSV.

4. Scripts d’expérimentation
4.1. scripts/run_bench.sh

Ce script automatise les campagnes de tests. Exemple de logique (simplifiée) :

#!/usr/bin/env bash

EXE=../src/matmul_pthreads
OUT=../results/mesures.csv

echo "n,threads,time_serial_s,time_parallel_s,speedup,gflops_serial,gflops_parallel" > "$OUT"

for n in 500 800 1000 1500 2000; do
  for t in 1 2 4 8; do
    echo "n=$n, threads=$t"
    TMP=$(mktemp)
    "$EXE" "$n" "$t" > "$TMP"

    TS=$(grep "Serial   time"   "$TMP" | awk '{print $3}')
    GS=$(grep "Serial   time"   "$TMP" | awk '{print $5}')
    TP=$(grep "Parallel time"   "$TMP" | awk '{print $3}')
    GP=$(grep "Parallel time"   "$TMP" | awk '{print $5}')
    SP=$(grep "Speedup"         "$TMP" | awk '{print $3}')

    echo "$n,$t,$TS,$TP,$SP,$GS,$GP" >> "$OUT"
    rm "$TMP"
  done
done

4.2. scripts/analyse_results.py

Ce script :

lit results/mesures.csv (via pandas ou csv + matplotlib) ;

pour chaque n, trace :

GFLOP/s parallèle en fonction du nombre de threads ;

speedup en fonction du nombre de threads ;

pour chaque nombre de threads, trace :

temps parallèle en fonction de n ;

trace la comparaison mono-thread vs meilleur temps multi-thread.

Les figures produites sont nommées de manière explicite, par exemple :

gflops_parallel_n1000.png

speedup_n1500.png

time_parallel_threads8.png

time_serial_vs_best_parallel.png

5. Reproduire les expériences
5.1. Compilation

Depuis le répertoire src/ :

gcc -O3 -march=native -pthread matmul_pthreads.c -o matmul_pthreads

5.2. Exécution simple

Exemple pour une matrice 1500 × 1500 avec 8 threads :

./matmul_pthreads 1500 8


La sortie affiche notamment :

Serial   time: ...
Parallel time: ...
Speedup (serial/parallel): ...

5.3. Lancer la campagne de benchmarks

Depuis le répertoire scripts/ :

./run_bench.sh


Cela crée/écrase ../results/mesures.csv.

5.4. Générer les figures

Toujours dans scripts/ :

python3 analyse_results.py


Les figures sont enregistrées dans ../results/figures/.

6. Interprétation rapide des résultats

Pour n = 500, les temps sont très courts → l’overhead des threads et le bruit de mesure rendent le speedup irrégulier.

Pour n = 800 à 1500, les courbes GFLOP/s et speedup deviennent plus régulières : le programme profite bien des 8 cœurs.

Pour n = 2000, on obtient un speedup proche d’un ordre de grandeur avec 8 threads, mais les GFLOP/s par thread sont limités par la bande passante mémoire.

Ce projet illustre concrètement :

l’intérêt du multi-threading pour les calculs lourds ;

les limites liées à la part séquentielle (loi d’Amdahl) et à la hiérarchie mémoire ;

l’importance de bien dimensionner la taille du problème pour que la parallélisation soit rentable.
::contentReference[oaicite:0]{index=0}

