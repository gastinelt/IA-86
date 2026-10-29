# Fiche 9 — Modèle flottant normatif (Chap. 5), multiplication entière (`xmpy`) et compatibilité IA-32 (Chap. 6)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3, Volume 1, Partie 1, Chapitres 5 et 6 (pages `1:85` à `1:134`).

---

## 1. Format registre flottant 82 bits et classes de valeurs (§ 5.1, Figure 5-1, Tables 5-1 et 5-2, p. `1:85`–`1:88`)

Tous les registres flottants `FR0`–`FR127` ont une largeur unique de **82 bits** :

```text
  81 80               64 63                                                    0
+---+-------------------+-------------------------------------------------------+
| s |     exponent      |       significand (bit 63 = bit entier explicite)     |
| 1 |      17 bits      |                        64 bits                        |
+---+-------------------+-------------------------------------------------------+
```

- **Formule pour un nombre fini normalisé (`exponent > 0`) :** $(-1)^{\text{sign}} \times 2^{(\text{exponent} - 65535)} \times (\text{significand}_{63}.\text{significand}_{62:0})$. Le biais de l'exposant 17 bits vaut **`65535` (`0xFFFF`)**.
- **Encodages remarquables dans les registres `FR` (Table 5-2, p. `1:86`–`1:88`) :**

| Classe / Valeur | Signe (`1b`) | Exposant biaisé (`17b`) | Mantisse (`64b`, avec bit entier `i.bb...`) |
|---|---|---|---|
| **Zéro signé (`+0.0` / `-0.0`)** (`FR0` = `+0.0`) | `0` ou `1` | `0x00000` | `0.000...00` |
| **Constante `+1.0` (`FR1`)** | `0` | **`0x0FFFF`** ($65535$) | `1.000...00` |
| **Entier 64 bits ou *Parallel FP* (forme canonique)** | **`0`** | **`0x1003E`** ($65535 + 63$) | Les 64 bits de l'entier ou des 2 `float` 32 bits |
| **`NaTVal` (exception spéculative différée)** | **`0`** | **`0x1FFFE`** | `0.000...00` *(codage de pseudo-zéro réservé)* |
| **Infini (`+Inf` / `-Inf`)** | `0` ou `1` | `0x1FFFF` | `1.000...00` |
| **Quiet NaN / Signaling NaN** | `0` ou `1` | `0x1FFFF` | `1.1xx...xx` (QNaN) / `1.0xx...xx` non nul (SNaN) |
| **Dénormalisés simple / double précision** | `0` ou `1` | `0x0FF81` (single) / `0x0FC01` (double) | `0.000...01` à `0.111...11` *(non normalisés en registre)* |

---

## 2. Le registre d'état flottant `AR[FPSR]` (`AR 40`, § 5.2, Figures 5-2 et 5-3, Tables 5-3 à 5-6, p. `1:88`–`1:91`)

`FPSR` (64 bits) contient **6 bits de masques de trappes globaux (`traps`, bits `5:0`)** et **4 champs d'état de 13 bits (`sf0` à `sf3`)** :
- **`traps` (bits `5:0`)** : `vd` (Invalid), `dd` (Denormal/Unnormal), `zd` (Zero Divide), `od` (Overflow), `ud` (Underflow), `id` (Inexact). Un bit à **`1` désactive** la trappe IEEE correspondante.
- **Structure de chaque champ `sfx` (13 bits, Figure 5-3, Table 5-4, p. `1:89`) :**
  - **7 bits de contrôle (`6:0`) :**
    - `ftz` (bit 0) : *Flush-to-Zero* (tronque à zéro les résultats en sous-dépassement si `ud == 1`).
    - `wre` (bit 1) : *Widest Range Exponent* (`1` = exposant sur 17 bits complets, `0` = exposant limité au type cible).
    - `pc` (bits `3:2`) : *Precision Control* dynamique (`00` = 24b single, `10` = 53b double, `11` = 64b double-extended), utilisé uniquement si l'instruction n'a pas de completer statique `.s` ou `.d` (Table 5-6, p. `1:90`).
    - `rc` (bits `5:4`) : mode d'arrondi (`00` au plus proche pair, `01` vers $-\infty$, `10` vers $+\infty$, `11` vers $0$).
    - `td` (bit 6) : *Traps Disable* local au champ `sf1`–`sf3` (réservé = `0` dans `sf0`).
  - **6 drapeaux d'exception collants (*sticky flags*, bits `12:7`) :** `v`, `d`, `z`, `o`, `u`, `i`.
