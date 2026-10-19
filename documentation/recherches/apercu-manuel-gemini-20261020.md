# Aperçu des 4 volumes du manuel Intel (Gemini, 20/10/2026)

Produit par Gemini 4 (effort maximal), à partir des 4 PDF du manuel Intel Itanium, révision 2.3. Texte ci-dessous légèrement condensé par Claude (mise en forme, répétitions de références), sans changer le fond.

## Contrôle par Claude (Opus 5.5), 20/10/2026
Cinq références vérifiées dans les PDF (`pdftotext`) :

| Affirmation | Verdict |
|---|---|
| Format du bundle, Figure 3-15, p. 1:38 | ✅ Confirmé |
| `ldfp` : faute si les deux registres sont dans le même banc (`fp_reg_bank_conflict`) | ✅ Confirmé (vol. 3) |
| Le logiciel ne peut pas compter sur la détection de toutes les violations de dépendances (§ 3.4.4) | ✅ Confirmé |
| RNAT : registre de collecte des bits NaT, bit 63 toujours lu à 0 | ✅ Cohérent avec le manuel (le « toutes les 63 sauvegardes » n'est pas vérifié mot pour mot) |
| « Il est déconseillé aux développeurs de programmer en assembleur Itanium » (p. 1:137) | ⚠️ **Mal interprété** : le manuel dit seulement que le *guide d'optimisation* ne doit pas servir de guide de programmation en assembleur. Ça ne déconseille pas l'assembleur en général. |
| 24 templates définis, 8 réservés | Non vérifié (le tableau ne s'extrait pas bien en texte) |

Conclusion : bon aperçu, fiable dans l'ensemble, mais chaque point repris dans nos fiches doit être revérifié à la source.

---

## Texte de Gemini

Voici un aperçu général de l'architecture IA-64 (EPIC) établi à partir des quatre volumes du manuel officiel *Intel® Itanium® Architecture Software Developer's Manual, Revision 2.3* (mai 2010) (Vol. 1, Couverture, p. i).

### 1. Carte des 4 volumes

- **Volume 1 : Application Architecture** (Vol. 1, Part 1, Chap. 1, Section 1.1, p. 1:3-1:4). Il définit l'environnement d'exécution applicatif, les registres, le modèle de programmation (entier, flottant, multimédia et compatibilité IA-32) en Partie 1, puis détaille les techniques d'optimisation du compilateur en Partie 2. **Chapitres essentiels :** Partie 1, chapitres **2, 3, 4 et 5** (fondations de l'ISA) ; Partie 2, chapitres **2, 3, 4 et 5** (parallélisme, spéculation, prédication, pipelining logiciel).
- **Volume 2 : System Architecture** (Vol. 1, Part 1, Chap. 1, Section 1.2, p. 1:4-1:6). Architecture privilégiée visible par le système d'exploitation (état système, mémoire virtuelle, interruptions, RSE, firmware PAL) en Partie 1, guide système (synchronisation multiprocesseur, gestion de contexte, gestionnaires de fautes) en Partie 2. **Chapitres essentiels :** Partie 1, chapitres **3, 4, 5 et 6** ; Partie 2, chapitres **2, 6 et 7**.
- **Volume 3 : Instruction Set Reference** (Vol. 1, Part 1, Chap. 1, Section 1.3, p. 1:6). Dictionnaire exhaustif du jeu d'instructions natif : comportement en pseudo-code C, formats binaires, règles de dépendances (Vol. 3, Chap. 2, Section 2.1, p. 3:11). **Chapitres essentiels :** chapitre **2** (référence alphabétique), chapitre **4** (formats et encodages sur 41 bits), chapitre **5** (sémantique des ressources, tables RAW/WAW/WAR).
- **Volume 4 : IA-32 Instruction Set Reference** (Vol. 1, Part 1, Chap. 1, Section 1.4, p. 1:6-1:7). Comportement des instructions 32/16 bits (IA-32, MMX, SSE) dans l'environnement Itanium (Vol. 4, Chap. 2, p. 4:11). **Essentiel :** seule la section 2.1 du chapitre 2 (fautes spécifiques Itanium, comme l'avortement de consommation NaT) ; le reste est secondaire pour étudier EPIC `[hors manuel]` (Vol. 4, Chap. 2, Section 2.1, p. 4:11-4:12).

