# IA-86 (nom provisoire)

Étude approfondie de l'architecture **IA-64** (*Itanium* d'Intel et HP, modèle **EPIC** — *Explicitly Parallel Instruction Computing*) à partir des manuels officiels d'Intel, suivie de la conception d'un nouveau jeu d'instructions (**IA-86**) qui conserve ses meilleures idées tout en corrigeant ses défauts structurels.

---

## 1. Pourquoi étudier IA-64 pour concevoir un nouveau langage machine ?

IA-64 repose sur un pari radical : confier au **compilateur**, et non au matériel d'exécution dans le désordre (*Out-of-Order*), la responsabilité d'exprimer explicitement quelles instructions sont indépendantes et peuvent s'exécuter en parallèle.

L'écart entre l'ambition intellectuelle d'Itanium et son échec industriel (abandon définitif en 2021, retrait de Linux et de GCC) en fait un cas d'étude exceptionnel :
- **Des idées architecturales très fortes** : prédication généralisée sans saut, comparaisons doubles `(p1, p2)`, pipelines logiciels de boucles sans prologue/épilogue séparé (`br.ctop`), modèle mémoire *acquire/release* (`.acq`/`.rel`) directement sur les instructions mémoire, extraction/dépôt de champs de bits (`extr`/`dep`/`shrp`).
- **Des défauts d'encodage et d'architecture mesurables bit à bit** : paquets rigides de 128 bits à 3 slots imposant 25 % à 33 % de `nop`, templates manquants (`M_IB_`), moteur de pile de registres asynchrone (RSE), télescopage entre arguments d'entrée (`r32+`) et registres généraux tournants, absence de multiplieur entier 64×64 dans l'ALU, tables `ALAT` et blocs de recouvrement hors ligne.

**Notre démarche :** ne jamais se contenter de rumeurs de forums. Chaque mécanisme est vérifié page par page dans la spécification officielle d'Intel (révision 2.3), testé par l'encodage manuel de programmes en binaire, puis tranché par un verdict technique documenté.

---

## 2. Feuille de route et avancement

| Phase | Objectif | État | Livrables principaux |
|---|---|---|---|
| **Phase 1 — Comprendre** | Étudier intégralement l'architecture applicative (Volume 1, Parties 1 et 2) et les formats d'instructions (Volume 3, Chap. 3 et 4), et encoder des programmes à la main. | 🟢 **Terminée** | 10 fiches d'étude sourcées ([`documentation/fiches/`](documentation/fiches/)) et 3 programmes encodés bit à bit |
| **Phase 2 — Analyser** | Confronter les critiques externes au manuel officiel, isoler les défauts internes de la spécification et fixer le cahier des charges d'IA-86. | 🟢 **Terminée** | Regard critique sur chaque fiche + [Synthèse globale et choix IA-86](documentation/recherches/synthese-critique-et-choix-ia86.md) |
| **Phase 3 — Expérimenter** | Écrire un simulateur d'un sous-ensemble d'IA-64 capable d'exécuter nos binaires encodés et de mesurer cycles, groupes, rotations et `nop`. | ⚪ **À venir** | Simulateur de référence IA-64 et suite de tests binaires |
| **Phase 4 — Refaire la base (IA-86)** | Spécifier le jeu d'instructions **IA-86**, puis réaliser son assembleur et son simulateur pour comparer ses métriques à celles d'IA-64. | ⚪ **À venir** | Spécification IA-86, assembleur et simulateur |

---

## 3. Fiches d'étude du manuel Intel (`documentation/fiches/`)

Chaque fiche résume les mécanismes exacts avec leurs références de volume et de page (`Volume:Page`), suivis d'un encadré **« Regard critique & limites architecturales (pour IA-86) »**.

