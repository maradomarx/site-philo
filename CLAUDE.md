# FicheÉclair — Document de projet

> **Note pour Claude** : ce document est lu automatiquement par Claude Code à
> chaque session, et fourni en tête de chaque nouvelle conversation
> stratégique sur claude.ai. Il a deux parties : une partie **stable**
> (identité, vision, stack, conventions) qui change rarement, et une partie
> **vivante** (état actuel, roadmap) à mettre à jour à chaque fin de mission.

---

# PARTIE STABLE

## Identité du projet

FicheÉclair n'est pas un générateur de fiches isolées, c'est un **transformateur
de cours en environnement de révision complet**. Un étudiant uploade
progressivement les chapitres de son cours universitaire (en photo ou en PDF).
Pour chaque chapitre, une fiche structurée est générée. À mesure que le cours
s'enrichit, des outils de révision (QCM, cartes, entraînement, dissertation,
ressources, mini-jeu) s'adaptent automatiquement au contenu accumulé.

L'unité naturelle d'étude est le **cours**, pas la fiche isolée. Une fiche est
un fragment de contenu qu'on ajoute à un cours. Les outils opèrent au niveau
du cours et se mettent à jour quand on lui ajoute une fiche.

## Cible

Étudiants en sciences humaines : philosophie, classes prépa littéraire et B/L,
masters de lettres et de philosophie. Les outils existants (NotebookLM,
Mindgrasp) sont génériques et conçus pour les sciences. FicheÉclair est
spécifiquement conçu pour les disciplines littéraires (problématiques, enjeux,
controverses, notions, citations).

## Auteur

Étudiant en master de philosophie, pas développeur. Bagage technique :
HTML/CSS basique au départ, expérience de code généré par IA, a appris
récemment Git, VS Code, Cursor/Claude Code, déploiement Cloudflare.

## Stack technique

- **Frontend** : HTML/CSS/JS vanilla, tout dans `index.html` (~5500 lignes),
  servi par un Worker Cloudflare nommé `site-philo` (déploiement auto depuis
  GitHub à chaque push sur `main`).
- **Backend** : trois Workers Cloudflare distincts.
  - `site-philo` : sert le HTML statique. Lié au repo GitHub
    `chevallierfabio-hue/site-philo`.
  - `claude-proxy` : proxy authentifié vers l'API Anthropic, utilisé par la
    fonctionnalité Disserter. Édité directement dans l'interface Cloudflare
    (pas dans le repo Git).
  - `ficheeclair-api` : Worker dédié à FicheÉclair. Endpoint
    `POST /generate-fiche` authentifié par JWT Supabase, avec quota
    journalier, qui appelle Claude Vision pour générer une fiche structurée
    et l'insérer dans Supabase. Code dans le sous-dossier `ficheeclair-api/`
    du repo, déployé automatiquement.
- **Base de données** : Supabase. Tables actuelles :
  - `scores` : classements QCM et Pong (avec RLS)
  - `sr_data` : données de répétition espacée pour l'entraînement
  - `fiches` : fiches générées (avec RLS, colonnes id/user_id/created_at/
    titre/discipline/contenu jsonb/cours_id)
  - `cours` : cours regroupant les fiches (avec RLS, colonnes id/user_id/
    titre/discipline/niveau/description/created_at/updated_at)
- **API IA** : API Anthropic. Modèle Sonnet 4.6 (`claude-sonnet-4-6`) pour
  les fiches et les dissertations. Clés API distinctes par Worker pour
  suivre les coûts par fonctionnalité. Limite de dépense configurée.
- **Dev** : VS Code + Claude Code, mode "Claude propose, l'utilisateur
  valide chaque modification".

## Vision architecturale

Hiérarchie cible :
Site
└─ Mes cours (écran principal pour les connectés)
├─ Cours "Épistémologie L3"
│   ├─ Onglet "Fiches" : la liste des fiches du cours
│   ├─ Onglet "QCM" : opère sur l'ensemble des fiches
│   ├─ Onglet "Cartes"
│   ├─ Onglet "Entraînement"
│   ├─ Onglet "Disserter"
│   ├─ Onglet "Ressources"
│   └─ Onglet "Pong"
└─ Cours "Philosophie de l'esprit" (ancien contenu, migré dans la base)
└─ Pour les non-connectés : page d'accueil / présentation