### 2. Les idées centrales d'IA-64

#### Bundles et templates
- **Ce que c'est :** trois instructions de 41 bits regroupées avec un champ *template* de 5 bits dans un conteneur de 128 bits aligné sur 16 octets, le *bundle* (Vol. 1, Part 1, Chap. 3, Section 3.3, Figure 3-15, p. 1:38). Six types d'instructions (`A`, `I`, `M`, `F`, `B`, `L+X`) ; le template associe chacun des trois slots à un type d'unité (`M`, `I`, `F`, `B`, `L+X`) (Tables 3-9 et 3-10, p. 1:38-1:39). Le template indique aussi les *stops* en fin ou au milieu du bundle (p. 1:38 ; Vol. 1, Part 2, Chap. 2, Section 2.3.3, p. 1:141).
- **Problème résolu :** le compilateur communique explicitement l'affectation des unités au matériel, ce qui simplifie le décodage parallèle (Vol. 1, Part 1, Chap. 2, Sections 2.4 et 2.5, p. 1:15-1:16 ; Part 2, Section 2.3.3, p. 1:141).

#### Groupes d'instructions et stops
- **Ce que c'est :** suite d'instructions sans dépendance de registre RAW ni WAW entre elles (sauf cas particuliers) (Vol. 1, Part 1, Chap. 3, Section 3.4, p. 1:40-1:41 ; Part 2, Section 2.3.2, p. 1:140-1:141). Délimités statiquement par des *stops* (`;;`) et dynamiquement par les branchements pris. Toutes les instructions d'un groupe lisent l'état avant qu'aucune ne le modifie ; un stop impose au moins un cycle logique de séparation (p. 1:40).
- **Problème résolu :** le processeur lance les instructions en parallèle sans vérifier matériellement leurs dépendances (Part 2, Section 2.3.2, p. 1:140-1:141).

#### Prédication
- **Ce que c'est :** exécution conditionnée par un prédicat de 1 bit (`PR0` à `PR63`, `PR0` câblé à 1) (Vol. 1, Part 1, Chap. 2, Section 2.6.3, p. 1:17-1:18 ; Chap. 3, Section 3.1.4, p. 1:26). Les comparaisons écrivent deux prédicats (modes normal, inconditionnel, parallèles `AND`/`OR`) (Chap. 4, Sections 4.3.2 et 4.3.3, p. 1:54-1:56). Prédicat à 0 : résultat annulé comme un `nop`, sans exception (p. 1:18 ; Part 2, Section 2.3.1, p. 1:140).
- **Problème résolu :** supprime des branchements en transformant les dépendances de contrôle en dépendances de données ; exécute `then` et `else` en parallèle ; évite les mauvaises prédictions de branchement (Section 2.6.3, p. 1:17-1:18 ; Part 2, Section 2.7, p. 1:145).

