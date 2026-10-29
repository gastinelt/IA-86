# Vérification du 2e rapport Gemini (20/10/2026)

Par Claude (Opus 5.5), nuit du 20/10/2026. Méthode : texte des 4 PDF extrait avec `pdftotext -layout` puis recherche des passages cités. Les numéros de page sont ceux imprimés dans le manuel (« 1:42 » = volume 1, page 42).

**Verdict d'ensemble : ⚠️ bon travail, utilisable comme base, mais 1 vraie erreur dans un programme, 1 fausse limitation, et toute la partie encodage était à refaire (faite ci-dessous).**

## 0. La limite annoncée par Gemini est fausse
Gemini dit n'avoir eu accès au volume 3 que jusqu'à la page 3:259. ❌ **Le PDF est complet** : `xor`, `tbit`, `zxt` et tout le chapitre 4 (formats, opcodes) y sont. Il a donc écrit « non trouvé dans le manuel » à tort pour xor, tbit, zxt et pour tous les encodages. C'était honnête de ne rien inventer, mais il faudra lui fournir le volume 3 seul la prochaine fois.

## 1. Sous-ensemble d'instructions : formats ✅
Tous les formats annoncés correspondent au manuel (vol. 3, chap. 2, ligne « Format: » de chaque instruction) :

| Instruction | Gemini | Manuel | Verdict |
|---|---|---|---|
| add / adds / addl | A1 / A4 / A5 | A1 / A4 / A5 | ✅ |
| sub (reg / imm8) | A1 / A3 | A1 / A3 | ✅ |
| shladd | A2 | A2 | ✅ |
| and, andcm, or (reg / imm8) | A1 / A3 | A1 / A3 | ✅ |
| xor | « non trouvé » | A1 / A3 | ❌ trouvé (vol. 3, chap. 2) |
| shl (variable) | I7 | I7 | ✅ |
| shr, shr.u | I5 | I5 | ✅ |
| extr, extr.u | I11 | I11 | ✅ |
| dep / dep.z | I15, I14, I12, I13 | I15, I14, I12, I13 | ✅ |
| sxt | I29 | I29 | ✅ |
| zxt | « non trouvé » | I29 | ❌ trouvé |
| tbit | « non trouvé » | I16 | ❌ trouvé |
| mpy4 | I2 | I2 | ✅ |
| movl | X2 | X2 | ✅ |
| cmp, cmp4 | A6 / A7 / A8 | A6 / A7 / A8 | ✅ |
| mov pr (from / to / rot) | I25 / I23 / I24 | I25 / I23 / I24 | ✅ |
| ld (sans incrément / reg / imm9) | M2 / M2 / M3 | M1 (noté M2 par coquille hors Table 4-4) / M2 / M3 | ✅ voir ci-dessous |
| st (normal / imm9) | M6 / M5 | M4 (noté M6 par coquille hors Table 4-4) / M5 | ✅ voir ci-dessous |
| br (relatif / compté / indirect) | B1 / B2 / B4 | B1 / B2 / B4 | ✅ |
| br.call (relatif / indirect) | B3 / B5 | B3 / B5 | ✅ |
| alloc | M34 | M34 | ✅ |
| mov b1 = r2 / mov r1 = b2 | I21 / I22 | I21 / I22 | ✅ |
| mov.i ar / mov.m ar | I26-I28 / M29-M31 | I26-I28 / M29-M31 | ✅ |
| clrrrb | B8 | B8 | ✅ |
| nop.i/b/m/f/x | I18, B9, M48, F16, X5 | idem | ✅ |
| break.i/b/m/f/x | I19, B9, M37, F15, X1 | idem | ✅ |

✅ **Contrôle visuel (29/10/2026) — `ld r1 = [r3]` (`M1` vs `M2`) et `st [r3] = r2` (`M4` vs `M6`) :**
C'est une **coquille d'édition du manuel Intel (rév. 2.3)** :
- La table récapitulative **Table 4-4** (vol. 3, p. 3:296) numérote dans l'ordre logique : `M1` = *Int Load* (`r1 = [r3]`, bits 19:13 ignorés en blanc), `M2` = *Int Load +Reg* (`r1 = [r3], r2`), `M3` = *Int Load +Imm*, `M4` = *Int Store* (`[r3] = r2`, bits 12:6 ignorés en blanc), `M5` = *Int Store +Imm*, `M6` = *FP Load*, `M9` = *FP Store*, `M13` = *Line Prefetch*.
- Mais aux pages `ld` (p. 3:151), `st` (p. 3:247), dans les tables d'opcodes 4-30/4-34 (p. 3:324, 3:326) et aux sections **4.4.1.1** (p. 3:329) et **4.4.1.4** (p. 3:332), le manuel imprime `M2` pour les deux chargements entiers (en distinguant quand même le schéma graphique : bits 19:13 blancs en 4.4.1.1, champ `r2` jaune en 4.4.1.2) et décale les stores entiers sur `M6`, les FP loads sur `M9`, les FP stores sur `M13` et `lfetch` sur `M18`.
- L'encodage binaire, lui, est sans aucune ambiguïté : bit 36 `m = 0` sans post-incrément (bits 19:13 ignorés = 0), `m = 1` avec post-incrément par `r2`.