| # | Fiche d'étude | Sections & pages du manuel Intel | Sujets couverts |
|---|---|---|---|
| **01** | [`01-etat-architectural-et-registres.md`](documentation/fiches/01-etat-architectural-et-registres.md) | Vol. 1 Chap. 1–3 (`1:1`–`1:37`) | Modèle EPIC, 128 `GR` (64 bits + bit `NaT`), 128 `FR` (82 bits), 64 `PR`, 8 `BR`, registres d'application `AR`, `IP`, `CFM`, `UM`, conventions d'appel. |
| **02** | [`02-bundles-templates-et-groupes.md`](documentation/fiches/02-bundles-templates-et-groupes.md) | Vol. 1 §3.3–3.4 (`1:38`–`1:40`), Vol. 3 (`3:25`, `3:293`) | Bundle de 128 bits (3 slots de 41 bits + template de 5 bits), table complète des 24 templates (`M`/`I`/`F`/`B`/`L`/`X`), groupes d'instructions, stops `;;` et exceptions RAW/WAW. |
| **03** | [`03-predication-et-comparaisons.md`](documentation/fiches/03-predication-et-comparaisons.md) | Vol. 1 §3.5 (`1:40`–`1:44`, `1:87`–`1:92`) | Prédicat de garde `(qp)`, les 10 relations `cmp`/`tbit`/`tnat`, les 6 types de comparaison (`normal`, `.unc`, `.and`, `.or`, `.or.andcm`, `.and.orcm`). |
| **04** | [`04-pile-de-registres-et-rotation.md`](documentation/fiches/04-pile-de-registres-et-rotation.md) | Vol. 1 §3.3.1, §3.5.5 (`1:33`–`1:55`, `1:69`–`1:80`) | Cadre de pile (`alloc`, `sof`/`sol`/`sor`), renommage lors de `br.call`/`br.ret`, moteur asynchrone RSE (`BSP`, `RNAT`), rotation `RRB` et boucles pipelinées `br.ctop`/`br.wtop`. |
| **05** | [`05-speculation-controle-et-donnees.md`](documentation/fiches/05-speculation-controle-et-donnees.md) | Vol. 1 §3.4.4–3.4.5 (`1:44`–`1:69`) | Spéculation de contrôle (`ld.s`, propagation du 65e bit `NaT` et `NaTVal`, `chk.s`) et spéculation de données (`ld.a`, table associative `ALAT`, `ld.c`, `chk.a`). |
| **06** | [`06-optimisation-memoire-predication-controle.md`](documentation/fiches/06-optimisation-memoire-predication-controle.md) | Vol. 1 Partie 2 Chap. 1–4 (`1:137`–`1:174`) | Coût réel du code de recouvrement spéculatif, compromis prédication vs branchement, fusion de conditions par comparaisons parallèles, verrous `ld.a` + `hint @pause`. |
| **07** | [`07-optimisation-boucles-et-flottants.md`](documentation/fiches/07-optimisation-boucles-et-flottants.md) | Vol. 1 Partie 2 Chap. 5–6 (`1:175`–`1:218`) | Déroulage vs pipeline logiciel modulo, sorties anticipées `br.cexit`/`br.wexit`, réductions parallèles `fma`, division et racine logicielle Newton-Raphson (`frcpa`/`frsqrta`), `ldfp`, `sf0`–`sf3`. |
| **08** | [`08-calcul-entier-modele-memoire-et-hints.md`](documentation/fiches/08-calcul-entier-modele-memoire-et-hints.md) | Vol. 1 Partie 1 Chap. 4 (`1:81`–`1:106`) | Calcul entier (`addl` limité à `r0..r3`, `addp4`), bitfields (`extr`/`dep`/`shrp`), atomiques (`cmpxchg`, `cmp8xchg16`, `fetchadd`), modèle mémoire faible (`.acq`, `.rel`, `mf`), hints de cache/branchement (`brp`), SIMD entier, `czx`/`popcnt`/`epc`. |
| **09** | [`09-modele-flottant-xmpy-et-ia32.md`](documentation/fiches/09-modele-flottant-xmpy-et-ia32.md) | Vol. 1 Partie 1 Chap. 5–6 (`1:107`–`1:135`) | Format flottant 82 bits, `FPSR` (`sf0`–`sf3`), pourquoi `FR1 = +1.0` interdit `fpadd` en SIMD, multiplication entière 64 bits via la FPU (`setf.sig` $\to$ `xmpy` $\to$ `getf.sig`), et destruction de l'état IA-64 lors des transitions `br.ia`/`jmpe` vers IA-32. |
| **10** | [`10-exemples-assembleur-et-encodage.md`](documentation/fiches/10-exemples-assembleur-et-encodage.md) | Vol. 3 Chap. 3–4 (`3:25`, `3:293`–`3:369`) | Écriture et encodage binaire manuel (41 bits et 128 bits little-endian) de 3 programmes complets (`clamp_abs_diff`, `sum_plus_one` + `double_val`, `vec_add_cloop` vs `vec_add_ctop` + noyau 1 bundle `vec_sum_ctop`) et mesure exacte des `nop` forcés. |

---

## 4. Recherches, vérifications et synthèse Phase 2 (`documentation/recherches/`)

