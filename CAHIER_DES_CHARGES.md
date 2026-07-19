## INFILTRÉS

## Diaporama interactif — Cahier des charges

Version 1 — à remettre à Claude Code, lot par lot, avec le Règlement v2

## 1. Objet et principes directeurs

Application web pilotant une partie complète d'INFILTRÉS : attribution et affichage du capital, déroulé des phases, votes pondérés, mouvements de capital vers le Fonds Meridian, vérification automatique des conditions de victoire, habillage narratif. Un seul opérateur : l'animateur. Un seul écran public : le vidéoprojecteur. Le document de référence des règles est le Règlement v2 ; en cas de contradiction, le règlement prime.

## Principes non négociables

- **Un seul fichier HTML autonome** (HTML + CSS + JS embarqués, aucune dépendance CDN, aucun appel réseau). Doit fonctionner ouvert en local (file://) comme hébergé sur GitHub Pages, entièrement hors ligne une fois chargé. Le camembert est dessiné en SVG maison — pas de bibliothèque graphique externe.

- **Sauvegarde continue :** l'état complet de la partie est enregistré dans le localStorage après chaque action. À l'ouverture, si une partie est en cours, proposer « Reprendre la partie ». En complément : export et import de l'état en fichier JSON téléchargeable (secours en cas de changement de machine).

- **Annulation :** chaque action de jeu empile un instantané de l'état. Bouton « Annuler la dernière action » dans la console animateur, utilisable en cascade jusqu'au début de la partie.

- **Séparation public / secret :** aucune information secrète (camps, rôles, cibles nocturnes) ne doit jamais pouvoir apparaître sur l'écran projeté. Voir architecture double fenêtre, §2.

- **Mode simulation** intégré pour la recette : voir §8.

## 2. Architecture d'affichage : double fenêtre

Deux fenêtres synchronisées en temps réel (BroadcastChannel ou équivalent localStorage) :

- **La Scène** (fenêtre projetée, plein écran 16:9) : camembert, bandeau d'état, écrans narratifs, votes, bilans. Ne contient jamais de commande ni de secret.

- **La Console** (fenêtre sur l'écran du portable de l'animateur) : toutes les commandes, les données secrètes, le journal des événements, l'annulation, les corrections manuelles.

Repli si la double fenêtre est impossible (projection dupliquée) : un « voile de nuit » plein écran s'affiche sur la Scène pendant que l'animateur manipule la Console, et la Console peut être masquée d'une touche (Échap). La Console pilote la navigation de la Scène : l'animateur choisit à tout moment quel écran est projeté.

## 3. Modèle de données

| **Entité** | **Champs**                                                                                                                                                                                                                                                                                                                                                              | **Notes**                                                                                         |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Partie     | manche (0–5), phase (travail / conseil / nuit / matin), tentativeRévocationUtilisée (bool, par manche), conséquenceDéfisEnAttente (indice / statuquo / alerte / panique), paliersMeridianAnnoncés \[15, 25, 35, 45\], journal des événements, pile d'annulation                                                                                                         | L'état complet doit être sérialisable en JSON (sauvegarde, export, simulation).                   |
| Joueur     | prénom, service (RH / Commercial / Finance), part (en millièmes de %), statut (actif / gelé / révoqué), camp (honnête / infiltré — SECRET), rôle (associé / DRH / Avocat / DAF / Président / Juriste — SECRET jusqu'à révélation), révélé (bool), flags : protégéCetteNuit, protégéNuitPrécédente, miseÀPiedManche (n° de manche concernée), mandataireDuJuriste (bool) | Le Président est un attribut transférable : champ présidentActuel au niveau Partie, réassignable. |
| Meridian   | part (en millièmes de %)                                                                                                                                                                                                                                                                                                                                                | Démarre à 1,000 % exactement.                                                                     |

**Arithmétique :** tous les calculs internes se font en millièmes de pourcent (entiers), jamais en flottants. Affichage arrondi au dixième de %. Invariant permanent, vérifié après chaque action : somme des parts des joueurs non révoqués + Meridian = 100,000 % exactement. Les restes d'arrondi d'une répartition sont attribués par la méthode du plus fort reste. Toute violation de l'invariant déclenche une alerte visible en Console.

## 4. Règles de calcul — le moteur

## 4.1 Attribution initiale des parts

- Les joueurs se partagent 99,000 % ; Meridian détient 1,000 %.

- Tirage aléatoire légèrement inégal : chaque joueur reçoit entre 0,65× et 1,35× la part moyenne (99/N), puis normalisation à 99,000 %. À 15 joueurs, cela donne des parts entre 4 % et 9 % environ.

- La Console affiche le tirage avant validation, avec « Retirer au sort » et l'édition manuelle de chaque part (re-normalisation automatique).

## 4.2 Sabotage (phase nocturne)

- Cible sélectionnée en Console parmi les joueurs actifs. Si la cible est le joueur protégé cette nuit : le sabotage échoue, le matin annonce une « nuit calme », rien d'autre ne change.

- Sinon : statut → gelé ; camp et rôle révélés publiquement au bilan du matin ; ses parts restent comptées au capital total mais sortent des actions votantes et sont exclues des prélèvements d'Alerte / Panique.

- **Exception Juriste :** s'il est saboté, la Console demande de désigner son mandataire (contrôle : un mandataire ne peut détenir qu'une procuration). Ses parts restent votantes (votées par le mandataire) et restent dans l'assiette des prélèvements — pas de séquestre, c'est son pouvoir.

## 4.3 Révocation (vote d'AG)

- Une seule tentative par manche (verrouillage automatique).

- Vote pondéré : la Console saisit Pour / Contre / Abstention pour chaque votant. Votants = joueurs actifs + le mandataire pour les parts du Juriste saboté. La révocation est acquise si le total « Pour » dépasse strictement 50,000 % des actions votantes totales (pas seulement des exprimées : s'abstenir revient à voter contre).

- Si acquise : carte révélée ; ses parts sont redistribuées au prorata de leurs parts entre tous les joueurs non révoqués (actifs ET gelés) — jamais à Meridian ; statut → révoqué, part → 0.

- Si le révoqué est le Président : la Console invite à désigner son successeur (annoncé sur la Scène).

## 4.4 Alerte et Panique (conséquences des défis)

- Bilan des 3 défis d'une manche : 3 réussis → Indice (révélé immédiatement, avant le vote de la même manche) ; 2/1 → rien ; 1/2 → Alerte (4,000 points) ; 0/3 → Panique (8,000 points). Les valeurs 4 et 8 sont des constantes de configuration modifiables en Console (calibrage playtest).

- Le transfert vers Meridian est appliqué et annoncé au bilan du matin suivant : chaque joueur de l'assiette (actifs + Juriste saboté) cède part_i × montant ÷ somme(assiette). Prélèvement strictement proportionnel, restes au plus fort reste.

- Indices proposés par la Console (l'animateur en choisit un) : « le service X compte n infiltré(s) » (service au choix ou au hasard) ; « le camp adverse — infiltrés et Meridian réunis — détient entre Y et Y+5 % du capital » (fourchette de 5 points contenant la vraie valeur, bornes en multiples de 5).

## 4.5 Vérification de la victoire

- **À chaque bilan du matin (nuits 1 à 5 incluses) :** si somme des parts personnelles des infiltrés non révoqués (quel que soit leur statut) + part Meridian \> 50,000 % → écran de victoire Meridian, partie terminée.

- À tout moment : si tous les infiltrés sont révoqués → écran de victoire honnête anticipée.

- Après le bilan du matin de la nuit 5, si aucune des deux conditions : écran du chevalier blanc, victoire honnête finale.

## 5. Écrans de la Scène

| **Écran**                         | **Contenu et comportement**                                                                                                                                                                                                                                                                                                                                                                                                       |
|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| S0 — Accueil                      | Logo INFILTRÉS / STRATÉJEUX. Piloté par la Console : Nouvelle partie / Reprendre la partie.                                                                                                                                                                                                                                                                                                                                       |
| S1 — Le conseil (tableau de bord) | Écran par défaut entre les temps forts. Camembert du capital (une part par joueur, tranche Meridian gris anthracite), liste des joueurs avec statut (actif / sous séquestre / révoqué, rôle si révélé), n° de manche et phase en cours, badge Président. Bandeau permanent rappelant le dernier événement.                                                                                                                        |
| S2 — Lettre d'intention           | Manche 0. Mise en scène « courrier » : le texte intégral de la lettre (Règlement v2), affiché progressivement. Déclenche l'apparition de la tranche Meridian à 1 % sur le camembert.                                                                                                                                                                                                                                              |
| S3 — Phase de travail             | Titre de la manche, rappel des 3 services, chronomètre de défi configurable (durée réglée en Console, alerte sonore optionnelle). La saisie des résultats se fait en Console ; si Indice : affichage immédiat de l'indice choisi, plein écran.                                                                                                                                                                                    |
| S4 — Conseil & AG                 | Étape a) Motion : nom du suspect proposé et de son « second » (saisis en Console). Étape b) Débat : chronomètre 5:00. Étape c) Vote : barre de progression du « Pour » en % des actions votantes, seuil 50 % marqué ; résultat ; si révocation : révélation de la carte (camp + rôle) puis animation de redistribution du camembert.                                                                                              |
| S5 — Voile de nuit                | Visuel sombre « NOVENTIS dort » ; aucun secret. Pendant ce temps, l'animateur traite la nuit en Console.                                                                                                                                                                                                                                                                                                                          |
| S6 — Bilan du matin               | Séquence ordonnée, avancée manuellement : 1) scénario de sabotage de la manche (textes du Règlement v2) avec révélation de la carte du saboté — ou « nuit calme » ; 2) communiqué Meridian (Alerte ou Panique) avec animation du camembert ; 3) annonce de palier si 15 / 25 / 35 / 45 % vient d'être franchi (chaque palier ne s'annonce qu'une fois) ; 4) vérification de victoire — si déclenchée, transition directe vers S7. |
| S7 — Écrans de fin                | Trois variantes : victoire Meridian (prise de contrôle, révélation de tous les camps), victoire honnête anticipée (tous les infiltrés révoqués), victoire honnête finale (chevalier blanc). Camembert final + récapitulatif de la partie (chronologie des événements).                                                                                                                                                            |