## 2. Encodage bit à bit (refait d'après le vol. 3, chap. 4)
✅ Bundle (Figure 3-15, p. 1:38) : template bits 4:0, slot 0 bits 45:5, slot 1 bits 86:46, slot 2 bits 127:87. Les formules de découpage de Gemini sont justes.

Schémas des 41 bits (bit 40 à gauche) :

| Format | Champs (taille en bits) |
|---|---|
| A1 (add r1 = r2, r3), section 4.2.1.1 | op=8 (4) · 0 (1) · x2a (2) · ve (1) · x4 (4) · x2b (2) · r3 (7) · r2 (7) · r1 (7) · qp (6) |
| A4 (adds r1 = imm14, r3), section 4.2.1.4 | op=8 (4) · s (1) · x2a=2 (2) · ve=0 (1) · imm6d (6) · r3 (7) · imm7b (7) · r1 (7) · qp (6) |
| A6 (cmp p1, p2 = r2, r3), section 4.2.2.1 | op=C à E (4) · tb (1) · x2 (2) · ta (1) · p2 (6) · r3 (7) · r2 (7) · c (1) · p1 (6) · qp (6) |
| M1/M2 (ld r1 = [r3]), section 4.4.1.1 | op=4 (4) · m (1) · x6 (6) · hint (2) · x (1) · r3 (7) · r1 (7) · qp (6) ; ld1/2/4/8 → x6 = 00/01/02/03 |
| M6 (st [r3] = r2), section 4.4.1.4 | op=4 (4) · m (1) · x6 (6) · hint (2) · x (1) · r3 (7) · r2 (7) · qp (6) |
| B1 (br relatif), section 4.5.1.1 | op=4 (4) · s (1) · d (1) · wh (2) · imm20b (20) · p (1) · btype (3) · qp (6) |
| M34 (alloc), section 4.4.9.1 | op=1 (4) · 0 (1) · x3 (3) · 0 (2) · sor (4) · sol (7) · sof (7) · r1 (7) · qp (6) |

Opcodes de l'addition (Table 4-9, p. 3:300) : add = x2a 0, ve 0, x4 0, x2b 0 ; add +1 = x2b 1 ; sub = x4 1, x2b 1 ; and/andcm/or/xor = x4 3, x2b 0/1/2/3. Table 4-8 : adds = x2a 2.

### Le bundle d'exemple, encodé
`{ .mii  ld4 r28 = [r8] ; add r9 = 2, r1 ; add r30 = 1, r1 ;; }`, template **0x01** (MII, stop à la fin) :

| Partie | Valeur (hexadécimal) |
|---|---|
| slot 0 : ld4 r28 = [r8] | `08080800700` |
| slot 1 : adds r9 = 2, r1 | `10800104240` |
| slot 2 : adds r30 = 1, r1 | `10800102780` |
| bundle (128 bits) | `84000813c0420004109010101000e001` |
| octets en mémoire (little-endian) | `01 e0 00 10 10 10 90 10 04 00 42 c0 13 08 00 84` |

