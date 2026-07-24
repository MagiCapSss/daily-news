# CONTEXT — Daily Briefing project

Ce fichier résume tout ce qui a été mis en place pour ce projet, dans quel ordre, et où en est l'état d'avancement. Écrit pour qu'un agent sans aucun autre contexte (humain ou Claude) puisse reprendre le travail immédiatement.

## Objectif du projet

Générer automatiquement, chaque matin à 6h00 (Europe/Paris), un briefing audio d'actualité d'environ 25 minutes en français, écoutable comme un podcast sur un téléphone Android (Samsung Galaxy S23), avec en parallèle une note texte sourcée dans le vault Obsidian de l'utilisateur pour approfondir un sujet si besoin. Contrainte forte : tout doit être gratuit, et l'ordinateur de l'utilisateur n'a pas besoin d'être allumé (génération 100% cloud).

## Architecture générale

1. Une **routine cloud planifiée** (Claude Code scheduled routine / RemoteTrigger) se déclenche chaque matin à 4h00 UTC (6h Paris).
2. Cette routine tourne dans un environnement cloud isolé, clone le repo GitHub du projet, fait des recherches web, écrit le script du jour, génère l'audio via une API TTS, met à jour un flux RSS podcast, et push tout sur GitHub.
3. **GitHub Pages** sert le contenu du repo (audio + flux RSS) publiquement, gratuitement.
4. L'utilisateur ajoute l'URL du flux RSS dans une app podcast Android (AntennaPod recommandé, gratuit).
5. Un dossier du vault Obsidian de l'utilisateur est lui-même un clone git de ce repo, synchronisé automatiquement via le plugin **Obsidian Git** — ce qui permet à la note sourcée du jour d'apparaître automatiquement dans Obsidian sans action manuelle.

## Repo GitHub

- URL : `https://github.com/MagiCapSss/daily-news`
- **Public** (nécessaire pour GitHub Pages gratuit)
- GitHub Pages activé : Settings → Pages → Source = branche `main`, dossier `/ (root)`
- URL publique : `https://magicapsss.github.io/daily-news/`
- Flux podcast (une fois généré) : `https://magicapsss.github.io/daily-news/feed.xml`

### Structure de fichiers attendue dans le repo

```
README.md                  — description du projet
CONTEXT.md                 — ce fichier
feed.xml                   — flux RSS podcast (créé au premier run réussi)
audio/YYYY-MM-DD.mp3        — épisode audio du jour
notes/YYYY-MM-DD.md         — script complet du jour, avec sources en liens markdown
```

### Rétention

Les fichiers audio (et les `<item>` correspondants dans `feed.xml`) de plus de 30 jours sont supprimés automatiquement par la routine pour garder le repo léger. Les notes dans `notes/` ne sont **jamais** supprimées — c'est l'archive permanente.

## Intégration Obsidian (vault local, PAS versionné entièrement)

**Important : seul un sous-dossier du vault est lié à GitHub, jamais le vault entier.** Le vault contient des informations personnelles sensibles (`_System/soul.md` etc.) — le lier en entier à un repo public exposerait tout publiquement. C'est un point de vigilance à ne jamais casser.

