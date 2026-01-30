# Système de Classement Elo - PokeRP Strat

## Vue d'ensemble

Ce document décrit le système de classement compétitif pour les combats Pokémon.

> **Paramétrable** : La plupart des valeurs (points, temps, seuils) sont configurables via `config/pokerp-strat/ranked.json`. Voir [CONFIG_RANKED.md](CONFIG_RANKED.md).
Le système est conçu pour :
- Récompenser l'investissement des joueurs, même avec un winrate modeste
- Protéger les débutants des pertes trop violentes
- Maintenir un challenge pour les joueurs expérimentés
- Encourager le jeu régulier via un système de decay

---

## Structure des Rangs

Les rangs sont **entièrement configurables** via `config/pokerp-strat/ranked.json` (section `ranks`).
Chaque rang possède **5 sous-rangs par défaut** (I, II, III, IV, V), paramétrables via `winsPerSubRank`.

**Valeurs par défaut** :

| Rang | Elo | Sous-rangs | ~Victoires/sous-rang |
|------|-----|------------|----------------------|
| Pokeball | 0 - 249 | I à V | 2-3 |
| Superball | 250 - 599 | I à V | 3-4 |
| Hyperball | 600 - 1049 | I à V | 4-5 |
| Masterball | 1050+ | I à V | 6-7 |

**Elo de départ** : 250 (Superball I) — configurable via `elo.startingElo`

### Matchs de placement (Unranked)

Les nouveaux joueurs commencent en **"Unranked"** et doivent jouer **3-4 matchs de placement** avant d'obtenir un rang.

