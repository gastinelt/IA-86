# Plan d'action (à grosse louche)

Version du 20/10/2026. Durées indicatives, au rythme « gros du travail le week-end, petites tâches en semaine ».

| Phase | But | Livrables | Durée |
|---|---|---|---|
| **1. Comprendre** | Maîtriser IA-64 à partir du manuel | Fiches d'étude en français (bundles, groupes et stops, prédication, spéculation, registres tournants, RSE, mémoire), glossaire | 3 à 5 semaines |
| **2. Analyser** | Juger chaque mécanisme | Un document par mécanisme : ce qu'il apporte, ce qu'il coûte, ce qu'en disent les critiques, notre verdict (garder / simplifier / jeter) | 3 à 4 semaines |
| **3. Simulateur IA-64** | Voir le comportement réel | Simulateur d'un sous-ensemble (~30 instructions), décodeur de bundles, petits programmes de test | 2 à 3 mois |
| **4. Notre jeu d'instructions** | Refaire la base | Spécification v0.1, puis assembleur et simulateur | 3 à 6 mois et plus |
| **5. Plus tard** | Aller plus loin | Petit compilateur, comparaison avec IA-64, peut-être un FPGA | à voir |

## Phase 1 : comprendre
1. Lire le vol. 1, Partie 1, chap. 2 (vue d'ensemble), puis la Partie 2, chap. 2.
2. Une fiche par mécanisme, chaque fait sourcé (volume, page).
3. Contrôles visuels en attente : p. 3:323 (M1/M2), stops de la Table 3-10, EC p. 1:76-77.
4. Écrire à la main 2 ou 3 petits programmes en assembleur IA-64, pour s'entraîner.

## Phase 2 : analyser
1. Compléter les critiques : Knuth, Raymond Chen, Real World Tech, processeur Mill, autres VLIW (DSP, Elbrus, Transmeta).
2. Pour chaque mécanisme : coût, gain, critiques, verdict. Ce sont les décisions du projet.
3. Choisir le langage du simulateur (proposition : C, simple et proche de la machine ; ou Rust).

## Phase 3 : simulateur IA-64
1. Décodeur de bundles (template, 3 slots) et désassembleur, testés sur l'exemple déjà encodé.
2. Exécution : entiers, comparaisons, prédicats, chargements et écritures, branchements.
3. Groupes d'instructions (lire avant d'écrire), détection des dépendances interdites.
4. `alloc`, `br.call`, `br.ret` (pile de registres simplifiée), puis registres tournants et `br.ctop`.
5. Les 3 programmes d'exemple tournent, avec une trace lisible pour présenter le code.
6. Mesures : nombre de `nop`, instructions par groupe, taille du code.

## Phase 4 : notre jeu d'instructions
1. Principes tirés de la phase 2 (taille des instructions, groupes et stops, nombre de prédicats, etc.).
2. Spécification v0.1 : registres, formats, encodage, liste des instructions.
3. Assembleur, puis simulateur (en réutilisant celui de la phase 3).
4. Comparaison avec IA-64 sur les mêmes programmes.
5. On choisit le vrai nom du projet.

## Règles tout du long
- Le manuel Intel est la référence ; tout ce que dit une IA est vérifié.
- Commit après chaque étape vérifiée, push lancé par Théo.
- Code et documents compréhensibles et présentables par Théo.
