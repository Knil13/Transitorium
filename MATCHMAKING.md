# Système de Matchmaking - PokeRP Strat

Ce document décrit le système de matchmaking pour les matchs classés.

---

## Vue d'ensemble

Le matchmaking combine plusieurs mécanismes :

1. **Élargissement progressif du rayon** : chaque joueur a son propre rayon de recherche qui s'élargit avec le temps (individuel)
2. **Intervalle fixe** : cycle de matchmaking toutes les 10 secondes
3. **Score mutuel** : satisfaction des deux joueurs + bonus d'attente
4. **Satisfaction progressive** : score basé sur la proximité Elo et les séries (winstreak/loss streak)
5. **Critères symétriques/asymétriques** : symétriques avant 90s, asymétriques après (match garanti)
6. **Limite de terrains** : si plus de paires que de terrains, priorité aux joueurs les plus anciens
7. **Matchmaking des placements** : joueurs Unranked dans la plage 0-530 Elo

---

## 1. Cycle de matchmaking (intervalle fixe)

### Principe

Le matchmaking s'exécute **toutes les 10 secondes** (configurable). À chaque cycle, on considère tous les joueurs en file et on forme les meilleures paires selon le score mutuel.

**Avantage** : permet de trouver des configurations optimales quand plusieurs joueurs arrivent proches dans le temps.

### Paramètres

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `intervalSeconds` | 10 | Intervalle entre chaque cycle de matchmaking |
| `guaranteedMatchThresholdSeconds` | 90 | Après ce délai : match garanti, haute priorité, critères asymétriques |

---

## 2. Critères de visibilité (symétriques / asymétriques)

Chaque joueur a un **rayon de recherche individuel** qui s'élargit avec le temps (voir section 6). Ce rayon détermine quels adversaires il peut "voir".

### Avant 90 secondes d'attente

**Critères symétriques** : les deux joueurs doivent se voir dans leur rayon respectif.

- A voit B (B dans le rayon de A) **et** B voit A (A dans le rayon de B) → match possible

### Après 90 secondes d'attente

**Critères asymétriques** : seul le joueur qui attend le plus longtemps impose sa plage.

- Si D attend depuis 95s et voit A dans son rayon → match possible (même si A ne voit pas encore D dans son propre rayon)

### Garantie à 90 secondes

Le joueur qui attend depuis 90s+ entre en **haute priorité** : on cherche activement un match pour lui, en utilisant son rayon (asymétrique) si nécessaire.

---

## 3. Score de satisfaction

### Principe

Chaque paire (A, B) reçoit un score combinant la satisfaction des deux joueurs. Plus le score est élevé, plus la paire est prioritaire.

### Formules de satisfaction (par joueur, 0-10)

#### Winstreak (3+ victoires)

Préfère un adversaire **plus fort**, mais le **plus proche** parmi les plus forts (pas le plus fort possible).

```
écart = elo_adv - elo_joueur

SI écart > 0 (adv plus fort):
    satisfaction = 10 - écart / 100   // plus proche = mieux
SINON (adv plus faible ou égal):
    satisfaction = 5   // neutre, pas de bonus
```

#### Loss streak (3+ défaites)

Préfère un adversaire **plus faible**, mais le **plus proche** parmi les plus faibles (pas le plus faible possible).

```
écart = elo_joueur - elo_adv

SI écart > 0 (adv plus faible):
    satisfaction = 10 - écart / 100   // plus proche = mieux
SINON (adv plus fort ou égal):
    satisfaction = 5   // neutre, pas de bonus
```

#### Neutre (pas de série significative)

Préfère l'adversaire le **plus proche** en Elo.

```
écart = |elo_adv - elo_joueur|
satisfaction = 10 - écart / 100
```

### Paramètre de scale

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `satisfactionEloScale` | 100 | Diviseur pour le calcul progressif (écart/100) |

---

## 4. Score final d'une paire

```
score_paire(A, B) = satisfaction_A(B) + satisfaction_B(A) + bonus_attente - malus_rematch
```

### Bonus d'attente

Le joueur qui attend le plus longtemps dans la paire apporte un bonus au score. Le bonus **augmente par paliers** à intervalles réguliers.

```
temps_attente = min(temps_attente_A, temps_attente_B)  // en secondes
palier = floor(temps_attente / waitTimeBonusStepSeconds)
bonus_attente = palier × waitTimeBonusStepPoints
```

**Exemple** (step = 30s, points = 1) :
| Attente | Palier | Bonus |
|---------|--------|-------|
| 0-29s | 0 | 0 |
| 30-59s | 1 | 1 |
| 60-89s | 2 | 2 |
| 90-119s | 3 | 3 |

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `waitTimeBonusStepSeconds` | 30 | Délai entre chaque augmentation du bonus (en secondes) |
| `waitTimeBonusStepPoints` | 1 | Points ajoutés à chaque palier |