- **Plage de placement** : Pokeball I (0) à Superball V (530)
- **Elo de départ pendant placement** : 265 (milieu de la plage)
- **K factor** : 64 (chaque match a plus d'impact pour converger vite)
- **Après placement** : le joueur reçoit son rang définitif

Voir [CONFIG_RANKED.md](CONFIG_RANKED.md) section `placement` pour la configuration.

---

## Calcul de la Probabilité de Victoire

La probabilité attendue de victoire se calcule selon la formule Elo standard :

```
P(A gagne) = 1 / (1 + 10^((EloB - EloA) / 400))
```

### Exemples

| Elo Joueur A | Elo Joueur B | Écart | Proba A gagne |
|--------------|--------------|-------|---------------|
| 1200 | 1200 | 0 | 50% |
| 1400 | 1200 | +200 | 76% |
| 1600 | 1200 | +400 | 91% |
| 1200 | 1400 | -200 | 24% |
| 1200 | 1600 | -400 | 9% |

---

## Calcul des Points

### Formule générale

```
Points = Variation Elo + Bonus Participation + Bonus Winstreak
```

### Variation Elo

```
SI victoire:
    Variation = +K_GAIN × (1 - Probabilité attendue)
SINON:
    Variation = -K_LOSS × Probabilité attendue
```

Pour des matchs équilibrés (50/50), cela se simplifie en :
- Victoire : `+K_GAIN / 2`
- Défaite : `-K_LOSS / 2`

---

## Facteurs K par Rang

Le système utilise des facteurs K asymétriques pour protéger les débutants :

| Rang | K Gain | K Perte | Effet sur les pertes |
|------|--------|---------|----------------------|
| Pokeball | 32 | 8 | Pertes ÷4 |
| Superball | 32 | 12 | Pertes ÷2.67 |
| Hyperball | 32 | 24 | Pertes ×0.75 |
| Masterball | 32 | 32 | Normal |

### Implémentation

```java
// Valeurs par défaut — l'implémentation doit charger depuis la config (ranks[].minElo, elo.kLoss)
int getKLoss(int elo) {
    if (elo < 250)   return 8;   // Pokeball
    if (elo < 600)   return 12;  // Superball
    if (elo < 1050)  return 24;  // Hyperball
    return 32;                    // Masterball
}
```

---

## Bonus de Participation

Chaque match joué rapporte **+3 points**, quelle que soit l'issue.

Ce bonus permet aux joueurs investis de progresser même avec un winrate modeste.

---

## Bonus Winstreak

Les victoires consécutives sont récompensées par un bonus croissant :

| Victoires consécutives | Bonus |
|------------------------|-------|
| 1 | +0 |
| 2 | +2 |
| 3 | +4 |
| 4 | +6 |
| 5+ | +8 (plafonné) |

### Implémentation

```java
int getWinstreakBonus(int streak) {
    if (streak <= 1) return 0;
    return Math.min((streak - 1) * 2, 8);
}
```

### Exemple

Un joueur qui enchaîne 5 victoires :
- Match 1 : +16 + 3 + 0 = **+19**
- Match 2 : +16 + 3 + 2 = **+21**
- Match 3 : +16 + 3 + 4 = **+23**
- Match 4 : +16 + 3 + 6 = **+25**
- Match 5 : +16 + 3 + 8 = **+27**

**Total : +115 points** (au lieu de +95 sans winstreak)

---

## Decay d'Inactivité

Les joueurs inactifs perdent progressivement des points. Le système est **progressif** : la perte augmente avec le temps d'inactivité, et reste **plus forte en haut de ladder** qu'en bas.

### Définition de l'inactivité

**Seuls les jours où le joueur se connecte SANS jouer de match classé comptent.**

- Jour où le joueur **ne se connecte pas** → **ne compte pas** (pas de pénalité)
- Jour où le joueur **se connecte mais ne joue pas de classé** → compte comme jour d'inactivité

> *Exemple : Un joueur part en vacances 2 semaines sans connexion. À son retour, son compteur d'inactivité n'a pas bougé. En revanche, s'il se connecte chaque jour pour jouer en créatif sans toucher au classé, ces jours comptent.*

### Pas de seuil minimum

Le decay peut faire descendre un joueur à la division inférieure. Il n'y a **aucune protection** : si les points perdus dépassent le seuil du rang, le joueur descend.

### Système progressif

La perte augmente jour après jour jusqu'à un plafond :

| Jour d'inactivité (après délai) | Taux de base | Pokeball | Superball (×0.5) | Hyperball (×0.75) | Masterball (×1.0) |
|---------------------------------|--------------|----------|------------------|-------------------|-------------------|
| Jour 1 (J+7) | -2 | — | -1 | -2 | -2 |
| Jour 2 (J+8) | -5 | — | -3 | -4 | -5 |
| Jour 3 (J+9) | -8 | — | -4 | -6 | -8 |
| Jour 4 (J+10) | -11 | — | -6 | -8 | -11 |
| Jour 5+ (J+11+) | -15 (max) | — | -8 | -11 | -15 |

### Délai avant decay (jours de grâce)

| Rang | Délai avant 1er decay |
|------|------------------------|
| Pokeball | Aucun decay |
| Superball | 7 jours |
| Hyperball | 7 jours |
| Masterball | 7 jours |

Le délai est le **nombre de jours connectés sans match classé** avant que le decay ne commence.

### Règles de réinitialisation

1. **Chaque match classé joué** → réinitialise le compteur d'inactivité à 0
2. **Connexion sans match classé** → incrémente le compteur
3. **Pas de connexion** → le compteur ne change pas

### Implémentation

```java
/**
 * Calcule la perte de points par decay.
 * @param elo Elo actuel du joueur
 * @param daysInactiveInClassed Nombre de jours où le joueur s'est connecté
 *                               SANS jouer de match classé (depuis son dernier match)
 * @return Nombre de points à retirer (toujours >= 0)
 */
int calculateDecay(int elo, int daysInactiveInClassed) {
    if (elo < 250) return 0; // Pokeball : pas de decay
    
    // Facteur multiplicatif selon le rang (plus fort en haut)
    double rankFactor;
    if (elo < 600)       rankFactor = 0.5;   // Superball
    else if (elo < 1050) rankFactor = 0.75;  // Hyperball
    else                 rankFactor = 1.0;   // Masterball
    
    int gracePeriod = 7;
    int daysOfDecay = Math.max(0, daysInactiveInClassed - gracePeriod);
    
    if (daysOfDecay == 0) return 0;
    
    // Taux progressif par jour (base avant facteur de rang)
    // Jour 1: -2, Jour 2: -5, Jour 3: -8, Jour 4: -11, Jour 5+: -15
    int[] decayRates = {2, 5, 8, 11, 15};
    
    int totalDecay = 0;
    for (int i = 0; i < daysOfDecay; i++) {
        int rateIndex = Math.min(i, decayRates.length - 1);
        int baseRate = decayRates[rateIndex];
        totalDecay += (int) Math.round(baseRate * rankFactor);
    }
    
    return totalDecay;
}
```

### Exemple de calcul

**Joueur Masterball à 3200 Elo, 12 jours connectés sans match classé :**

```
Jours de grâce : 7
Jours de decay : 12 - 7 = 5 jours

Jour 1 : -2 pts
Jour 2 : -5 pts
Jour 3 : -8 pts
Jour 4 : -11 pts
Jour 5 : -15 pts
─────────────────
Total : -41 points

Nouveau Elo : 3200 - 41 = 3159 (reste Masterball)
```

**Même joueur, 20 jours d'inactivité en classé :**

```
Jours de decay : 13 jours

Jours 1-4 : -2, -5, -8, -11 = -26
Jours 5-13 : 9 × -15 = -135
─────────────────────────────
Total : -161 points

Nouveau Elo : 3200 - 161 = 3039 (reste Masterball)
```

**Joueur Masterball à 3050, 15 jours d'inactivité :**

```
Jours de decay : 8 jours
Total : -2-5-8-11-15-15-15-15 = -86 points

Nouveau Elo : 3050 - 86 = 2964 → DESCEND en Hyperball ✓
```

### Données à stocker par joueur

| Champ | Description |
|-------|-------------|
| `lastClassedMatchDate` | Date du dernier match classé joué |
| `daysInactiveInClassed` | Nombre de jours où le joueur s'est connecté SANS jouer de classé (depuis dernier match) |
| `lastDateCountedForDecay` | Dernière date pour laquelle un jour a été ajouté au compteur (évite double comptage) |

**Logique à chaque connexion :**
1. Si le joueur a joué un match classé aujourd'hui → `lastClassedMatchDate = today`, `daysInactiveInClassed = 0`, `lastDateCountedForDecay = null`
2. Sinon, si `today != lastDateCountedForDecay` et `today > lastClassedMatchDate` :
   - `daysInactiveInClassed++`
   - `lastDateCountedForDecay = today`
3. Appliquer le decay si `daysInactiveInClassed > 7`

> *Ainsi, les jours sans connexion ne sont jamais comptés : on n'incrémente que lorsqu'on détecte une connexion sans match classé.*

---

## Récapitulatif : Calcul Complet des Points

```java
int calculatePoints(boolean victory, int elo, int winstreak, int opponentElo) {
    int kGain = 32;
    int kLoss = getKLoss(elo);
    int participationBonus = 3;
    
    // Calcul de la probabilité attendue
    double expectedProb = 1.0 / (1.0 + Math.pow(10, (opponentElo - elo) / 400.0));
    
    if (victory) {
        int eloGain = (int) Math.round(kGain * (1 - expectedProb));
        int streakBonus = getWinstreakBonus(winstreak);
        return eloGain + participationBonus + streakBonus;
    } else {
        int eloLoss = (int) Math.round(kLoss * expectedProb);
        return -eloLoss + participationBonus;
    }
}
```

### Version simplifiée (matchs équilibrés)

Pour des adversaires de niveau similaire :

```java
int calculatePointsSimple(boolean victory, int elo, int winstreak) {
    int participationBonus = 3;
    
    if (victory) {
        int baseGain = 16; // K_GAIN / 2
        int streakBonus = getWinstreakBonus(winstreak);
        return baseGain + participationBonus + streakBonus;
    } else {
        int baseLoss = getKLoss(elo) / 2;
        return -baseLoss + participationBonus;
    }
}
```

---

## Simulations

### Points par match (adversaire équivalent, sans winstreak)

| Rang | Victoire | Défaite |
|------|----------|---------|
| Pokeball | +16 + 3 = **+19** | -4 + 3 = **-1** |
| Superball | +16 + 3 = **+19** | -6 + 3 = **-3** |
| Hyperball | +16 + 3 = **+19** | -12 + 3 = **-9** |
| Masterball | +16 + 3 = **+19** | -16 + 3 = **-13** |

---

### Simulation sur 30 matchs par rang

#### Pokeball (0-249 Elo)

| Winrate | Victoires | Défaites | Calcul | Total |
|---------|-----------|----------|--------|-------|
| 20% | 6 × +19 | 24 × -1 | +114 - 24 | **+90** ✓ |
| 33% | 10 × +19 | 20 × -1 | +190 - 20 | **+170** ✓ |
| 50% | 15 × +19 | 15 × -1 | +285 - 15 | **+270** ✓ |

**Winrate minimum pour stagner : ~5%**

---

#### Superball (250-599 Elo)

| Winrate | Victoires | Défaites | Calcul | Total |
|---------|-----------|----------|--------|-------|
| 20% | 6 × +19 | 24 × -3 | +114 - 72 | **+42** ✓ |
| 33% | 10 × +19 | 20 × -3 | +190 - 60 | **+130** ✓ |
| 50% | 15 × +19 | 15 × -3 | +285 - 45 | **+240** ✓ |

**Winrate minimum pour stagner : ~14%**

---

#### Hyperball (600-1049 Elo)

| Winrate | Victoires | Défaites | Calcul | Total |
|---------|-----------|----------|--------|-------|
| 20% | 6 × +19 | 24 × -9 | +114 - 216 | **-102** ✗ |
| 33% | 10 × +19 | 20 × -9 | +190 - 180 | **+10** ~ |
| 40% | 12 × +19 | 18 × -9 | +228 - 162 | **+66** ✓ |
| 50% | 15 × +19 | 15 × -9 | +285 - 135 | **+150** ✓ |

**Winrate minimum pour stagner : ~32%**

---

#### Masterball (1050+ Elo)

| Winrate | Victoires | Défaites | Calcul | Total |
|---------|-----------|----------|--------|-------|
| 33% | 10 × +19 | 20 × -13 | +190 - 260 | **-70** ✗ |
| 40% | 12 × +19 | 18 × -13 | +228 - 234 | **-6** ~ |
| 45% | ~14 × +19 | ~16 × -13 | +266 - 208 | **+58** ✓ |
| 50% | 15 × +19 | 15 × -13 | +285 - 195 | **+90** ✓ |

**Winrate minimum pour stagner : ~41%**

---

### Impact du Winstreak

Exemple : Joueur Masterball à 45% winrate avec des séries de victoires

**Sans winstreak :**
```
14 victoires × +19 = +266
16 défaites × -13 = -208
Total : +58 points
```

**Avec winstreaks (estimation réaliste) :**
```
6 victoires isolées : 6 × +19 = +114
4 victoires streak 2 : 4 × +21 = +84
4 victoires streak 3+ : 4 × +23 = +92
Total victoires : +290

16 défaites × -13 = -208
Total : +82 points (+24 grâce aux winstreaks)
```

---

### Simulation : 75% winrate → Masterball

**Question** : Combien de matchs pour passer de 250 Elo (Superball I) à 1050 (Masterball) avec 75% de winrate ?

**Méthode** : 500 simulations Monte Carlo, winrate fixe 75%, matchs équilibrés (50/50), ladder avec 5 sous-rangs par rang.

| Métrique | Valeur |
|----------|--------|
| **Moyenne** | **~53 matchs** |
| Minimum | ~33 matchs |
| Maximum | ~104 matchs |
| 25e-75e percentile | 47 - 58 matchs |

**Conclusion** : Avec 75% de winrate et le ladder par défaut (2-3 victoires en Pokeball, 6-7 en Masterball), il faut environ **50-55 matchs** pour atteindre Masterball depuis Superball I.

---

## Tableau Récapitulatif

### Winrates minimum par rang

| Rang | Pour stagner | Pour monter confortablement |
|------|--------------|----------------------------|
| Pokeball | ~5% | 20%+ |
| Superball | ~14% | 25%+ |
| Hyperball | ~32% | 40%+ |
| Masterball | ~41% | 50%+ |

### Philosophie du système

| Aspect | Design |
|--------|--------|
| Débutants | Très protégés, montent facilement |
| Joueurs réguliers | Récompensés par le bonus participation |
| Joueurs en série | Récompensés par le winstreak |
| Joueurs inactifs | Pénalisés par le decay |
| Top players | Doivent maintenir un bon niveau |

---

## Configuration

**Tous les paramètres du système ranked sont configurables** via le fichier `config/pokerp-strat/ranked.json`.

Voir **[CONFIG_RANKED.md](CONFIG_RANKED.md)** pour la documentation complète des paramètres.

### Paramètres principaux (valeurs par défaut)

| Catégorie | Paramètres |
|-----------|------------|
| **Rangs** | `ranks[]` : id, name, minElo, subRanks[] (sous-rangs optionnels) |
| **Elo** | `startingElo`, `eloScale`, `kGain`, `kLoss` (par rank id) |
| **Bonus** | `participationBonus`, `winstreakBonusPerWin`, `winstreakBonusMax` |
| **Decay** | `enabled`, `gracePeriodDays`, `rates`, `factors` (par rank id) |
| **Matchmaking** | `searchRadiusInitial`, `searchIntervalSeconds`, etc. |
| **Match** | `acceptTimeoutSeconds`, `abandonPenalty`, etc. |

L'implémentation doit **charger ces valeurs depuis la config** plutôt que d'utiliser des constantes en dur.

---

## Notes de Design

1. **Pourquoi K asymétrique ?**
   - Permet aux nouveaux joueurs de progresser même avec des défaites
   - Évite la frustration en début de partie
   - Les joueurs expérimentés ont prouvé leur niveau, les pertes normales sont justifiées

2. **Pourquoi un bonus de participation ?**
   - Récompense le temps investi
   - Permet une progression lente même à faible winrate
   - Encourage à jouer plutôt qu'à éviter les matchs

3. **Pourquoi le winstreak ?**
   - Récompense la régularité
   - Ajoute de l'excitation ("je ne veux pas casser ma série")
   - Aide les joueurs sous-classés à remonter rapidement

4. **Pourquoi le decay ?**
   - Maintient un classement actif et pertinent
   - Libère les places au sommet pour les joueurs actifs
   - Encourage le jeu régulier

5. **Pourquoi l'inactivité ne compte que les jours connectés ?**
   - Ne pénalise pas les joueurs absents (vacances, indisponibilité)
   - Pénalise ceux qui se connectent mais évitent le classé
   - Plus équitable et moins frustrant

6. **Pourquoi un decay progressif ?**
   - Donne une marge avant que la perte ne devienne forte
   - Évite une chute brutale après quelques jours
   - La progression (-2 → -5 → -15) reflète la gravité de l'inactivité prolongée