- **Synchronisation d'un champ spéculatif (`sf1`–`sf3`) vers `sf0` (§ 5.3.5, Table 5-16, p. `1:100`–`1:101`) :**
  - **`fchkf.sfx target25`** compare les drapeaux de `sfx` avec `sf0.flags` et `FPSR.traps` : si une exception activée est survenue ou si un drapeau allumé dans `sfx` est encore à `0` dans `sf0.flags`, **`fchkf` saute au code de récupération** !
  - **`fsetc.sfx and_mask7, or_mask7`** initialise les bits de contrôle de `sfx` à partir de ceux de `sf0` ; **`fclrf.sfx`** remet les 6 drapeaux de `sfx` à `0`.

---

## 3. Instructions flottantes, pseudo-opérations et multiplication entière `xma` / `xmpy` (§ 5.3, p. `1:91`–`1:101`)

### 3.1 Pas d'addition ni de multiplication flottante dédiée : tout repose sur `fma` (§ 5.3.3, Table 5-13, p. `1:98`–`1:99`)
Grâce aux constantes câblées **`FR0 = +0.0`** et **`FR1 = +1.0`**, l'unité flottante n'implémente en matériel que `fma` ($f_3 \times f_4 + f_2$), `fms` ($f_3 \times f_4 - f_2$) et `fnma` ($-(f_3 \times f_4) + f_2$) :
- `fmpy f1 = f3, f4` est un alias assembleur de **`fma f1 = f3, f4, f0`** (*Attention p. `1:99` : utiliser un autre registre contenant `+0.0` que `f0` ne donne pas toujours le signe de zéro IEEE exact d'une multiplication !*).
- `fadd f1 = f3, f2` est un alias de **`fma f1 = f3, f1, f2`**.
- `fsub f1 = f3, f2` est un alias de **`fms f1 = f3, f1, f2`**.
- `fnorm f1 = f3` (normalisation / conversion) est un alias de **`fma f1 = f3, f1, f0`**.
- **Conséquence bizarre pour le *Parallel FP* (2 $\times$ `float` 32 bits, p. `1:99`) :** comme `FR1` vaut `1.0` en format 82 bits et **ne contient pas** une paire de `1.0f` 32 bits, **il n'existe pas d'instruction `fpadd` ni `fpsub` directe** ! Pour additionner deux paires de `float` en SIMD, il faut d'abord fabriquer une paire `(1.0f, 1.0f)` dans un registre avec `fpack` puis appeler `fpma` !

### 3.2 Transferts `GR` $\leftrightarrow$ `FR` (`setf` / `getf`) et la multiplication entière `xma` (§ 5.3.2 et § 5.3.6, Tables 5-8 à 5-10 et 5-17, p. `1:97`–`1:98` et `1:101`)
Les unités entières (I/M) **n'ont pas de multiplieur 64 bits**. La multiplication entière 64 bits (`xma.l`, `xma.h`, `xma.hu`, ou pseudo-op `xmpy` avec `f0`) s'exécute **dans l'unité flottante (F-unit)** sur la mantisse 64 bits des registres `FR` :

```assembly
// Séquence obligatoire pour multiplier deux registres généraux r2 et r3 dans r1 :
setf.sig f6 = r2        // M-unit : transfère r2 dans la mantisse de f6 (latence ~6 cycles !)
setf.sig f7 = r3 ;;     // M-unit : transfère r3 dans la mantisse de f7 (exp = 0x1003E)
xmpy.l   f8 = f6, f7 ;; // F-unit : encode xma.l f8 = f6, f7, f0 (latence ~5 cycles)
getf.sig r1 = f8        // M-unit : récupère la mantisse 64 bits de f8 dans r1 (latence ~5 cycles)
```

- `setf.*` et `getf.*` (exécutés sur **M-unit**) convertissent automatiquement `rX.nat == 1` $\leftrightarrow$ `NaTVal` (Tables 5-9 et 5-10, p. `1:98`).
- **`PSR.dfl` et `PSR.dfh` (§ 5.4.1.1, p. `1:102`) :** comme même un programme purement entier a besoin des registres flottants pour faire une multiplication ou une division entière, l'OS peut désactiver séparément `FR32–FR127` (`PSR.dfh = 1`) et laisser `FR2–FR31` actifs (`PSR.dfl = 0`) pour éviter de sauvegarder les 96 registres flottants hauts lors d'un changement de contexte d'un processus « entier ».

---

## 4. Le modèle d'exécution matériel IA-32 (Chapitre 6, p. `1:109`–`1:134`)

Les premiers processeurs Itanium (Merced, McKinley) intégraient un décodeur matériel IA-32 (niveau Pentium III : 16/32 bits, MMX, SSE), remplacé plus tard par le traducteur binaire logiciel *IA-32 Execution Layer* (§ 6.1, p. `1:109`, puis retiré sur Poulson) :

1. **Basculement de jeu d'instructions (`PSR.is`, § 6.2.1, Figure 6-1, p. `1:110`–`1:112`) :**
   - `PSR.is = 0` (mode Itanium) $\xrightarrow{\text{\texttt{br.ia bX}}}$ `PSR.is = 1` (mode IA-32).
   - `PSR.is = 1` (mode IA-32) $\xrightarrow{\text{\texttt{jmpe} ou interruption}}$ `PSR.is = 0` (mode Itanium).
2. **Mappage de tout l'état x86 dans les registres Itanium (§ 6.2.2, Figure 6-2, Table 6-1, p. `1:113`–`1:117`) :**
   - `GR8`–`GR15` (bits `31:0`) = `EAX`, `ECX`, `EDX`, `EBX`, `ESP`, `EBP`, `ESI`, `EDI`.
   - `GR16`–`GR17` = sélecteurs de segments (`DS`, `ES`, `FS`, `GS`, `CS`, `SS`, `LDT`, `TSS`) ; `GR24`, `GR27`–`GR31` et `AR25` (`CSD`), `AR26` (`SSD`) = descripteurs de segments « désembrouillés » (*unscrambled*).
   - `AR21` (`FCR`), `AR24` (`EFLAG`), `AR27` (`CFLG` = `CR0`/`CR4`), `AR28` (`FSR`), `AR29` (`FIR`), `AR30` (`FDR`) = registres de contrôle/état x86 et x87.
   - `FR8`–`FR15` = pile x87 `ST(0)`–`ST(7)` / `MM0`–`MM7` ; paires `FR16`–`FR31` = `XMM0`–`XMM7` (128 bits répartis sur les mantisses de 2 registres `FR` pair/impair).
3. **Destruction de l'état EPIC lors du passage en mode IA-32 (§ 6.2.4, p. `1:133`–`1:134`) :**
   - Avant `br.ia`, le logiciel **doit vider toute la pile de registres avec `flushrs`** : le RSE est désactivé en mode IA-32 et **écrase `GR32`–`GR127`, `FR32`–`FR127`, `PR1`–`PR63` et l'`ALAT`** comme zone de travail temporaire microarchitecturale !
   - Aucun des registres `GR8`–`GR31` mappés à l'état IA-32 ne doit avoir son bit `NaT == 1` à l'entrée en mode IA-32, sous peine d'un *NaT Register Consumption Abort* au milieu d'une instruction x86.

---

## 5. Regard critique & limites architecturales (pour IA-86)

| Mécanisme | Verdict | Analyse critique |
|---|---|---|
| **Multiplication entière exilée dans les `FR` (`setf.sig` $\to$ `xma` $\to$ `getf.sig`)** | ❌ **Erreur majeure** | Obliger une multiplication entière `r2 * r3` à traverser deux fois les unités mémoire (`setf.sig` / `getf.sig`) et l'unité flottante impose **3 instructions, 2 stops et ~16 à 18 cycles de latence** (comme le montre l'aveu même d'Intel dans la [Fiche 6 § 3 Cas A](file:///C:/Users/theog/Documents/_CODE/10-ia64/IA86/documentation/fiches/06-optimisation-memoire-predication-controle.md#L51) où `r2 * r1` prend 18 cycles contre 2 pour `r2 + r1` !). Dans IA-86, `mul` et `madd` entiers doivent être natifs sur les registres généraux en 3–4 cycles. |
| **Format flottant 82 bits universel + `fma` + `FR0`/`FR1`** | ⚠️ **Demi-succès** | L'idée d'unifier `+`, `-`, `*` et `fma` grâce à `FR0 = 0.0` et `FR1 = 1.0` est très élégante pour le scalaire, mais **casse la vectorisation SIMD** (déjà visible sur le *Parallel FP* 2x32 bits où `FR1` ne marche plus, et impossible à étendre proprement à des vecteurs de 128/256/512 bits). |
| **Exceptions IEEE différées au logiciel (*Software Assist fault/trap*, § 5.4.1.2 & 5.4.7, p. `1:102`–`1:107`)** | ❌ **Piège de performance** | Autoriser le matériel à déclencher une interruption *Software Assist* vers l'OS sur les dénormalisés ou les cas limites de `frcpa` provoque des effondrements de performance imprévisibles (des centaines de cycles par trappe) dès qu'une simulation physique rencontre des sous-dépassements. |
| **Compatibilité matérielle IA-32 (`br.ia` / `AR21–30`)** | ❌ **Poids mort** | A encombré la banque `AR`, compliqué le renommage des `FR`/`GR`, forcé des vidages `flushrs` complets à chaque appel inter-mode, pour être finalement remplacé par un émulateur logiciel (IA-32 EL) dès Itanium 2. |