⚠️ **Calculé par un script à partir des tables du manuel, non testé** avec un vrai assembleur IA-64 (aucun n'est installé). À contrôler plus tard avec un désassembleur (binutils `objdump` compilé pour ia64, par exemple).

## 3. Tableau des templates (Table 3-10, p. 1:38-1:39 et Table 4-2, p. 3:294)
✅ Unités par slot exactes pour les 32 valeurs. ✅ Réservés : 06, 07, 14, 15, 1A, 1B, 1E, 1F. ✅ MLX = 04/05 (anciennement MLI, note a).
✅ **Contrôle visuel (29/10/2026) — Position exacte des stops (doubles traits) :**
- La règle **« bit 0 = 1 ↔ stop après le slot 2 (fin du bundle) »** est **confirmée à 100 %** sur les 24 templates définis (tous les codes impairs ont un double trait à droite du slot 2, aucun code pair n'en a).
- Seuls 4 templates ont un stop interne :
  - `0x02` (`MI_I`) : stop après le slot 1.
  - `0x03` (`MI_I_`) : stop après le slot 1 **et** après le slot 2.
  - `0x0A` (`M_MI`) : stop après le slot 0.
  - `0x0B` (`M_MI_`) : stop après le slot 0 **et** après le slot 2.

## 4. Les trois programmes

| Programme | Verdict | Détail |
|---|---|---|
| 1. Somme de 1 à N | ✅ | Règles justes : `alloc` en tête de groupe, qualifiant p0 (p. 1:42) ; écriture de LC puis branchement dans le même groupe **interdite**, donc le stop est nécessaire (p. 1:42) ; écriture de PFS puis `br.ret` dans le même groupe **permise** (p. 1:42) ; `br.cloop` décrémente LC tant qu'il est non nul. |
| 2. Si/sinon prédiqué | ❌ **Erreur** | Le bundle 2 n'a pas de `;;` alors que le bundle 3 lit `r12` et `r13` écrits dans le bundle 2 : c'est une dépendance RAW dans le même groupe, interdite (§ 3.4, p. 1:40-1:41). Gemini l'écrit lui-même dans son commentaire (« doit avoir un stop, donc 0x09 ») mais ne met pas le `;;` dans le code. Correction : `(p1) mov r16 = r12 ;;` à la fin du bundle 2. |
| 3. Boucle pipelinée (br.ctop) | ✅ (logique) | `clrrrb` en dernière instruction du groupe (p. 3:37) ; `alloc` qui change `sor` exige des `rrb` à zéro, sinon faute Reserved Register/Field (vol. 3, page d'`alloc`) : d'où `clrrrb` avant ; `br.ctop` interdit dans le même groupe qu'`alloc` (p. 1:42) : respecté. ✅ **Contrôle visuel (29/10/2026) :** p. 1:77 (§ 4.5.1) confirme mot pour mot : *« LC is initialized to one less than the number of iterations for the counted loop and EC is initialized to the number of stages into which the loop body has been partitioned. »* |

## 5. Tableau « garder / simplifier / jeter »
C'est l'avis de Gemini (marqué « hors manuel »), pas un fait. Les faits qu'il cite sont justes :
- ✅ Le prédicat qualificatif prend 6 bits sur 41 (≈ 15 %) : visible dans tous les formats (champ `qp`, bits 5:0).
- ✅ Une ALAT de **zéro entrée** est permise (ld.c et chk.a se comportent alors comme un échec) : vol. 1, § 4.4.5.3.
- ✅ Les registres font 64 bits + 1 bit NaT ; `st8.spill`/`ld8.fill` passent par UNAT, indexé par les bits 8:3 de l'adresse (p. 1:42-1:43).
- Le « 20 à 30 % de nop » n'est pas dans le manuel (estimation de Gemini).
Nos propres verdicts se feront plus tard, dans la phase 2.

## 6. Ce qu'un simulateur peut ignorer ou non : ✅
- ✅ Écrire dans GR0 provoque une faute Illegal Operation (p. 1:25).
- ✅ PR0 vaut toujours 1 en lecture, et ce qu'on y écrit est jeté (p. 1:26).
- ✅ `cmp.unc` avec qp = 0 met les deux prédicats à 0 (Table 4-9, vol. 1).
- ✅ Plusieurs comparaisons peuvent écrire le même prédicat dans un groupe si elles sont toutes de type AND ou toutes de type OR, et tout est permis sur PR0 (§ 3.4.2).
- ✅ `mov pr` accède aux prédicats tournants comme si `CFM.rrb.pr` valait 0 (vol. 1, § 4.3.4 et page de `mov pr`).
- ✅ `br.call` : adresse de retour = IP + 16 (pseudo-code du vol. 3).
- ✅ Une écriture de BR, PFS ou d'un prédicat (sauf depuis une instruction flottante) est vue par un branchement du même groupe ; c'est **interdit pour LC et EC** (p. 1:42). Gemini l'a bien dit.

## À faire ensuite
- [x] Regarder dans le PDF : p. 3:323 / 3:296 / 3:329 (M1 ou M2), Table 3-10 (stops), p. 1:76-1:77 (EC) — fait le 29/10/2026.
- [ ] Trouver un désassembleur ia64 pour tester l'encodage du bundle d'exemple.