| Document | Contenu |
|---|---|
| [`critiques-ia64-20261019.md`](documentation/recherches/critiques-ia64-20261019.md) | Recensement initial des 8 critiques classiques d'IA-64 dans la littérature et les retours d'expérience. |
| [`apercu-manuel-gemini-20261020.md`](documentation/recherches/apercu-manuel-gemini-20261020.md) | Cartographie des 4 volumes du manuel Intel et première vérification des citations. |
| [`verification-gemini2-20261020.md`](documentation/recherches/verification-gemini2-20261020.md) | Vérification point par point des formats 41 bits et contrôles visuels sur les PDF originaux (coquille d'édition `M1`/`M2` p. `3:296` vs `3:329`, stops de la Table 3-10, `EC` p. `1:77`). |
| **[`synthese-critique-et-choix-ia86.md`](documentation/recherches/synthese-critique-et-choix-ia86.md)** | **Bilan complet de la Phase 2 :** confrontation des 8 critiques au manuel, liste des 6 défauts d'ingénierie internes découverts dans la spécification et matrice de décision sur les 12 sous-systèmes pour **IA-86**. |

---

## 5. En bref : ce qu'IA-86 garde, modifie ou jette d'IA-64

L'analyse détaillée est exposée dans [`synthese-critique-et-choix-ia86.md`](documentation/recherches/synthese-critique-et-choix-ia86.md) :

| ✅ Ce qu'IA-86 conserve | ⚠️ Ce qu'IA-86 transforme | ❌ Ce qu'IA-86 supprime |
|---|---|---|
| • **Groupes d'instructions parallèles explicites** déterminés par le compilateur<br>• **Prédication générale `(p) op`** (`p0 = 1`) et comparaisons doubles `(p1, p2)` (`normal` et `.unc`)<br>• **Compteurs de boucles `LC`/`EC`**, `br.cloop` et pipelines logiciels `br.ctop`<br>• **Modèle mémoire `.acq`/`.rel` par instruction** et atomiques (`cmpxchg`, `fetchadd`)<br>• **Primitives bitfields** (`extr`, `dep`, `shrp`, `mux`, `czx`, `popcnt`) et `fma` | • **Encodage des groupes :** passage du bundle rigide de 128 bits (3 slots de 41 bits + template 5 bits) à des instructions alignées 32/64 bits portant **1 bit de `stop` par instruction** (zéro `nop` de remplissage)<br>• **Registres :** 64 registres généraux uniformes (`r0..r63`, avec `r0 = 0`) et zone rotative de boucle **séparée des arguments d'entrée**<br>• **Spéculation sans exception :** `ld.nf r1, p1 = [r2]` écrivant directement un prédicat de validité `p1` (sans 65e bit `NaT` ni saut de recouvrement hors ligne)<br>• **Flottants :** IEEE 754 standard (32/64/128 bits) avec `fadd`/`fdiv`/`fsqrt` matériels | • **Les 24 templates de 5 bits** et le couplage binaire aux unités `M`/`I`/`F`/`B`<br>• **Le moteur matériel asynchrone RSE** (`alloc`, `BSP`, `RNAT`)<br>• **Le 65e bit physique `NaT`** et le format flottant propriétaire de **82 bits**<br>• **La table associative `ALAT`** (`ld.a`/`ld.c`/`chk.a`)<br>• **Le détour par la FPU (`xmpy`)** pour la multiplication entière 64 bits<br>• **Le mode matériel hybride IA-32** (`br.ia`) |

---

## 6. Manuel de référence officiel

*Intel® Itanium® Architecture Software Developer's Manual*, révision 2.3 (mai 2010), 4 volumes :

| Volume | Titre original | Contenu | Pages |
|---|---|---|---|
| **Volume 1** | *Application Architecture* | Architecture applicative (Partie 1) et Guide d'optimisation (Partie 2) | 244 |
| **Volume 2** | *System Architecture* | Architecture système, pagination, interruptions, firmwares PAL/SAL/EFI | 676 |
| **Volume 3** | *Instruction Set Reference* | Référence alphabétique des instructions et formats binaires de 41 bits | 420 |
| **Volume 4** | *IA-32 Instruction Set* | Jeu d'instructions IA-32 sur processeur Itanium | 604 |

Ces documents appartiennent à Intel Corporation et ne sont pas redistribués dans ce dépôt (`documentation/intel/` est ignoré par Git). Ils sont téléchargeables sur `intel.com`, notamment le [Volume 3](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/itanium-architecture-vol-3-manual.pdf) et le [recueil complet des 4 volumes](https://www.intel.com/content/dam/doc/manual/itanium-architecture-vol-1-2-3-4-reference-set-manual.pdf).

---

## Licence

Code et documentation du projet : [Apache 2.0](LICENSE).  
*Itanium* et *Intel* sont des marques d'Intel Corporation ; ce projet d'étude et de recherche est indépendant et non affilié à Intel.

Auteur : **Théo Gastinel**.
