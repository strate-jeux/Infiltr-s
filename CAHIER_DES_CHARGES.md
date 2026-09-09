# INFILTRÉS — Diaporama interactif pour l'animateur

**Cahier des charges — Version 2** *(architecture inchangée par le brief v3.0 ; modèle de données, règles et interface mis à jour ponctuellement ci-dessous pour rester alignés avec le Règlement v3.0 et le brief de mise à jour du diaporama v3.0 — séquestre supprimé, carte unique par joueur, plafond de révocations à 2/manche, refonte de la phase de défi en 4 temps.)*

À remettre à Claude Code, lot par lot, avec le Règlement v3.0. Cette version 2 remplace l'architecture double fenêtre (Scène/Console) par une **fenêtre unique**, et actualise l'habillage visuel sur la base des cartes physiques du jeu. L'arithmétique et les tests d'acceptation (§3, §4, §8) restent inchangés dans leur principe (millièmes entiers, plus fort reste) sauf mention contraire ci-dessous. Le document de référence des règles est le Règlement v3.0 ; en cas de contradiction, le règlement prime.

---

## 1. Objet et principes directeurs

Application web pilotant une partie complète d'INFILTRÉS : attribution et affichage du capital, déroulé des phases, votes pondérés, mouvements de capital vers le Fonds Meridian, vérification automatique des conditions de victoire, habillage narratif. Un seul opérateur : l'animateur, qui **clique — il ne saisit jamais de texte libre**. Tout le contenu (questions, QCM, réponses, textes narratifs, durées de chronomètre) est préchargé dans le fichier.

**Principes non négociables**

- **Un seul fichier HTML autonome** (HTML + CSS + JS embarqués, aucune dépendance CDN, aucun appel réseau, images encodées en base64 dans le fichier). Doit fonctionner ouvert en local (`file://`) comme hébergé sur GitHub Pages, entièrement hors ligne une fois chargé. Le camembert est dessiné en SVG maison — pas de bibliothèque graphique externe.
- **Fenêtre unique, projetée directement.** Il n'y a plus de séparation Scène/Console : un seul écran, celui que l'animateur pilote, est celui qui est projeté. La confidentialité ne repose plus sur une séparation technique mais sur le déroulé du jeu lui-même (yeux fermés pendant la nuit, saisie des rôles faite avant branchement du vidéoprojecteur — voir §2).
- **Sauvegarde continue :** l'état complet de la partie est enregistré dans le localStorage après chaque action. À l'ouverture, si une partie est en cours, proposer « Reprendre la partie ». En complément : export et import de l'état en fichier JSON téléchargeable (secours en cas de changement de machine).
- **Annulation :** chaque action de jeu empile un instantané de l'état. Un bouton « ◀ Annuler » discret, accessible en permanence, utilisable en cascade jusqu'au début de la partie.
- **Mode masqué :** un bouton permet à tout moment de cacher l'écran (calque plein écran, fond uni + texte « Préparation en cours »), en filet de sécurité si le vidéoprojecteur est déjà branché pendant une saisie sensible.
- **Mode simulation** intégré pour la recette : voir §8.

---

## 2. Architecture

### 2.1 Principe général

Une seule page HTML, un seul état de jeu, un déroulé linéaire d'écrans que l'animateur fait avancer par ses clics. La structure visuelle commune à la quasi-totalité des écrans :