### Malus rematch

Si les deux joueurs se sont affrontés récemment, on applique un malus (tiebreaker pour favoriser la diversité).

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `rematchPenalty` | -2 | Malus au score si affrontés récemment |
| `rematchPenaltyWindowMinutes` | 15 | Fenêtre de temps pour le malus rematch |

---

## 5. Ordre de départage

Quand plusieurs paires ont le même score (ou plusieurs adversaires le même score de satisfaction), on applique les tiebreakers dans cet ordre :

| Priorité | Critère | Description |
|----------|---------|-------------|
| 1 | **Score de satisfaction** | Critère principal (satisfaction mutuelle + bonus attente - malus rematch) |
| 2 | **Temps d'attente** | Priorité au joueur qui attend le plus longtemps |
| 3 | **Proximité Elo** | Parmi les ex-aequo, adversaire le plus proche en Elo |
| 4 | **Aléatoire** | Si toujours égal, tirage au sort |

---

## 6. Élargissement progressif du rayon de recherche

### Principe

**Chaque joueur possède son propre rayon de recherche**, calculé individuellement selon son temps d'attente. Le rayon s'élargit progressivement tant que le joueur reste en file.

- Joueur A entre à t=0 → rayon ±100
- Joueur B entre à t=20 → rayon ±100
- À t=30 : A a rayon ±200, B a rayon ±100
- À t=60 : A a rayon ±300, B a rayon ±200

### Formule du rayon par joueur

```
temps_attente = maintenant - joueur.queueJoinTime
palier = min(floor(temps_attente / searchIntervalSeconds), searchMaxIntervals)
rayon_joueur = searchRadiusInitial + (palier × searchRadiusStep)
```

### Évolution du rayon

| Palier | Délai d'attente | Rayon | Exemple (joueur 500 Elo) |
|--------|-----------------|-------|---------------------------|
| 0 | 0-29s | ±100 | Candidats : 400 - 600 |
| 1 | 30-59s | ±200 | Candidats : 300 - 700 |
| 2 | 60-89s | ±300 | Candidats : 200 - 800 |
| 3 | 90s+ | ±400+ | Candidats : toute la file |

### Rôle dans la visibilité

Le rayon de chaque joueur détermine **quels adversaires il peut voir** :

- **A voit B** si B.elo est dans [A.elo - rayon_A, A.elo + rayon_A]
- **B voit A** si A.elo est dans [B.elo - rayon_B, B.elo + rayon_B]

Les critères symétriques/asymétriques (section 2) utilisent ces rayons pour déterminer si une paire est éligible.

### Paramètres

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `searchRadiusInitial` | 100 | Rayon initial (±100 pts) à l'entrée en file |
| `searchRadiusStep` | 100 | Élargissement par palier (en pts) |
| `searchIntervalSeconds` | 30 | Délai entre chaque palier d'élargissement (en secondes) |
| `searchMaxIntervals` | 3 | Nombre de paliers avant d'accepter n'importe qui (rayon max) |

---

## 7. Matchmaking des placements (Unranked)

Les joueurs **Unranked** (en matchs de placement) ont des règles spécifiques :

| Règle | Description |
|-------|-------------|
| **Plage Elo** | Adversaires uniquement dans **0 - 530** Elo (Pokeball I à Superball V) |
| **Priorité** | Idéalement adversaires autour de **265** Elo (milieu de la plage) |
| **Autres Unranked** | Prioriser les autres joueurs en placement quand c'est possible |

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `placementSearchMinElo` | 0 | Elo minimum des adversaires |
| `placementSearchMaxElo` | 530 | Elo maximum des adversaires |
| `placementPreferredElo` | 265 | Elo cible pour prioriser |

---

## 8. Limite de terrains (matchs simultanés)

### Principe

Le nombre de matchs simultanés peut être limité (`maxSimultaneousMatches`). Chaque match en cours occupe un **terrain**.

- **maxSimultaneousMatches = -1** : illimité (pas de vérification)
- **maxSimultaneousMatches >= 0** : limite active (ex: 5 terrains max)

### Vérification en premier

**Avant tout calcul** : si aucun terrain n'est disponible (tous occupés), le cycle de matchmaking est **interrompu immédiatement**. Aucun calcul de paires, aucun tri — on attend le prochain cycle.

### Priorité quand plus de paires que de terrains

Si plusieurs paires sont formées mais qu'il y a **moins de terrains disponibles** que de paires :