## 6. La Console animateur

- **Configuration :** nombre de joueurs (8–18), prénoms, affectation aux services (proposition automatique équilibrée, modifiable), rappel du nombre d'infiltrés recommandé (⌊N/3⌋), tirage des parts (§4.1).

- **Saisie secrète (après distribution physique des cartes) :** camp de chaque joueur, détenteurs des 5 rôles, Président initial. Contrôles de cohérence : nombre d'infiltrés conforme, rôles à pouvoir honnêtes sauf éventuellement le Président, un rôle par joueur maximum.

- **Panneau de nuit :** protection de l'Avocat (contrôles : pas lui-même, pas deux nuits de suite la même personne), cible du sabotage, mise à pied DRH (1×/partie — marque le joueur exclu du défi de son service à la manche suivante, annoncé sobrement au matin), enquête DAF (1×/manche — affiche à l'animateur seul le rôle exact du joueur désigné, pour transmission discrète au DAF).

- **Panneau de vote :** liste des votants avec leurs parts, saisie Pour / Contre / Abstention, calcul en direct, bouton de validation.

- **Corrections :** annulation en cascade, édition manuelle d'une part (avec re-normalisation et trace au journal), changement du Président, bascule manuelle de tout écran de la Scène.

- **Journal :** chaque événement horodaté (sabotages, votes avec détail, mouvements de capital, indices révélés). Exportable en texte pour le débriefing.

