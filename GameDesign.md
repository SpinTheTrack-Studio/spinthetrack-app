# GAME DESIGN DOCUMENT : SpinTheTrack (App Edition)

**Version :** 3.0 (Final Design)
**Date :** 2026-02-15
**Type :** Party Game Musical / Quiz Connecté 100% Digital
**Plateformes :** Mobile (iOS / Android)
**Moteur Audio :** Deezer API

---

## 1. CONCEPT & IDENTITÉ

* **Titre :** **SpinTheTrack**
* **Accroche :** *"Votre musique, 4 façons de jouer."*
* **Pitch :** Un jeu de société musical compétitif qui transforme vos playlists Deezer en un plateau de jeu imprévisible. Contrairement à un blind-test classique, le défi change à chaque tour : culture, chant, rapidité ou mime vocal.

---

## 2. FLUX UTILISATEUR (USER FLOW)

### 2.1 Configuration (Lobby)
1.  **Connexion :** L'hôte lance l'app et se connecte à son compte Deezer (Premium recommandé).
2.  **Les Joueurs :** Ajout des participants (ex: "Alex", "Sam", "Julie").
3.  **Le Deck Musical :**
    * L'hôte sélectionne les sources musicales parmi ses playlists Deezer (ex: "Coups de cœur", "Soirée 90s", "Rap FR").
    * L'application fusionne ces sources pour créer une **"Pioche Audio"** unique pour la partie.
4.  **Start :** Lancement de la partie.

### 2.2 La Boucle de Jeu (Core Loop)
1.  **Désignation :** L'écran affiche *"C'est au tour de [JOUEUR]"*.
2.  **Le Spin (RNG) :** Une roue virtuelle tourne et sélectionne aléatoirement :
    * Une piste audio du Deck.
    * Un des **4 Modes de Jeu**.
3.  **Le Challenge :** Le défi se lance (voir section 3).
4.  **La Réponse :** Le joueur donne sa proposition à l'oral.
5.  **La Révélation :** Le joueur appuie sur "RÉVÉLER". L'app affiche la réponse complète (Titre, Artiste, Année, Paroles).
6.  **Validation :** Validation manuelle (Succès/Échec) par les autres joueurs via l'app.
7.  **Scoring :** Mise à jour du classement et passage au joueur suivant.

---

## 3. LES 4 MODES DE JEU (GAMEPLAY)

L'application détermine le mode *avant* que la musique ne démarre.

### MODE 1 : L'EXPERT (Culture)
* **Concept :** Prouver sa connaissance globale du titre.
* **Mécanique :** L'extrait joue normalement (30s).
* **Objectif :** Le joueur doit donner au moins **2 informations correctes** sur 3 possibles :
    1.  Nom de l'Artiste.
    2.  Titre de la chanson.
    3.  Année de sortie (à +/- 1 an).
* **UI :** Disque vinyle qui tourne. Champs masqués "???".
* **Points :** 1 pt si 2 critères validés. 2 pts si tout est bon (Perfect).

### MODE 2 : LE MAESTRO (Lyrics)
* **Concept :** "N'oubliez pas les paroles".
* **Mécanique :**
    * La musique démarre.
    * Les paroles s'affichent en temps réel à l'écran (façon karaoké).
    * **Coupure :** À un moment clé (refrain ou fin de phrase), le son se coupe net et le texte est remplacé par des pointillés `.......`.
* **Objectif :** Le joueur doit chanter la suite correcte a cappella.
* **UI :** Un micro rétro. Texte dynamique. Bouton "Voir la réponse" qui affiche le texte manquant.
* **Points :** 2 pts (Challenge difficile).

### MODE 3 : LE TWISTED (Oreille)
* **Concept :** Reconnaître un titre déformé temporellement.
* **Mécanique :**
    * L'application altère la vitesse de lecture (Playback Rate).
    * Soit **Rapide (Nightcore)** : 1.5x (Voix aigüe).
    * Soit **Lent (Chopped & Screwed)** : 0.75x (Voix grave).
* **Objectif :** Retrouver simplement **Titre + Artiste**.
* **UI :** Une icône de Lièvre (Rapide) ou de Tortue (Lent). Onde sonore déformée.
* **Points :** 1 pt.

### MODE 4 : LE FREDONNEUR (Social)
* **Concept :** Faire deviner la musique aux autres sans les paroles.
* **Mécanique :**
    * **Instruction :** *"Mets le téléphone à ton oreille ! Toi seul écoutes."*
    * **Audio :** Le son est routé vers l'écouteur interne (comme un appel téléphonique) ou demande des écouteurs. Le haut-parleur principal est coupé.
* **Objectif :** Le joueur actif écoute et doit fredonner la mélodie (Humming) ou faire "Lalalala". Les **autres joueurs** doivent deviner le titre.
* **UI :** Icône "Oreille barrée" ou "Chut". Compte à rebours de 45s.
* **Points :** 1 pt pour le Fredonneur + 1 pt pour celui qui trouve.

---

## 4. DESIGN & INTERFACE (UI)

L'application doit avoir une identité visuelle forte ("Neon / Dark Mode") pour s'imposer comme le centre de la soirée.

* **Écran "Le Spin" :** Transition critique. Animation fluide d'une roue ou de cartes qui défilent ("Shuffling...") avec un son mécanique satisfaisant.
* **Le HUD (En jeu) :**
    * Gros bouton central contextuel (Play / Pause / Révéler).
    * Barre de progression de la musique (Waveform).
    * Scoreboard simplifié toujours visible en bas (Top 3).
* **Écran de Révélation :**
    * Doit être gratifiant.
    * Affiche la pochette de l'album en HD.
    * Joue le refrain (si possible) ou reprend la lecture normale.

---

## 5. SPÉCIFICATIONS TECHNIQUES

### 5.1 Gestion des Playlists (Deck Building)
* L'API Deezer permet de récupérer les tracks d'une playlist `user`.
* L'app doit gérer un tableau local `CurrentGameDeck[]` contenant les IDs des pistes.
* **Anti-Répétition :** Une fois jouée, une track est marquée `played: true` et ne ressort plus.

### 5.2 Audio Player Custom
* Pour le **Mode 3 (Twisted)** : Nécessite un player natif capable de modifier le `pitch` / `rate` sans changer la tonalité (Time Stretching) ou en changeant la tonalité (Varispeed). Le Varispeed (plus simple) est souvent plus drôle.
* Pour le **Mode 4 (Fredonneur)** : Nécessite de basculer la sortie audio (`AVAudioSession` sur iOS) vers le `Receiver` (écouteur d'oreille) au lieu du `Speaker`.

### 5.3 Données Paroles (Mode Maestro)
* **Option A :** API Deezer Lyrics (si accessible).
* **Option B :** Base de données statique pour les "Classiques" (si API trop chère/complexe).
* **Fallback :** Si pas de paroles dispos pour un titre, le RNG ne doit jamais proposer le Mode 2 pour cette chanson spécifique.

---

## 6. CONDITIONS DE VICTOIRE

* **Objectif :** Premier joueur à atteindre **10 Points**.
* **Mort Subite :** En cas d'égalité à 10, un dernier "Duel Fredonneur" départage les vainqueurs.