- Les paires avec les **joueurs les plus anciens** partent en priorité
- Les autres paires restent en file et seront **reconsidérées au prochain cycle**

### Exemple

- 3 terrains disponibles
- 5 paires formées (triées par score puis temps d'attente)
- Les 3 premières paires (joueurs les plus anciens) → matchs lancés
- Les 2 autres paires → restent en file, recalculées au prochain cycle (10s plus tard)

---

## 9. Algorithme complet

### Pseudocode du cycle de matchmaking

```
Toutes les intervalSeconds secondes:

0. VÉRIFICATION EN PREMIER : terrains disponibles
   SI maxSimultaneousMatches >= 0:  // Limite définie
     terrains_disponibles = maxSimultaneousMatches - matchs_en_cours
     SI terrains_disponibles <= 0:
         FIN DU CYCLE  // Pas de calcul, pas de tri
   SINON:  // maxSimultaneousMatches = -1 (illimité)
     terrains_disponibles = infini

1. Récupérer tous les joueurs en file (non en combat)

2. Pour chaque paire possible (A, B):
   - Calculer rayon_A et rayon_B (selon temps d'attente de chacun)
   - Vérifier visibilité : A voit B (B dans rayon_A) et B voit A (A dans rayon_B)
     - Avant 90s : symétrique (les deux doivent se voir)
     - Après 90s : asymétrique (le joueur 90s+ impose son rayon)
   - Calculer score_paire(A, B) = satisfaction_A(B) + satisfaction_B(A) + bonus - rematch
   - Stocker la paire avec son score

3. Trier les paires par score (décroissant), puis par tiebreakers:
   - Temps d'attente (joueur le plus ancien)
   - Proximité Elo
   - Aléatoire

4. Pour chaque paire (en ordre), jusqu'à épuisement des terrains:
   SI terrains_disponibles <= 0: STOP
   SI les deux joueurs sont encore disponibles:
     - Lancer le match
     - Retirer A et B du pool
     - terrains_disponibles--
   // Les paires non lancées restent en file, recalculées au prochain cycle
```

### Calcul de la satisfaction

```
Fonction satisfaction(joueur, adversaire):
    écart_adv = adversaire.elo - joueur.elo
    
    SI joueur.winstreak >= 3:
        SI écart_adv > 0:
            return max(0, 10 - écart_adv / satisfactionEloScale)
        SINON:
            return 5
    
    SINON SI joueur.lossstreak >= 3:
        SI écart_adv < 0:
            return max(0, 10 - (-écart_adv) / satisfactionEloScale)
        SINON:
            return 5
    
    SINON:  // Neutre
        return max(0, 10 - |écart_adv| / satisfactionEloScale)
```

---

## 10. Récapitulatif des critères de satisfaction

| # | Critère | Type | Formule / logique |
|---|---------|------|-------------------|
| 1 | Winstreak | Situation | Adv plus fort → 10 - écart/100 ; sinon → 5 |
| 2 | Loss streak | Situation | Adv plus faible → 10 - écart/100 ; sinon → 5 |
| 3 | Neutre | Situation | 10 - \|écart_elo\| / 100 |
| 4 | Rematch récent | Malus | -1 ou -2 si affrontés dans la fenêtre |
| 5 | Attente | Bonus | +N par palier (toutes les 30s) du joueur le plus ancien |
| 6 | Temps d'attente | Tiebreaker | Plus long = priorité |
| 7 | Proximité Elo | Tiebreaker | Plus petit écart = meilleur |
| 8 | Aléatoire | Tiebreaker | Dernier recours |

---

## 11. Données à suivre par joueur

| Champ | Description |
|-------|-------------|
| `winstreak` | Victoires consécutives |
| `lossstreak` | Défaites consécutives |
| `queueJoinTime` | Heure d'entrée en file |
| `isUnranked` | En matchs de placement |
| `lastOpponentId` | Dernier adversaire affronté |
| `lastMatchTime` | Heure du dernier match (pour rematch penalty) |

---

## 12. Cas particuliers

### File vide

Si aucun joueur n'est en file, aucun match n'est lancé. Le prochain cycle attendra de nouveaux joueurs.

### Limite de combats simultanés

Voir section 8. Si aucun terrain n'est disponible, le cycle est interrompu sans calcul. Les joueurs déjà en combat ne sont pas candidats pour le matchmaking.

### Joueur à 90s+ d'attente

Priorité haute : on cherche un match en priorité pour ce joueur, avec critères asymétriques si nécessaire.

---

## 13. Configuration

Voir [CONFIG_RANKED.md](CONFIG_RANKED.md) section `matchmaking` pour tous les paramètres.