- **Bandeau haut** : numéro de manche, phase en cours, nombre de joueurs actifs.
- **Camembert du capital**, ~40 % de la hauteur, centré en permanence (v3.0 : agrandi depuis le petit widget compact), qui s'anime (transition 1200 ms, cubic-bezier) à chaque mouvement de capital.
- **Carte centrale** : le contenu propre à l'écran courant (question, choix, résultat...).
- **Bandeau « Joueurs »** en bas : un bouton par joueur, visuellement grisé + icône si sorti (sabotage ou révocation — v3.0 : plus d'état « gelé » intermédiaire, cf. § Modèle de données).
- **Historique** dépliable, journal horodaté de tous les événements.
- **Contrôles animateur**, discrets et toujours accessibles : ◀ Annuler · Masquer.

### 2.2 Confidentialité sans double fenêtre

Le règlement impose qu'aucune information secrète (camps, rôles, cibles nocturnes) n'apparaisse publiquement en dehors des moments de révélation prévus. Sans double fenêtre, cette confidentialité est assurée par :

1. **La saisie des rôles se fait avant de brancher le vidéoprojecteur** (réflexe par défaut).
2. **Le bouton « Masquer »** (calque plein écran neutre) reste disponible à tout instant en filet de sécurité, y compris pendant la partie si un rôle doit être corrigé.
3. **Pendant la phase nocturne**, la confidentialité n'est plus technique mais physique : les joueurs honnêtes ferment les yeux ; seul le rôle appelé ouvre les yeux pour agir, exactement comme au Loup-Garou classique. Aucun nom n'est jamais affiché à l'écran pour ces appels (voir §5.6) — l'écran reste neutre, ce sont les joueurs eux-mêmes qui se reconnaissent.

### 2.3 Le camembert du capital

- Tranches **bleues** : tous les joueurs actifs, camp non révélé (l'écrasante majorité du temps).
- Tranche **rouge** : joueur infiltré, uniquement une fois son camp révélé (sabotage ou révocation).
- Tranche **noire** : Fonds Meridian.
- Le camembert ne doit jamais laisser deviner un camp avant sa révélation officielle : tant qu'un joueur n'est pas révélé, sa tranche reste bleue quel que soit son camp réel.

---

## 3. Modèle de données

*(Reprend le modèle de la v1, avec les ajustements suivants.)*

| Entité | Champs | Notes |
| --- | --- | --- |
| Partie | manche (0–5), phase, revocationsAdopteesParManche (compteur 0–2, par manche — v3.0 : plafond porté à 2), résolutionsEnCours (liste de 0 à 3 noms), paliers Meridian Annoncés [15, 25, 35, 45], journal des événements, pile d'annulation | Sérialisable en JSON (sauvegarde, export, simulation). |
| Joueur | prénom, service (RH / Commercial / Finance), part (en millièmes de %, **non éditable manuellement**), statutCapital (actif / sorti — v3.0 : plus d'état « gelé », deux états seulement), motifSortie (sabote / revoque, renseigné à la sortie), **carte** (associé / DRH / DAF / Juriste / infiltré — SECRET, champ unique v3.0, l'Avocat(e) a disparu et son pouvoir est repris par le Juriste), révélé (bool), flags : protégéCetteNuit, protégéNuitPrécédente, miseÀPiedManche, miseAPiedAnnoncee (bool) | **Un seul champ d'identité : le camp se déduit entièrement de la carte** (carte = « infiltré » → camp adverse, toute autre carte → camp honnête) — plus de champ camp séparé, plus de flag mandataireDuJuriste (la procuration disparaît avec le séquestre). Le Président est un attribut de Partie (`présidentActuel`), assigné en cours de partie et non plus à la Préparation. |
| Meridian | part (en millièmes de %) | Démarre à 1,000 % exactement. |
| Défi | manche, service, choix (A/B/C/D préchargés), réponseCorrecte (secrète, préchargée), réponseÉquipe (saisie par tap animateur), réussi (dérivé automatiquement) | Les 15 défis et leur clé de réponse sont préchargés dans le fichier (annexe corrigé, à fusionner depuis la version existante du cahier des charges v1.1). |

**Arithmétique :** tous les calculs internes se font en millièmes de pourcent (entiers), jamais en flottants. Affichage arrondi au dixième de %. Invariant permanent : somme des parts des joueurs non révoqués + Meridian = 100,000 % exactement. Restes d'arrondi attribués par la méthode du plus fort reste. Toute violation déclenche une alerte visible.

**Durées de chronomètre** (préréglées, non modifiables en cours de partie) :

| Manches | Durée du défi |
| --- | --- |
| 1 et 2 | 5 min |
| 3 et 4 | 15 min |
| 5 | 7 min |

Débat en AG : 5:00 fixe, toutes manches confondues.

---

## 4. Règles de calcul — le moteur

*(Identique à la v1, sauf les points suivants.)*

### 4.1 Attribution initiale des parts

- Les joueurs se partagent 99,000 % ; Meridian détient 1,000 %.
- Tirage aléatoire légèrement inégal (0,65× à 1,35× la moyenne), normalisé à 99,000 %.
- Un seul bouton **« Tirer les parts au sort »**, re-cliquable pour relancer le tirage (plus de bouton « Retirer au sort » séparé, redondant). **Aucune édition manuelle des parts** : l'invariant est garanti dès le tirage, sans risque de saisie erronée.

### 4.2 Sabotage (phase nocturne)

Inchangé : cible sélectionnée parmi les actifs ; si protégée, « nuit calme » ; sinon gel + révélation du **rôle** (plus de mention de camp séparée, voir §3) au bilan du matin. Exception Juriste inchangée (procuration).

**Ajout — avantage des Infiltrés :** juste après le choix de la cible, les Infiltrés (seuls éveillés à ce moment) voient s'afficher les réponses correctes des 3 défis de la manche suivante (voir §5.6, étape « Infiltrés »).

### 4.3 Révocation — vote d'AG par résolutions successives

*(Remplace intégralement le mécanisme de la v1 : plus de « Motion » avec proposeur/second, plus de vote en CA.)*

- Pendant le **débat** (5:00), l'animateur peut inscrire **jusqu'à 3 noms** à l'ordre du jour, dans l'ordre où les accusations émergent — sans exigence de soutien formalisé.
- Chaque nom devient une **résolution** (« Révocation de [Nom] »), votée **une à la fois, dans l'ordre d'inscription**, par la grille pondérée Pour / Contre / Abstention (abstention = compte comme contre, cf. règle des 50 % des actions votantes totales).
- Dès qu'une résolution dépasse strictement 50 % des actions votantes, elle est adoptée : révocation immédiate (sortie du capital en 100 % / 0 %, cf. § Règle de sortie unique), révélation de la **carte** du révoqué, redistribution intégrale au prorata entre les joueurs actifs restants — jamais à Meridian.
- Si une résolution échoue, on passe à la suivante. Si les 3 échouent (ou s'il y a eu moins de 3 noms), la manche se termine sans révocation.
- **Jusqu'à 2 révocations par manche** (v3.0 : plafond porté de 1 à 2 — un rejet ne consomme pas ce plafond, seule une adoption le fait ; verrouillage dès que la 2ᵉ résolution de la manche est adoptée).
- Si le révoqué est le Président : succession désignée immédiatement après (parmi les joueurs restants), annoncée à l'écran.
- **Vérification de palier** (voir §4.5) effectuée immédiatement après toute redistribution consécutive à une révocation.

### 4.4 Alerte et Panique — mécanique QCM

*(Remplace la saisie manuelle « réussi/échoué » de la v1.)*

- Pour chacun des 3 défis de la manche (un par service), l'animateur **sélectionne la réponse donnée par l'équipe** parmi les choix A/B/C/D préchargés (entité Défi, §3).
- Le système compare automatiquement à la réponse correcte secrète et en déduit réussi/échoué.
- Bilan des 3 défis : 3 réussis → **Indice**, affiché immédiatement en plein écran, avant le vote de la même manche ; 2/1 → rien ; 1/2 → **Alerte** (4 000 points, constante configurable) ; 0/3 → **Panique** (8 000 points, constante configurable).
- Transfert vers Meridian appliqué et annoncé au Bilan du matin, avec animation du camembert : chaque joueur de l'assiette (actifs + Juriste saboté) cède part_i × montant ÷ somme(assiette), au plus fort reste.
- **Vérification de palier** (voir §4.5) effectuée immédiatement après cette animation, avant de passer à l'étape suivante du Bilan du matin.
- Indices proposés à l'animateur (il en choisit un), inchangé par rapport à la v1.

### 4.5 Paliers et vérification de la victoire

- **Paliers Meridian (15/25/35/45 %)** : vérifiés et annoncés **immédiatement après chaque mouvement de capital** — fin d'une résolution de révocation adoptée (§4.3), ou communiqué Meridian Alerte/Panique (§4.4). Chaque palier ne s'annonce qu'une seule fois par partie.
- **Vérification de victoire** : à chaque Bilan du matin (nuits 1 à 5), si somme des parts des infiltrés non révoqués + part Meridian > 50,000 % → écran de victoire Meridian. À tout moment, si tous les infiltrés sont révoqués → victoire honnête anticipée. Après le Bilan du matin de la nuit 5, si aucune condition n'est atteinte → chevalier blanc, victoire honnête finale.

---

## 5. Écrans (fenêtre unique)

### 5.1 Préparation (avant de brancher le vidéoprojecteur)

Assistant pas à pas en 4 étapes (retour de test) — un seul écran de configuration visible à la fois, jargon technique purgé de l'interface animateur. Bouton **Précédent** actif dès l'étape 2, qui revient à l'étape précédente sans perdre les données déjà saisies (réduction du nombre de joueurs → troncature confirmée si des données seraient perdues ; augmentation → champs vides ajoutés en fin de liste). Le panneau entier est démonté du DOM une fois la partie lancée (voir §6, jamais un simple `display:none`).

**Étape 1 — Nombre de joueurs**
Liste déroulante uniquement (8 à 18, pas de saisie libre). Aucun autre champ visible à cette étape.

**Étape 2 — Prénoms**
Un champ prénom + un menu service (RH / Commercial / Finance, modifiable) par joueur, nombre de champs fixé par l'étape 1. Bouton « Remplir avec des prénoms fictifs ».

**Étape 3 — Rôles**
Phrase calculée automatiquement : *« Pour [N] joueurs, désignez [⌊N/3⌋] infiltrés. »* Bouton **« Attribuer les rôles au hasard »** (symétrique au tirage des parts) : distribue les 3 cartes à pouvoir à 3 joueurs distincts, puis les infiltrés parmi les joueurs restants, répartis le plus équitablement possible entre les 3 services. Sinon, un menu déroulant par joueur : Associé (par défaut) · DRH · DAF · Juriste · Infiltré (v3.0 : l'Avocat(e) a disparu, son pouvoir de protection est repris par le Juriste). Chaque carte à pouvoir ne peut être attribuée qu'une fois (retirée des menus dès qu'elle est prise). Compteur en direct *« Infiltrés désignés : X / [⌊N/3⌋] »*. Bouton **« Suivant »** inactif tant que le compte n'est pas exact (message explicite : nombre d'infiltrés manquants ou en trop), activé automatiquement à l'égalité, regrisé si on redescend en dessous.

**Étape 4 — Parts sociales**
Tableau des parts (résultat du tirage) + ligne Meridian (1,000 %). Un seul bouton **« Tirer les parts au sort »**, re-cliquable pour relancer le tirage. Total affiché (toujours 100,000 %, aucune édition manuelle possible). Bouton **« Lancer la partie »** inactif tant que les parts n'ont pas été tirées.

Pas de saisie du Président à cette étape (voir §5.4).

### 5.2 Manche 0 — Lettre d'intention et mise en place

1. **Lettre d'intention** : mise en scène « courrier », texte intégral (Règlement v2) affiché progressivement via « Suivant ». Au dernier paragraphe, apparition animée de la tranche Meridian (1 %) sur le camembert.
2. **Reconnaissance des Infiltrés** : *« Les Infiltrés ouvrent les yeux. C'est le moment de se rassembler pour préparer le plan de sabotage de la semaine. »* Écran neutre, sans aucun nom affiché — les infiltrés se reconnaissent physiquement entre eux. Bouton « Suivant » quand l'animateur juge que c'est fait. *« Les Infiltrés se rendorment. »*
3. **Réponses de la Manche 1** : 3 cartes A/B/C/D (une par service), minimum 3 secondes chacune, avec bouton « Suivant » discret pour accélérer si besoin.
4. **Reconnaissance des cartes à pouvoir**, un par un, ordre fixe : Juriste → DAF → DRH. Pour chacun : *« [Carte] ouvre les yeux. »* (v3.0 : sous-titre « Prenez connaissance de votre rôle » supprimé) → Suivant → *« [Carte] se rendort. »* Écran neutre, sans nom affiché. (Le Président n'est pas concerné : rôle de jour, sans carte à connaître en secret.)
5. **Réveil général** : *« NOVENTIS se réveille ! Ouvrez les yeux. »* → enchaîne vers la Manche 1 (Phase de travail).

### 5.3 Tableau de bord (écran par défaut)

Camembert permanent + dernier événement en rappel. Bandeau Joueurs en bas (statut visuel). Badge Président si désigné. Un seul bouton **« Suivant »** qui enchaîne automatiquement la séquence programmée de la manche. Un « ◀ Retour » discret à côté, pour rattraper un clic en trop.

### 5.4 Phase de travail

- Titre « Manche X — Phase de travail », rappel des 3 services.
- Chronomètre préréglé selon la manche (§3), déclenché automatiquement.
- Pour chaque service, le QCM du défi de la manche s'affiche ; l'animateur tape la réponse donnée par l'équipe (aucune saisie libre).
- Bouton « Valider les résultats » → calcul automatique (§4.4).
- Si Indice (3/3) : rupture immédiate, écran plein écran de l'indice choisi.
- Sinon : retour au tableau de bord.

### 5.5 Conseil & Assemblée générale

**a) Débat (5:00)**
Titre : *« Le Conseil délibère. »* Sous-titre permanent : *« Le Président du conseil retient jusqu'à 3 noms qui seront soumis au vote de l'Assemblée Générale. »* L'animateur tape sur les joueurs accusés au fil du débat pour les ajouter à l'ordre du jour (compteur *« X / 3 noms retenus »*, retrait possible avant la fin du chrono).

**b) Vote, résolution par résolution**
Titre : *« Résolution [n] — Révocation de [Nom] »*. Sous-titre : *« Chaque actionnaire vote selon son nombre de parts. »* Grille pondérée Pour / Contre / Abstention (voir §5.7 pour le mode de saisie), barre de progression en direct, seuil 50 % marqué.
- Adoptée : *« Résolution adoptée — [Nom] est révoqué. »* → révélation du rôle + animation de redistribution + vérification de palier → fin du vote, résolutions suivantes non votées.
- Rejetée : *« Résolution rejetée — [Nom] reste en fonction. »* → passage automatique à la résolution suivante.
- Aucune résolution adoptée (3 échecs ou moins de 3 noms) : *« Aucune résolution n'a été adoptée. Le conseil reste inchangé. »*

### 5.6 Séquence de nuit

Écran neutre entre chaque appel, aucun nom jamais affiché pour les appels eux-mêmes.

1. **« La nuit tombe »** (v3.0) : écran assombri, camembert désaturé, texte de scénario propre à la manche, aucun bouton visible — l'animateur avance à la barre d'espace.
2. *« Le Juriste se réveille. Qui protège-t-il/elle cette nuit ? »* → grille des joueurs actifs (auto-protection autorisée depuis la v3.0 ; exclusion uniquement de la cible protégée la nuit précédente) → tap + Confirmer → *« Le Juriste se rendort. »*
3. *(si applicable)* *« Le DAF se réveille. Sur qui exerce-t-il son droit à l'information ? »* → tap sur un joueur → encart avec la carte exacte révélée à l'animateur seul → « Fermer » → *« Le DAF se rendort. »*
4. *(v3.0 : chaque nuit, plus 1×/partie)* *« La DRH se réveille. Qui met-elle à pied ? »* → tap → Confirmer → *« La DRH se rendort. »*
5. *« Les Infiltrés se réveillent. Qui sabotent-ils ? »* → grille des actifs → tap + Confirmer → *« Les Infiltrés se rendorment. »*
6. **Réponses de la manche suivante** *(dès la Manche 1, avant chaque nouvelle phase de travail)* : écran d'annonce seul, puis les 3 réponses affichées **ensemble** dans une grille (v3.0 : plus de cascade carte par carte), clic pour les cacher une fois lues.
7. **« NOVENTIS se réveille ! »** — flash blanc 120 ms puis retour sur 600 ms (v3.0) → enchaîne vers le Bilan du matin.

### 5.7 Vote pondéré — mode de saisie

Grille à bascule : chaque joueur votant est un seul bouton, qui fait défiler Abstention (état par défaut, gris) → Pour (vert) → Contre (rouge) → Abstention à chaque tap. Un tap suffit dans la majorité des cas. Le mandataire du Juriste saboté vote à la place du Juriste (parts comptées normalement).

### 5.8 Bilan du matin

1. **Résultat du sabotage** : si réussi, scénario narratif (Règlement v2, un des 5, un par manche) puis révélation du **rôle** du saboté (modale, style « [Prénom] était... [Rôle] », sans mention de camp). Si la cible était protégée : *« Nuit calme. »*
2. **Communiqué Meridian** *(si Alerte ou Panique)* : texte du communiqué (Règlement v2), animation du camembert, puis vérification et annonce de palier si franchi à cet instant.
3. **Vérification de victoire** : calcul silencieux. Si la partie continue → retour au tableau de bord (manche suivante). Si une condition de victoire est atteinte → transition directe vers l'écran de fin.

### 5.9 Écrans de fin

Sobre et dramatique : l'annonce (une des 3 variantes — victoire Meridian, victoire honnête anticipée, chevalier blanc) + camembert final. Pas de récapitulatif affiché par défaut. Bouton **« Voir le récapitulatif »** qui ouvre la chronologie complète des événements (pour le débriefing), avec export possible.

---

## 6. Contrôles de l'animateur

*(Remplace « La Console animateur » de la v1 — les mêmes fonctions existent, intégrées à la fenêtre unique plutôt que dans un panneau séparé.)*

- **◀ Annuler** : annulation en cascade, disponible en permanence, discrète.
- **Masquer** : calque plein écran neutre, disponible en permanence (voir §2.2).
- **Corrections** : édition manuelle d'un rôle en cours de partie possible via Masquer → modification → démasquer ; changement de Président à tout moment ; bascule manuelle d'écran en cas de besoin (accès discret, non mis en avant).
- **Modifier la configuration** (retour de test) : bouton visible uniquement en cours de partie, avec confirmation (« interrompt la partie en cours »). Remonte l'assistant de préparation (§5.1), pré-rempli depuis la partie en cours, pour un dépannage en profondeur (nombre de joueurs erroné, refonte complète des rôles…). Le panneau de préparation est démonté du DOM dès le lancement de la partie — pas un simple `display:none` — et ne peut donc pas réapparaître par un défilement ou raccourci accidentel ; ce bouton est l'unique façon d'y revenir.
- **Journal** : chaque événement horodaté (sabotages, votes avec détail par résolution, mouvements de capital, indices révélés). Exportable en texte pour le débriefing.

---

## 7. Habillage

### 7.1 Palette

Fond général de l'application : **noir/anthracite très sombre** (confort visuel sur 2–3 h de projection).

- **Bleu Infiltrés** — dominante des cartes de contenu et des boutons d'action (≈ `#2F6FEB`/`#3B79F0`, à ajuster sur le code exact de l'imprimeur si disponible).
- **Noir** — texte, contours des illustrations, tranche Meridian sur le camembert.
- **Blanc cassé** — fond des cartes de contenu (rôle, résolution, révélation), bordure bleue épaisse, coins arrondis — reprend le format des cartes physiques « VOUS ÊTES : [RÔLE] ».
- **Rouge** — uniquement les tranches de camembert des infiltrés révélés (jamais avant révélation).

### 7.2 Typographie

Grasse, arrondie, capitales pour les titres — type Poppins ExtraBold ou Nunito Black (polices libres, embarquées en base64 ou en `@font-face` local, pas de CDN).

### 7.3 Composants

- Cartes de contenu : fond blanc, bordure bleue épaisse, coins arrondis.
- Boutons d'action : bleu plein, texte blanc gras, coins arrondis.
- Camembert : bleu (actifs) / rouge (infiltrés révélés) / rouge très sombre (Meridian, v3.0 — remplace le noir), transitions 1200 ms cubic-bezier(.4,0,.2,1) (v3.0, remplace 800 ms easeInOutQuad).
- Illustrations : dessin au trait noir façon bâtonnet, fond transparent, style des cartes physiques (fichiers fournis, à intégrer en base64).

### 7.4 Illustrations — table de correspondance

| Élément | Illustration |
| --- | --- |
| Accueil / Préparation | Logo Infiltrés (poignée de main) |
| Associé honnête | Bonhomme + gratte-ciel |
| Infiltré | Bonhomme lunettes + cartes/jetons |
| DRH | Bonhomme au bureau, piles de dossiers |
| DAF | Bonhomme + écran, courbe qui chute |
| Juriste | Bonhomme au contrat + stylo, grand sourire (reprend désormais aussi le pouvoir de protection par référé, v3.0) |
| Président | Pas d'illustration dédiée (badge de statut uniquement) |
| Reconnaissance des Infiltrés (Manche 0) | Groupe en lunettes noires |
| Débat (Conseil) | Deux mégaphones face à face |
| Révocation (résolution adoptée) | Coup de pied + porte-documents qui vole |
| Sabotage — Manche 1 | Duel au marteau |
| Sabotage — Manche 2 | Brouette de dossiers envolés |
| Sabotage — Manche 3 | Écran boursier qui s'effondre |
| Sabotage — Manche 4 | Homme qui court avec une pile de livres |
| Sabotage — Manche 5 | Loupe sur un document |
| Victoire (les 3 variantes) | Trophée |

### 7.5 Textes

Textes à intégrer tels quels depuis le Règlement v2 : lettre d'intention, 5 scénarios de sabotage, 2 communiqués Meridian, 4 annonces de palier, 3 textes de fin de partie. Sons optionnels et désactivables (gong de nuit, notification de communiqué), inchangé par rapport à la v1.

Tout l'habillage est ajouté sans modifier le moteur : les fonctionnalités des lots précédents doivent rester fonctionnelles à tout moment.

---

## 8. Mode simulation et tests d'acceptation

*(Inchangé par rapport à la v1 — voir document d'origine pour le détail des tests T1 à T5. Le mode simulation reste intégré aux contrôles de l'animateur, §6.)*

---

## 9. Découpage en lots — ordre impératif

| Lot | Périmètre | Critère de recette |
| --- | --- | --- |
| **Lot 1 — Moteur de capital** | Modèle de données (avec entité Défi), arithmétique en millièmes, attribution initiale (sans édition manuelle), les 4 actions (sabotage, révocation par résolutions, alerte, panique via QCM), camembert SVG (bleu/rouge/noir), vérification de victoire, sauvegarde / reprise / export JSON, annulation, mode simulation. | Tests T1 à T5 passent ; une partie simulée complète se déroule sans violation d'invariant ; fermer et rouvrir le navigateur restaure la partie exactement. |
| **Lot 2 — Déroulé fenêtre unique** | Tous les écrans de §5 (Préparation, Manche 0, tableau de bord, phase de travail avec QCM, débat + résolutions AG, séquence de nuit avec appels de rôles, Bilan du matin), mode masqué, contrôles animateur (§6), chronomètres préréglés. | Une partie test réelle à 3 personnes (animateur + 2 « joueurs ») se joue de bout en bout sans toucher au code ; aucun secret n'apparaît jamais à l'écran en dehors des révélations prévues. |
| **Lot 3 — Habillage narratif** | Identité visuelle (§7), illustrations intégrées, textes du Règlement v2, animations, sons optionnels, récapitulatif de fin pour le débriefing. | Relecture complète des textes projetés ; test de lisibilité en conditions réelles de projection ; les lots 1 et 2 fonctionnent à l'identique. |

Méthode de travail avec Claude Code : fournir à chaque session ce cahier des charges v2 + le Règlement v2, et ne demander qu'un lot à la fois. Ne passer au lot suivant qu'après recette du précédent. Versionner sur GitHub à chaque étape validée (un commit par fonctionnalité recettée).

**Note :** l'annexe du corrigé complet des 15 défis (QCM + réponse correcte par service et par manche), déjà rédigée dans la version 1.1 du cahier des charges d'origine, doit être reprise telle quelle et jointe à ce document avant transmission à Claude Code — elle n'est pas reproduite ici.

---

## 10. Hors périmètre de la v2

- Combos de défis, variantes de règles, mode « moins de 8 joueurs ».
- Affichage des fiches de défis dans le diaporama (elles restent sur papier).
- Multijoueur en réseau, application mobile, multilingue.
- Statistiques inter-parties (pourra venir en v3 pour le calibrage).
- Illustration dédiée pour le Président (badge uniquement dans cette version).


## Annexe — Banque de défis (corrigé QCM)

Contenu des 15 fiches défis (Manche 1 à 5 × RH / Finance / Commercial), fourni séparément en fiches imprimables (Infiltres_Fiches_Defis). Ce tableau donne la réponse correcte de chaque QCM et son nombre de choix (variable selon la manche), à charger dans la banque de défis du moteur (§3). En cas de correction du contenu d'un défi, mettre à jour la fiche papier et cette réponse en parallèle.

| **Manche** | **Service**  | **Choix** | **Bonne réponse**                          |
|------------|--------------|-----------|---------------------------------------------|
| Manche 1   | RH           | A–C       | A — Faute grave                             |
| Manche 1   | Finance      | A–C       | A — Faute de gestion                        |
| Manche 1   | Commercial   | A–C       | A — Relation commerciale établie            |
| Manche 2   | RH           | A–D       | D — Licenciement pour faute grave           |
| Manche 2   | Finance      | A–D       | A — Conciliation                            |
| Manche 2   | Commercial   | A–D       | B — Franchise                               |
| Manche 3   | RH           | A–B       | B — Plan de résorption de l'absentéisme     |
| Manche 3   | Finance      | A–B       | B — Plan de recouvrement                    |
| Manche 3   | Commercial   | A–B       | B — Programme de fidélisation               |
| Manche 4   | RH           | A–B       | A — M. [J] (le salarié)                     |
| Manche 4   | Finance      | A–B       | B — Le liquidateur                          |
| Manche 4   | Commercial   | A–B       | A — La société L'Amy                        |
| Manche 5   | RH           | A–F       | C — SARL                                    |
| Manche 5   | Finance      | A–F       | B — SA                                      |
| Manche 5   | Commercial   | A–F       | D — SAS                                     |