Niveau 1 (nav globale, pour les connectés) : Mes cours, Mon compte.
Niveau 2 (à l'intérieur d'un cours) : Fiches + les outils qui ont été
rebranchés sur les fiches du cours.

Règle pour la transition : un onglet n'apparaît dans la nav du cours que
quand il est effectivement rebranché sur les fiches. Tant qu'il n'est pas
prêt, il est invisible (pas grisé).

## Conventions de code

- JavaScript vanilla uniquement, pas de framework, pas d'outil de build
- Indentation 2 espaces
- Commentaires en français
- Variables courtes acceptables (cohérence avec le style existant)
- Préfixe `fe-` pour les classes CSS de FicheÉclair (évite les collisions
  avec le CSS existant)
- Fonction `escHTML()` (qui échappe `&`, `<`, `>`, `"`, et `'`) à utiliser
  systématiquement pour toute interpolation de chaîne dans du HTML construit
  par template string

## Décisions techniques actées

- Photos envoyées directement à Claude Vision en base64, pas d'OCR séparé
- Redimensionnement côté navigateur avant envoi (max 2000px, JPEG 0.85 avec
  fallback 0.7 puis 0.5 si la taille dépasse 5 Mo)
- Fiches stockées en JSON structuré dans la colonne `contenu jsonb` de la
  table `fiches` (pas en HTML), pour permettre les outils dérivés