## 7. Habillage (lot 3)

- Identité STRATÉJEUX : fond bleu nuit 0E2A47, accents sarcelle 1B8C82, blanc cassé pour les textes. Typographie lisible à 8 mètres (corps minimal équivalent 28 pt projeté).

- Textes à intégrer tels quels depuis le Règlement v2 : lettre d'intention, 5 scénarios de sabotage, 2 communiqués Meridian, 4 annonces de palier, 3 textes de fin de partie.

- Animations sobres : transitions du camembert (800 ms), effet « communiqué de presse » pour Meridian, révélation de carte façon retournement. Sons optionnels et désactivables (gong de nuit, notification de communiqué).

- Tout l'habillage est ajouté sans modifier le moteur : les lots 1 et 2 doivent rester fonctionnels à tout moment.

## 8. Mode simulation et tests d'acceptation

La Console comporte un mode simulation : création instantanée d'une partie fictive (prénoms générés), exécution pas à pas ou automatique d'un scénario scripté, affichage de l'état chiffré complet à chaque étape. Les trois tests suivants font partie de la recette du lot 1 — les valeurs attendues sont exactes et doivent être reproduites au millième :

| **Test**        | **Situation initiale**                                                        | **Action**                       | **Résultat attendu**                                                                                                                                                                                                  |
|-----------------|-------------------------------------------------------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| T1 — Panique    | 10 joueurs à 9,900 % chacun, Meridian 1,000 %                                 | Panique (8,000 points)           | Chaque joueur cède 9,900 × 8 ÷ 99 = 0,800. Joueurs à 9,100 % chacun ; Meridian à 9,000 % ; total 100,000 %.                                                                                                           |
| T2 — Révocation | État final de T1                                                              | Révocation d'un joueur (9,100 %) | Redistribution au prorata entre les 9 restants (somme 81,900) : chacun reçoit 9,100 × 9,100 ÷ 81,900 ≈ 1,011. Joueurs à 10,111 % ; Meridian inchangé à 9,000 % ; total 100,000 %.                                     |
| T3 — Victoire   | Parts des infiltrés non révoqués + Meridian = 50,2 % / puis variante à 49,9 % | Bilan du matin                   | À 50,2 % : écran de victoire Meridian. À 49,9 % : la partie continue. Le seuil est strictement supérieur à 50,000 %.                                                                                                  |
| T4 — Séquestre  | État de T1, un joueur gelé (non-Juriste)                                      | Alerte (4,000 points)            | Le gelé ne cède rien ; l'assiette est la somme des parts des actifs ; sa part reste comptée dans le capital total et dans le calcul de victoire s'il est infiltré.                                                    |
| T5 — Invariant  | Toute simulation complète (5 manches, événements mélangés)                    | —                                | Après chaque action : somme joueurs non révoqués + Meridian = 100,000 %. Un joueur ne peut être ciblé deux fois, une 2e révocation dans la même manche est refusée, chaque palier Meridian n'est annoncé qu'une fois. |