- Chemin local : `C:\Users\aizar\Documents\Hub\1-Projects\Daily Briefing\`
- Ce dossier est son propre repo git indépendant (`git init` + `git remote add origin https://github.com/MagiCapSss/daily-news.git`), cloné/lié manuellement, PAS un submodule.
- Branche locale `main` trackant `origin/main`.
- Le plugin **Obsidian Git** est configuré avec un "custom base path" pointant vers `1-Projects/Daily Briefing` (dans les settings Obsidian, cherché via la barre de recherche des Settings avec le mot-clé "base path" — le nommage exact du setting varie selon les versions du plugin). Auto-pull activé (intervalle configuré par l'utilisateur, ~60 min). Confirmé fonctionnel par l'utilisateur.
- Résultat : la note du jour apparaît automatiquement dans Obsidian après chaque run réussi, sans action manuelle, dès que le plugin fait son prochain pull.

## Routine cloud (RemoteTrigger)

- Nom : "Daily Briefing Podcast"
- ID : `trig_01C9gW6gQoeQEeFaQjizWuss`
- Page de gestion : `https://claude.ai/code/routines/trig_01C9gW6gQoeQEeFaQjizWuss`
- Cron : `0 4 * * *` (4h00 UTC = 6h00 Europe/Paris), tous les jours
- Environment : `env_01PhMeDNjerzfXWh6WzEfjWJ` (environment "Default")
- Modèle : `claude-sonnet-5`
- Tools autorisés : Bash, Read, Write, Edit, Glob, Grep, WebSearch, WebFetch
- Source : clone du repo `https://github.com/MagiCapSss/daily-news`
- Le prompt complet de la routine (dans `job_config.ccr.events[0].data.message.content`) contient toutes les instructions détaillées : structure du script, durées cibles par section, procédure TTS, format du flux RSS, règles de rétention, gestion des erreurs. Consultable/modifiable via l'outil `RemoteTrigger` (actions `get`/`update`) ou la page de gestion ci-dessus.
- Pour tester manuellement sans attendre 6h : `RemoteTrigger` action `run` avec ce `trigger_id`.

### Contenu du briefing (spécifié par l'utilisateur)

Langue : **français**, ton spoken/oral (pas de markdown, pas de tirets, pas de puces dans le texte destiné au TTS).

- Accroche (hook) : 2-4 phrases, sans tirets, tout au début.
- Géopolitique & politique française (2-3 min) : 2-3 événements les plus importants des dernières 24h, factuel, strictement neutre.
- Marchés financiers, crypto & matières premières (6-7 min) : clôtures US/Europe, taux/obligations si événement Fed/BCE notable, Bitcoin/Ethereum, énergie/matières premières si mouvement notable, avis d'analystes sourcés. Sauter les sous-thèmes sans rien de notable.
- Tech générale hors IA (2 min) : semi-conducteurs, cybersécurité, GAFAM.
- IA, vue d'ensemble (7-8 min) : impact éco/politique/social, régulation (AI Act, RGPD, antitrust), cas d'usage business, débats publics, annonces produit. Plafond 4-6 sujets choisis pour leur importance réelle.
- Macroéconomie (2 min) : inflation, emploi, décisions banques centrales — toujours expliquer le "pourquoi".
- Startups & levées de fonds (2-3 min).
- Pépite du jour (1-2 min) : anecdote légère/scientifique, sans lien avec l'actu dure.
- Citation du jour : citation inspirante liée à l'actu, auteur nommé, ton calme.
- Sign-off exact : "Bonne journée, et à demain."

Total cible : ~25 minutes (~3750-4000 mots à ~150 mots/minute en français). Si un sujet est calme ce jour-là, le raccourcir/sauter plutôt que remplir avec du contenu creux.

## TTS (partie la plus fragile du pipeline)

### Tentative 1 — edge-tts : ÉCHEC, abandonnée

`edge-tts` (Microsoft, gratuit, pas de clé API) était le choix initial. **Ne fonctionne pas** : l'environnement sandbox cloud a une politique d'egress réseau très restrictive — seuls les hôtes `*.googleapis.com` sont autorisés. `speech.platform.bing.com` (backend d'edge-tts) et toutes les alternatives testées (Azure TTS, OpenAI, ElevenLabs) reçoivent un `403 host_not_allowed`. Ne pas re-essayer cette voie, elle est bloquée au niveau réseau du sandbox, pas contournable.

### Tentative 2 — Google Cloud Text-to-Speech : EN COURS DE VALIDATION

Solution retenue car `texttospeech.googleapis.com` est sur l'allowlist réseau du sandbox.

- API : `POST https://texttospeech.googleapis.com/v1/text:synthesize?key=API_KEY`
- Voix utilisée : `fr-FR-Wavenet-D` (français, WaveNet)
- Limite de l'API : ~5000 caractères par requête → la routine découpe le script en chunks de 4500 caractères max (sur des frontières de phrase), fait un appel par chunk, décode le `audioContent` (base64) de chaque réponse en `.mp3`, puis **concatène les fichiers mp3 en binaire brut** (pas de ffmpeg disponible dans le sandbox — ni `pip install`, ni `apt install` ne fonctionnent, seul `curl` + `python3` stdlib + `base64` sont utilisables). La concaténation binaire brute de plusieurs mp3 fonctionne pour la lecture dans la plupart des lecteurs, avec potentiellement une micro-coupure imperceptible entre sections.
- Palier gratuit Google : voix WaveNet ≈ 1 million de caractères/mois gratuits. Usage réel estimé : ~20-25k caractères/jour × 30 ≈ 750k/mois → reste dans le gratuit.
- **La clé API n'est PAS dans ce repo** (public !) — elle est stockée uniquement dans le prompt de la routine cloud (`RemoteTrigger`, visible seulement au propriétaire du compte). Les instructions de la routine interdisent explicitement d'écrire la clé dans un fichier committé. Ne jamais ajouter la clé API dans ce repo.

### Historique des runs de test

1. **Run 1** (~2026-07-23 12:34, session `cse_013VgTSaP71XYL69BHgTe6Fq`, avant la config Google TTS) : recherche + script écrits en interne, mais **rien commité** — deux blocages : (a) edge-tts bloqué par la politique réseau, (b) push GitHub refusé (403 permission-denied, l'app GitHub connectée n'avait que l'accès lecture). Rapporté à l'utilisateur, aucun état cassé laissé dans le repo (comportement voulu : ne jamais committer un pair audio/feed incomplet).
2. **Fix accès GitHub** : résolu côté utilisateur (permissions de l'app/connecteur GitHub corrigées pour avoir l'écriture sur `MagiCapSss/daily-news`).
3. **Config Google Cloud TTS** : utilisateur a créé un projet Google Cloud, activé l'API Text-to-Speech, généré une clé API. Routine mise à jour (prompt réécrit, cf. section TTS ci-dessus) pour utiliser Google Cloud TTS au lieu d'edge-tts.
4. **Run 2** (~2026-07-23 13:59, session `cse_01NN5cKe1r7CvefpAzuU9mto`) : script généré avec succès (~3705 mots, dans la cible). Échec TTS : `403 API_KEY_SERVICE_BLOCKED` sur tous les appels à `texttospeech.googleapis.com`. Rien commité (comportement voulu). Causes possibles diagnostiquées et communiquées à l'utilisateur : (a) API Text-to-Speech pas réellement activée sur le projet, (b) clé API restreinte à d'autres APIs, (c) pas de compte de facturation lié au projet (Google bloque les appels API sans facturation même pour l'usage gratuit).
5. **Run 3** (~2026-07-23 14:20, session `cse_01A8CYr7ARxzMDVAM31D8WDR`) : utilisateur a dit avoir corrigé le souci ("c'est de ma faute") côté Google Cloud Console et a redéclenché la routine. **Statut au moment de l'écriture de ce fichier : pas encore confirmé.** Dernière vérification (git fetch sur le repo local + `RemoteTrigger get`) ne montrait toujours aucun nouveau commit et `ended_reason` toujours vide côté API. À vérifier en premier par le prochain agent — voir section "Prochaines étapes".

## Limitations connues de l'environnement sandbox cloud

- Egress réseau restreint à `*.googleapis.com` + connecteur GitHub (MCP). Toute autre destination réseau directe est bloquée (`403 host_not_allowed`).
- `pip install` et `apt install` ne fonctionnent probablement pas (dépendent de pypi.org / miroirs Debian, hors allowlist) — ne pas en dépendre. Utiliser uniquement les outils déjà présents : `curl`, `python3` (stdlib), `base64`, `git`.
- Pas de `ffmpeg` disponible — d'où la concaténation binaire brute des mp3 plutôt qu'un vrai merge audio.
- Chaque run de routine est une session fraîche sans mémoire des runs précédents — toute information nécessaire à la reprise doit être dans le repo (ce fichier) ou dans le prompt de la routine lui-même.
- Je (l'agent qui gère la routine côté conversation utilisateur) n'ai pas accès aux logs live d'une session cloud en cours — seulement à `ended_reason` (vide/rempli) via `RemoteTrigger get` et à l'état réel du repo via `git fetch`. Aucune certitude sur "en cours" vs "bloqué silencieusement" sans ces deux signaux, ou sans que l'utilisateur consulte directement la page de la routine sur claude.ai.

## Prochaines étapes / TODO pour le prochain agent

1. Vérifier l'état du run 3 (`cse_01A8CYr7ARxzMDVAM31D8WDR`) : `git fetch` + `git log origin/main` sur le repo, et `RemoteTrigger get` sur `trig_01C9gW6gQoeQEeFaQjizWuss` pour voir `ended_reason`.
2. Si réussi : vérifier que `audio/2026-07-23.mp3` fait plusieurs Mo (pas quelques Ko = échec silencieux), que `feed.xml` est du XML valide avec un `<enclosure>` pointant vers la bonne URL GitHub Pages, et que `notes/2026-07-23.md` contient bien le script sourcé.
3. Si toujours échoué : re-diagnostiquer l'erreur exacte retournée par l'API Google TTS (visible dans le run detail sur la page routines) plutôt que de re-essayer à l'aveugle.
4. Une fois un run complet réussi : tester le flux RSS dans AntennaPod (`https://magicapsss.github.io/daily-news/feed.xml`) — l'utilisateur avait eu une erreur 404 avant que `feed.xml` n'existe, à retester.
5. Vérifier que le run automatique quotidien de 6h00 Paris se déclenche correctement les jours suivants sans intervention manuelle.