- Modèle Sonnet 4.6 pour les fiches (Haiku envisagé au départ, mais Sonnet
  validé après test sur cas réel d'un cours de philo)
- Quota journalier de 10 fiches par utilisateur, calculé en UTC (pas en
  fuseau Europe/Paris — simple et prévisible)
- Vérification du JWT côté Worker en interrogeant `/auth/v1/user` de
  Supabase (plutôt que de vérifier la signature avec le secret JWT)
- Vérification du quota via la table `fiches` avec le JWT de l'utilisateur
  (pas la clé service_role), pour bénéficier des politiques RLS
- Prompt système calibré pour privilégier **fidélité** sur exhaustivité
  formelle : préférer un tableau vide à un contenu inventé
- INSERT dans `cours` exige `user_id` explicite côté frontend
  (la politique RLS ne le remplit pas automatiquement)
- Les modales (et autres éléments en `position:fixed`) doivent être
  enfants directs de `<body>` pour ne pas être cassées par les `transform`
  des ancêtres

## Méthodologie

- **Réflexion stratégique et revue de code via Claude.ai** (claude.ai) :
  conversation longue qui sert d'architecte, prépare les prompts pour
  Claude Code, et relit le code généré.
- **Exécution dans Claude Code** (VS Code) : Sonnet par défaut, Opus
  exceptionnel. L'utilisateur valide chaque modification proposée.
- **Une mission = une session ciblée.** Validation des plans avant
  exécution. Pas d'enchaînement de plusieurs missions dans une même
  session sans pause.
- **Une branche Git par mission.** Format `mission-<id>-<description>`,
  par exemple `mission-3a-2-liste-fiches`. Travail sur la branche, push,
  fusion dans `main` seulement après test en local et accord explicite.
- **Relecture systématique du code généré** avant déploiement. Pas de
  "commit-puis-on-verra".

---

# PARTIE VIVANTE

> **À mettre à jour à la fin de chaque mission.** Garde à jour cette
> section et seulement celle-ci. La date est utile pour situer la dernière
> mise à jour.

## État au [date du jour] — après R5.3

**Côté base Supabase** :
- Table `exercices` enrichie avec le type 'flashcard' (déjà autorisé 
  par le check polymorphe posé en R5.1).
- Format `contenu` jsonb pour flashcard : { recto: terme, verso: definition }.
- Clé naturelle de matching avec les notions_cles : `terme` ↔ `recto` 
  (comparaison exacte, sensible à la casse).

**Côté frontend (`index.html`)** :
- Bouton "Synchroniser les flashcards" sur la vue détail de fiche, 
  à côté du bouton "Générer un QCM". Toute la logique côté SDK 
  Supabase, pas d'appel Worker. INSERT/UPDATE/DELETE en parallèle 
  via Promise.all. Garde-fou : si notions_cles vide, aucun DELETE 
  déclenché.
- UPDATE préserve les champs futurs du contenu via spread 
  (...match.contenu, verso: ...) : prévoit l'ajout futur de champs 
  comme données de répétition espacée.
- Écran d'accueil de l'onglet "S'entraîner" refait en menu à deux 
  cartes : "QCM" et "Flashcards", chacune avec son compteur 
  d'éléments disponibles. État vide unifié si pool QCM et pool 
  flashcards sont vides.
- Mode flashcards complet : tirage de l'ensemble du pool mélangé 
  Fisher-Yates, recto (terme seul) → bouton Retourner → verso 
  (recto + définition + fiche d'origine) → auto-évaluation 
  "Je savais" / "À revoir". Bouton "Terminer la session" en 
  permanence. Reprise de session après switch d'onglet, exclusivité 
  mutuelle entre sessions QCM et flashcards (l'une bloque l'autre).
- Helper `feChargerPoolExercices(coursId, type)` : factorise la 
  jointure-or-fallback pour QCM et flashcards.
- Pools rechargés à chaque entrée dans l'onglet S'entraîner (pas 
  de cache pour rester synchrone avec ce que l'utilisateur vient 
  de générer).

## Roadmap

### Refonte vers la nouvelle architecture
- ~~R1, R2, R2-bis, R5.1, R5.2, R5.3~~ ✓
- R5.4 — type auteur généré par IA (nouvel endpoint Worker, 
  asymétrique avec QCM)
- R3 — partiellement absorbé par R5.2, à réviser
- R4 — migration du cours philo de l'esprit
- R6+ — autres outils à rebrancher

## Leçons retenues (R5.3)

- L'UPDATE polymorphe sur une colonne jsonb doit toujours préserver 
  les champs existants (spread du contenu actuel), même si 
  aujourd'hui le contenu n'a que deux champs. Anticipation des 
  évolutions futures.
- Quand on a un comportement destructif (DELETE silencieux), prévoir 
  un garde-fou explicite. Cas typique : pas de notions_cles → ne 
  supprime pas les flashcards existantes, sort proprement.
- Quand plusieurs types de "sessions" coexistent dans un même 
  onglet, prévoir l'exclusivité mutuelle au moment du lancement, 
  pas au moment du rendu.
- Mismatch de cadrage : ce que l'utilisateur dit vouloir au moment 
  des questions binaires ne correspond pas toujours à sa vision 
  réelle. Cas vécu en R5.3 : l'utilisateur a choisi "bouton sur 
  la vue fiche" parce que c'était une option présentée, alors 
  qu'il voyait spontanément une génération automatique invisible 
  dans l'onglet S'entraîner. Toujours présenter aussi l'option 
  "automatique" dans les futurs cadrages, même si elle paraît plus 
  complexe.

Mission localhost-detection — faite
Côté frontend (index.html) : les deux constantes FICHEECLAIR_API et FICHEECLAIR_API_QCM, auparavant en dur sur le domaine de prod, dérivent maintenant d'une base FICHEECLAIR_BASE qui détecte l'environnement : http://localhost:8787 si location.hostname ∈ {localhost, 127.0.0.1}, sinon le Worker prod. Les fetch héritent sans modification.
Côté ficheeclair-api/wrangler.jsonc : ajout de "dev": { "port": 8787 } pour figer le port de wrangler dev, afin que l'URL locale corresponde toujours à la constante frontend.
Vérification CORS : aucune modif nécessaire — buildCorsHeaders autorisait déjà tout http://localhost: (liste blanche dynamique, pas de wildcard). Le socle d'environnement est donc prêt pour R5.4 : un futur FICHEECLAIR_API_AUTEUR = FICHEECLAIR_BASE + '/generate-auteur' héritera de la détection ET de l'autorisation CORS.
Dev local : API via wrangler dev dans ficheeclair-api/ (port 8787) ; frontend via python3 -m http.server 8788 à la racine (pas wrangler dev à la racine, qui boucle à cause de assets.directory: "."). Supabase : http://localhost:8788 ajouté aux Redirect URLs.

## Pool de missions annexes mis à jour

- **NOUVEAU — Refonte UX de la génération QCM + flashcards.** 
  Vision cible : pas de bouton "Générer/Synchroniser" sur la vue 
  détail de fiche. À la place, dans l'onglet "S'entraîner", 
  synchro automatique invisible à chaque ouverture (flashcards : 
  trivial car gratuit ; QCM : plus complexe car coût API par fiche, 
  à cadrer). Mission ouverte sans urgence — l'usage en condition 
  réelle dira si l'UX actuelle est gênante ou pas.
- (autres missions inchangées)