## 9. Découpage en 3 lots — ordre impératif

| **Lot**                           | **Périmètre**                                                                                                                                                                                                                        | **Critère de recette (« terminé quand »)**                                                                                                                                               |
|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Lot 1 — Moteur de capital         | Modèle de données, arithmétique en millièmes, attribution initiale, les 4 actions (sabotage, révocation, alerte, panique), camembert SVG, vérification de victoire, sauvegarde / reprise / export JSON, annulation, mode simulation. | Les tests T1 à T5 passent ; une partie simulée complète se déroule sans violation d'invariant ; fermer et rouvrir le navigateur restaure la partie exactement.                           |
| Lot 2 — Déroulé et double fenêtre | Scène + Console synchronisées, tous les écrans S0–S7 en version sobre, saisie secrète, panneau de nuit avec tous les contrôles (Avocat, Juriste-procuration, DRH, DAF), panneau de vote pondéré, chronomètres, journal.              | Une partie test réelle à 3 personnes (animateur + 2 « joueurs ») se joue de bout en bout sans toucher au code ni ouvrir la console du navigateur ; aucun secret n'apparaît sur la Scène. |
| Lot 3 — Habillage narratif        | Identité visuelle, textes du Règlement v2 intégrés, animations, sons optionnels, récapitulatif de fin pour le débriefing.                                                                                                            | Relecture complète des textes projetés ; test de lisibilité en conditions réelles de projection ; les lots 1 et 2 fonctionnent à l'identique.                                            |

Méthode de travail avec Claude Code : fournir à chaque session ce cahier des charges + le Règlement v2, et ne demander qu'un lot à la fois. Ne passer au lot suivant qu'après recette du précédent. Versionner sur GitHub à chaque étape validée (un commit par fonctionnalité recettée).

## 10. Hors périmètre de la v1

- Combos de défis, variantes de règles, mode « moins de 8 joueurs ».

- Affichage des fiches de défis dans le diaporama (elles restent sur papier).

- Multijoueur en réseau, application mobile, multilingue.

- Statistiques inter-parties (pourra venir en v2 pour le calibrage).
