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
└── README.md                  # Ce fichier