#### Spéculation (contrôle et données)
- **Contrôle :** un chargement `ld.s` et ses calculs dépendants sont exécutés *avant* le branchement qui les garde (Vol. 1, Part 1, Section 2.6.1, p. 1:16 ; Part 2, Section 2.4.3, p. 1:142). Une exception est différée sous forme de jeton (bit *NaT*, 65e bit d'un registre général, ou *NaTVal* en flottant) qui se propage jusqu'à `chk.s`, qui saute vers du code de récupération (Chap. 4, Sections 4.4.4.1 et 4.4.4.2, p. 1:60-1:61).
- **Données :** un chargement avancé `ld.a` passe *avant* une écriture potentiellement conflictuelle ; l'adresse est inscrite dans l'**ALAT** (*Advanced Load Address Table*) (Section 2.6.2, p. 1:17 ; Sections 4.4.5.2 et 4.4.5.3, p. 1:64-1:66). Une écriture qui chevauche supprime l'entrée ; `ld.c` ou `chk.a` vérifie ensuite et recharge ou saute vers la récupération (p. 1:64-1:67).
- **Problème résolu :** remonter les chargements au-dessus des branchements et des écritures ambiguës pour masquer la latence mémoire, sans fausser le programme (Sections 2.4 et 2.6, p. 1:15-1:17 ; Part 2, Chap. 3, Section 3.4, p. 1:151-1:152).

#### Registres tournants et boucles pipelinées
- **Ce que c'est :** le pipelining logiciel (*modulo scheduling*) chevauche plusieurs itérations : prologue, noyau, épilogue (Section 2.9, p. 1:19 ; Section 4.5.1, p. 1:75-1:76). Les registres `FR32–FR127`, `PR16–PR63` et une partie des généraux (à partir de `GR32`, par multiples de 8) tournent via `CFM.rrb` (Sections 3.1.2 à 3.1.4 et 3.1.7, p. 1:25-1:27). À chaque branchement de boucle (`br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`), `LC` et `EC` sont décrémentés, les prédicats d'étage s'activent ou s'éteignent, et les registres tournent d'un cran (p. 1:76-1:77 ; Part 2, Section 2.7.3, p. 1:145-1:146).
- **Problème résolu :** paralléliser une boucle sans la dérouler ni renommer les registres en logiciel, donc sans grossir le code (Section 2.9, p. 1:19 ; Part 2, Section 2.7.2, p. 1:145).

#### Pile de registres (RSE)
- **Ce que c'est :** `GR32` à `GR127` forment une pile ; chaque procédure a son cadre (zone locale + zone de sortie), dimensionné par `alloc` (Section 3.1.2, p. 1:25 ; Section 4.1.1, p. 1:47-1:48). À l'appel (`br.call`), la zone de sortie de l'appelant devient le début du cadre de l'appelé (`GR32`) : arguments passés sans copie (Figure 4-1, p. 1:48-1:49 ; Part 2, Section 2.6.1, p. 1:144). Quand les registres physiques manquent, le *Register Stack Engine* bloque et transfère vers une zone mémoire (*backing store*) (Section 2.7, p. 1:18 ; Section 4.1, p. 1:47).
- **Problème résolu :** plus de sauvegardes/restaurations explicites aux appels ; transferts faits par le matériel pendant les cycles libres (Sections 2.4 et 2.7 ; Part 2, Section 2.6, p. 1:144).

#### Modèle mémoire
- **Ce que c'est :** accès par chargements, écritures et trois sémaphores atomiques (`xchg`, `cmpxchg`, `fetchadd`), little-endian ou big-endian (`UM.be`) (Section 3.2.3, p. 1:36 ; Section 4.4, p. 1:57-1:60). Indications de cache sur 2 bits (`.nt1`, `.nt2`, `.nta`) (Section 2.5, p. 1:16 ; Section 4.4.6.1, p. 1:69-1:71). Accès non ordonnés par défaut ; ordre imposé par `ld.acq`, `st.rel` ou `mf` (Section 4.4.7, p. 1:73-1:74 ; Vol. 2, Part 1, Section 4.4.7, p. 2:82-2:84). Cache d'instructions non cohérent avec le cache de données : le code auto-modifiant exige `fc`/`fc.i`, `sync.i`, `srlz.i` (Section 4.4.6.2, p. 1:72-1:73).
- **Problème résolu :** le matériel réordonne librement en multiprocesseur ; les indications du compilateur évitent de polluer les caches (Sections 2.5 et 2.12.3 ; Sections 4.4.6.1 et 4.4.7, p. 1:70-1:73).

### 3. Points fragiles ou coûteux visibles dans le manuel

#### Complexité
- **Cas particuliers dans les dépendances :** la règle « pas de RAW ni WAW dans un groupe » a de nombreuses exceptions asymétriques (prédicat écrit par une instruction entière visible par un branchement du même groupe, mais interdit si l'instruction est flottante ; WAR interdit sur `PR63` entre deux branchements ; `alloc` autorise certains RAW sur `CFM` mais interdit 15 instructions dans le même groupe). Un chapitre entier de tables dans le vol. 3 (Vol. 1, Sections 3.4.1 à 3.4.3, p. 1:42-1:44 ; Vol. 3, Chap. 5, p. 3:371-3:389).
- **Violations pas toujours détectées :** comportement indéfini ; le logiciel ne peut pas supposer que le processeur détectera toutes les violations, ni la même à chaque fois (Section 3.4.4, p. 1:44).
- **Sauts qui ne ferment pas un groupe :** `chk.s`, `chk.a`, `rfi`, `brl` sans stop ne terminent pas le groupe ; des instructions placées après le saut (non exécutées) peuvent rendre des valeurs indéfinies à la cible (Section 3.4, p. 1:40).
- **Arithmétique laissée au logiciel :** pas de division entière, ni de division ou racine carrée flottante directe (via `frcpa`, `frsqrta`) ; multiplication entière 64 bits via les registres flottants (`xma`) ou 5 instructions entières sur 3 groupes (Section 4.2.1, p. 1:51 ; Chap. 5, Sections 5.3.6 et 5.4.7, p. 1:101, 1:107 ; Vol. 3, `mpyshl4`, p. 3:189). Fautes *Software Assist* pour déléguer à l'OS les cas flottants complexes (dénormalisés) (Sections 5.4.1.2 et 5.4.1.3, p. 1:102, 1:104).
- **Sérialisation manuelle pour l'OS :** les écritures dans les registres système ne sont pas sérialisées ; l'OS doit insérer `srlz.d` / `srlz.i` (Vol. 2, Part 1, Section 3.2, p. 2:17-2:19). Changer la cacheabilité d'une page : procédure de 11 étapes (`ptc.ga`, `mf`, `srlz.i`, appels PAL, IPI, `fc` ligne par ligne) (Vol. 2, Section 4.4.11.1, p. 2:88-2:90).

#### Contraintes sur le compilateur
- **Compilateur sophistiqué indispensable :** le guide d'optimisation n'est pas un guide de programmation en assembleur (Vol. 1, Part 2, Chap. 1, p. 1:137) *(voir la correction en tête de fichier)*. Respecter les stops garantit un code correct, mais pour éviter les blocages il faut connaître les latences exactes (Part 2, Section 3.3.2.3, p. 1:150-1:151).
- **Code de vérification et de récupération :** chaque spéculation impose `chk.s`, `chk.a`, `fchkf` et un bloc de récupération qui recopie les instructions spéculées : slots consommés, code plus gros (Sections 4.4.4.2 et 4.4.5.2 ; Part 2, Sections 3.5 et 3.5.1, p. 1:157).
- **Coût caché du RSE et des bits NaT :** les transferts du RSE peuvent coûter cher ; le compilateur doit économiser les registres (Part 2, Section 2.6.2, p. 1:144). Avec `st8.spill`/`ld8.fill`, le bit `UNAT` dépend des bits 8:3 de l'adresse : interdit d'avoir deux spills partageant ces bits dans un même groupe (Sections 3.4.1, 3.4.2 ; Section 4.4.4.6, p. 1:62-1:63).

#### Place perdue dans l'encodage
- **Templates peu orthogonaux :** 24 encodages définis sur 32 (12 combinaisons, avec ou sans stop final), seulement 2 (`MI_I`, `M_MI`) avec un stop au milieu (Table 3-10, p. 1:38-1:39). Branchements de boucle seulement en slot 2 (Section 4.5, p. 1:75) ; `alloc`, `flushrs`, `loadrs` obligatoirement en tête de groupe (Vol. 3, p. 3:16, 3:76, 3:167).
- **NOP typés et instructions sur 2 slots :** cinq variantes de `nop` (`nop.i`, `.m`, `.b`, `.f`, `.x`) pour boucher les slots (Vol. 3, `nop`, p. 3:193). `movl` et `brl` prennent 2 slots dans un bundle `MLX` (p. 1:39) ; toute cible de branchement est alignée sur le slot 0 d'un bundle (Section 4.5, p. 1:74).

#### Détails internes exposés au logiciel
- **ALAT dépendante de l'implémentation :** taille, associativité et remplacement non fixés (une ALAT de zéro entrée est permise) ; le compilateur gère la durée de vie des entrées avec `.clr` et `.nc` (Sections 4.4.5.3.1, 4.4.5.3.2, 4.4.5.5, p. 1:66-1:69 ; Part 2, Section 3.5.1, p. 1:157).
- **Bancs pair/impair des registres flottants :** `ldfp` lève une faute si ses deux registres ont la même parité après rotation (Vol. 3, `ldfp`, p. 3:161-3:162).
- **RSE, TLB et caches exposés :** mot `RNAT` inséré toutes les 63 sauvegardes, passage obligatoire en mode *enforced lazy* avant `loadrs` (Sections 3.1.8.2 à 3.1.8.5, p. 1:29-1:30 ; Vol. 3, p. 3:167, 3:173). `ptc.e` exige deux boucles imbriquées dépendant de l'organisation interne du TLB (Vol. 3, p. 3:230). Alias virtuels à espacer d'un multiple de 1 Mo (Vol. 2, Section 4.1.10, p. 2:72).

### 4. Chiffres clés
- **Bundle :** 128 bits (16 octets), 3 slots de 41 bits + template de 5 bits (Figure 3-15, p. 1:38).
- **Templates :** 32 encodages, 24 définis (12 de base, avec ou sans stop final, dont 2 avec stop au milieu), 8 réservés ; 6 types d'instructions, 5 types d'unités (Tables 3-9 et 3-10).
- **Registres généraux :** 128 × (64 bits + 1 bit NaT) ; 32 statiques (`GR0 = 0`), 96 empilés, dont un multiple de 8 peut tourner ; plus 16 registres en banque (`GR16–GR31`) pour le système (Vol. 2, Section 3.3.7, p. 2:42).
- **Registres flottants :** 128 × 82 bits (1 signe, 17 exposant, biais 65535, 64 mantisse) ; 32 statiques (`FR0 = +0.0`, `FR1 = +1.0`), 96 tournants (Section 3.1.3, p. 1:26 ; Figure 5-1, p. 1:86).
- **Prédicats :** 64 × 1 bit ; 16 statiques (`PR0 = 1`), 48 tournants (Section 3.1.4, p. 1:26).
- **Branchement :** 8 registres de 64 bits (Section 3.1.5, p. 1:26).
- **Registres d'application et d'état :** jusqu'à 128 AR de 64 bits (dont `KR0–KR7`, `LC` 64 bits, `EC` 6 bits, `FPSR` avec 4 champs de 13 bits), `CFM` 38 bits, *User Mask* 6 bits, au moins 5 registres `CPUID` (Sections 3.1.7 à 3.1.11, p. 1:27-1:35).
- **Système :** 4 niveaux de privilège ; 128 registres de contrôle ; 8 *Region Registers* (RID 24 bits, au moins 18 implémentés) ; au moins 16 PKR, 8 ITR, 8 DTR, 4 paires DBR/IBR, 4 paires PMC/PMD (Vol. 2, Sections 3.1, 3.3, 4.1).
- **Adressage :** adresse virtuelle 64 bits (8 régions de 2^61 octets, au moins 54 bits implémentés, espace global de 85 bits) ; physique 63 bits (32 à 63 implémentés) ; pages de 4 Ko à 256 Mo (insertion), jusqu'à 4 Go (purge) ; unités de 1, 2, 4, 8, 10, 16 octets ; ligne de cache visible minimale 32 octets ; branchements relatifs ±16 Mo, ou 64 bits avec `brl`.
- **ALAT et registres physiques du RSE :** non fixés par l'architecture, dépendent de l'implémentation (Section 4.4.5.3.1, p. 1:67 ; Vol. 3, `loadrs`, p. 3:167).

### 5. Ordre de lecture conseillé `[hors manuel]`
Le manuel précise que la Partie 2 du vol. 1 suppose de connaître l'assembleur et l'architecture applicative (Part 2, Section 2.1, p. 1:139).

1. **Vue d'ensemble (1 à 2 jours) :** Vol. 1, Partie 1, chap. 2 (p. 1:13-1:21) et Partie 2, chap. 2 (p. 1:139-1:146).
2. **Modèle d'exécution (1 à 2 semaines) :** Vol. 1, Partie 1, chap. 3 (p. 1:23-1:45) et 4 (p. 1:47-1:84), puis chap. 5 (flottant). Insister sur les sections 3.3, 3.4, 4.1, 4.3, 4.4, 4.5.
3. **Ce que fait le compilateur (2 semaines) :** Vol. 1, Partie 2, chap. 3, 4 et 5 (p. 1:147-1:204).
4. **Encodage binaire (2 semaines) :** Vol. 3, chap. 4 (p. 3:293-3:369) et 5 (p. 3:371-3:396), avec le chap. 2 comme dictionnaire (`alloc`, `br`, `chk`, `ld`, `st`). Étape cruciale pour concevoir un jeu d'instructions.
5. **Côté système (2 semaines) :** Vol. 2, Partie 1, chap. 3 à 6, et Partie 2, chap. 2, 6 et 7. Le vol. 4 et les chapitres IA-32 peuvent être ignorés.
