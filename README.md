# IA-86 (nom provisoire)

Étude de l'architecture **IA-64** (Itanium d'Intel et HP) et conception, à terme, d'un nouveau jeu d'instructions qui reprend ses bonnes idées et corrige ses défauts.

## Pourquoi IA-64 ?
IA-64 repose sur le principe **EPIC** (*Explicitly Parallel Instruction Computing*) : c'est le compilateur, et non le processeur, qui décide à l'avance quelles instructions s'exécutent en parallèle. L'idée était ambitieuse et l'échec commercial a été net (Itanium arrêté en 2021, retiré de Linux et de GCC ensuite). Ce contraste en fait un excellent cas d'étude : beaucoup d'idées fortes (prédication, spéculation, registres tournants), et des défauts bien documentés.

## Objectifs
1. **Comprendre** l'architecture à partir du manuel officiel Intel.
2. **Analyser** son comportement et ses points critiques, en confrontant le manuel aux critiques (articles, forums, retours d'expérience).
3. **Expérimenter** avec un simulateur d'un sous-ensemble d'IA-64.
4. **Refaire la base** : spécifier un nouveau jeu d'instructions, puis écrire son assembleur et son simulateur.

C'est un projet au long cours (plusieurs mois, voire années). Chaque étape est documentée en français, avec ses sources.

## Organisation
| Dossier | Contenu |
|---|---|
| `documentation/fiches/` | Fiches d'étude du Volume 1 (Parties 1 et 2), exemples d'assembleur et encodages binaires (`01` à `10`) |
| `documentation/recherches/` | Notes de recherche : critiques, vérifications et synthèse des choix pour IA-86 |
| `documentation/intel/` | Manuel officiel Intel, **non versionné** (voir ci-dessous) |

## Manuel de référence
*Intel® Itanium® Architecture Software Developer's Manual*, révision 2.3 (mai 2010), 4 volumes :

| Volume | Contenu | Pages |
|---|---|---|
| 1 | Architecture applicative | 244 |
| 2 | Architecture système | 676 |
| 3 | Jeu d'instructions | 420 |
| 4 | Instructions IA-32 | 604 |

Ces documents appartiennent à Intel et ne sont pas redistribués ici. Ils se téléchargent sur intel.com, par exemple le [volume 3](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/itanium-architecture-vol-3-manual.pdf) ou le [recueil des 4 volumes](https://www.intel.com/content/dam/doc/manual/itanium-architecture-vol-1-2-3-4-reference-set-manual.pdf).

## Avancement
- [x] Manuel récupéré
- [x] Première recherche sur les critiques d'IA-64
- [x] Fiches d'étude du volume 1 et programmes d'assembleur encodés à la main (`documentation/fiches/01` à `10`)
- [x] Analyse des points critiques, verdict par point et cahier des charges d'IA-86 (`documentation/recherches/synthese-critique-et-choix-ia86.md`)
- [ ] Simulateur d'un sous-ensemble d'IA-64
- [ ] Spécification du nouveau jeu d'instructions (IA-86)

## Licence
Code et documentation du projet : [Apache 2.0](LICENSE). Itanium et Intel sont des marques d'Intel Corporation ; ce projet n'est pas affilié à Intel.

Auteur : Théo Gastinel.
