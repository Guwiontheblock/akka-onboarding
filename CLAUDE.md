# Akka — Funnel V0 · CLAUDE.md

## Projet
Funnel d'onboarding quiz (mobile-first, plein écran) pour **Akka** (akka.app), plateforme d'investissement. **Produit régulé (AMF)** → prudence maximale sur toute allégation.
Spec complète et à jour (source de vérité — la lire avant toute tâche) : @akka_funnel_v0_prototype_spec.md
En ligne (preview) : https://akka-onboarding.netlify.app · Cible pilote : sous-domaine **join.akka.app**.

## Stack
- **Vite + React 18** (`@vitejs/plugin-react`), dépendance **`libphonenumber-js`**.
- Composant actuel : `src/AkkaFunnelV0.jsx` (monolithe, default export `AkkaFunnelV0`).
- Entrée `src/main.jsx` · Build `npm run build` (→ `dist/`) · Déploiement Netlify : `npx netlify-cli deploy --prod --dir=dist`.

## Règles (IMPORTANT)
- **Ne JAMAIS modifier le rendu visuel lors d'un refactor.** Le « build propre » doit rester **pixel-identique** ; le vérifier avant de conclure.
- **Changements minimaux** : ne pas refactorer du code hors de la tâche en cours.
- **Un commit par changement logique**, messages en **français**.
- **Conformité AMF** : ne jamais inventer de chiffres de perf / social proof / témoignages. Toute modif de copy vient de la spec ou d'une demande explicite (voir §10 de la spec).
- **Copy du funnel = EN, ne pas traduire.** Discussion et commits = FR.
- **Toujours `npm run build` après une modif**, vérifier qu'il passe, comparer le poids du bundle. Skill `verification-before-completion`.
- **Plan d'abord** : pour toute tâche, proposer un plan (mode plan / skill `writing-plans`) et **attendre validation** avant d'exécuter.

## Faits d'intégration (à respecter au mot près)
- **Webhook Make** : POST `{ keepalive: true }` vers `https://hook.eu2.make.com/iwotrkjtymboayre3o2kmlkk3pbmmd9d` ; payload **plat & complet** (toutes les clés, `investor_*` incluses même vides), **avant** la redirection. Premier envoi = structure de référence à figer.
- **Téléphone** : sélecteur custom recherchable (`CountrySelect`) listant **tous** les pays (libphonenumber-js + `Intl.DisplayNames` pour les noms). Toujours envoyer le **E.164** via `parsePhoneNumberFromString(form.phone, country.code).number` (jamais `form.phone` brut) — pour **iClosed ET le webhook**.
- **Redirections** : `final` (form) → `https://www.akka.app/join-form-confirmation` (via écran `redirecting`, in-app, sans flash). iClosed → `https://www.akka.app/join-call-confirmation` (réglé dans le **dashboard iClosed**, pas dans le code).
- **iClosed embed** (`final_call`) : data-url `https://app.iclosed.io/e/akka/strategy-call-2` ; mapping `iclosedName/Email/Phone` + UTM ; script tiers **invisible en localhost** (normal).
- **Branchement de fin** : `loader` → `lt_3k ? final : final_pre_call → final_call`.
- **preipo** : grille 3×3 de **logos de marque** (9 SVG inlinés base64, rendus blanc monochrome) ; table `PREIPO_LOGO_FIT` pour la taille par logo.

## Hébergement & embed (option retenue)
- **Page publique** : page Webflow dédiée (blanche, sans nav/footer) sur **akka.app** (ex. `akka.app/join`) → **pas de sous-domaine** pour le funnel ; tout est sur akka.app (évite le cross-domain).
- **Assets** (JS/CSS/fonts/logos/loupe) sur le **VPS Contabo** (Nginx, Ubuntu 24.04, IP **173.249.25.190**) sous **`cdn.akka.app`** — vhost **isolé** de `commande.groupe-pineau.fr` (PM2/Postgres). DNS A `cdn.akka.app → 173.249.25.190` **déjà posé sur Cloudflare**.
- **Cloudflare** : si l'enregistrement reste **proxied (orange)**, on gagne un **cache edge CDN** (atténue le poids du bundle mondialement) → **Origin Certificate** + SSL **Full (strict)**. Sinon **DNS-only (grey)** + Let's Encrypt.
- **Embed Webflow** = `<div id="akka-funnel-root"></div>` + `<link rel="stylesheet" href="https://cdn.akka.app/akka-funnel.css">` + `<script src="https://cdn.akka.app/akka-funnel.js" defer>`.
- Monté **directement** (pas en iframe) → `window.location` (redirect), iClosed, **capture UTM/gclid** lisent la vraie URL akka.app → natif.
- **Aligner l'hôte des redirections** (`final` → `…/join-form-confirmation`, iClosed → `…/join-call-confirmation`) sur le **canonical** akka.app (apex vs `www`) pour éviter un saut www↔apex. Actuellement codé en `www.akka.app` → confirmer/ajuster.
- **CSP akka.app** (si Webflow en pose une) : autoriser `script-src`/`style-src`/`font-src` `https://cdn.akka.app` et `connect-src` `https://ipwho.is https://app.iclosed.io https://hook.eu2.make.com`.

## Ordre des tâches (une à la fois, avec validation)
1. **Build propre + build embeddable** — décomposer le monolithe en modules (composants / écrans / hooks / data) ; **extraire les assets base64 déjà embarqués** (loupe PNG, étoiles SVG, **9 logos preipo SVG**, polices PP Pangram & Satoshi) en les **décodant** vers `public/` (svg/png + woff2), `@font-face` + preload ; code-split. **Rendu inchangé.** Cible : faire chuter le bundle (actuellement ≥ ~1 Mo non-gzip).
   Produire **deux sorties** :
   - **(a) standalone** (test / Netlify) — montage sur `#root`, noms hashés OK.
   - **(b) embeddable Webflow** (`npm run build:embed`) — entrée dédiée `src/embed.jsx` montant `<AkkaFunnelV0/>` sur `#akka-funnel-root` ; **noms de fichiers FIXES** `akka-funnel.js` / `akka-funnel.css` (pas de hash, pour ne pas changer l'embed Webflow à chaque deploy) ; `base: "https://cdn.akka.app/"` (fonts/logos/loupe chargés depuis le CDN) ; React bundlé ; **styles scopés** au conteneur (ne pas casser la page Webflow ni imposer un reset html/body agressif).
2. **Webhook Make** (cf. faits ci-dessus).
3. **Tracking GA4/GTM** (dataLayer : `funnel_start` + UTM / `funnel_step_view {step_index, step_id, investor_type}` / `funnel_answer {step_id, field, value}` / `generate_lead` au POST OK ; objet `STEP_META` ; capture + persistance UTM).
4. **Conformité** — consentement RGPD/AMF sur les formulaires + points §10 de la spec.
5. **Hébergement VPS `cdn.akka.app` + embed Webflow** — servir le build embeddable depuis le VPS, monter le funnel dans `akka.app/join`. **Vhost 100% isolé de Groupe Pineau** (ne PAS toucher à sa conf Nginx / PM2 / Postgres / CSP). **Credentials VPS & Cloudflare HORS repo** (clés SSH, tokens en env ; jamais committer — l'autre projet a eu une fuite SSH, ne pas reproduire). Hook `gitleaks` pré-commit recommandé.
   1. `mkdir -p /var/www/akka-funnel` (root du vhost).
   2. Vhost Nginx `cdn.akka.app` (fichier dédié dans `sites-available`, symlink, `nginx -t && systemctl reload nginx`) : CORS `Access-Control-Allow-Origin: https://akka.app` + `Cross-Origin-Resource-Policy: cross-origin` ; **`akka-funnel.js`/`.css` en `Cache-Control: no-cache`** (noms fixes → propager les deploys) ; **assets fingerprintés (fonts/logos/loupe) `immutable` 1 an** ; gzip/brotli.
   3. SSL : **Origin Certificate Cloudflare + SSL Full(strict)** (si proxied) **ou** `certbot --nginx -d cdn.akka.app` (si DNS-only).
   4. `deploy-embed.sh` : `npm run build:embed` → `rsync -avz --delete dist-embed/ USER@173.249.25.190:/var/www/akka-funnel/` → (si proxied) **purge du cache Cloudflare** des 2 fichiers d'entrée via API (token en variable d'env, jamais en clair).
   5. Page Webflow `akka.app/join` (blanche) : coller l'embed (div + link + script).
   6. Vérifs : zéro erreur CORS en console ; fonts/logos chargés depuis `cdn.akka.app` ; redirect confirmation + iClosed + capture UTM OK sur la vraie URL.
6. **GA4/GTM sur akka.app + Optibase A/B** — tout étant sur **akka.app**, **plus de cross-domain** : le GTM/GA4 du site principal couvre le funnel et reçoit les events dataLayer (Tâche 3) ; conversion = visite de `akka.app/join-form-confirmation`. Baseline GA4 + métrique-relais, puis **Optibase** (split URL control vs variante, objectif bayésien P2BB).

## Skills & outils recommandés (environnement Claude Code)
> Choix d'**environnement** (à installer une fois), distincts des règles projet ci-dessus. Rester sélectif : trop de skills auto-déclenchées = bruit + tokens.

**Déjà installés, utiles ici** : `verification-before-completion`, `writing-plans`, `systematic-debugging`, `test-driven-development`, `git-pushing`, `owasp-security`, `vibesec-skill`, `webapp-testing`, `design-auditor`, `oiloil-ui-ux-guide`.

**À ajouter (net-new), mappés aux tâches** :
- **`using-git-worktrees` + `finishing-a-development-branch`** — faire les tâches 1→6 une par branche, proprement.
- **`agnix`** — linter qui valide ce `CLAUDE.md` / les SKILL.md / hooks.
- **`review-claudemd`** — garder ce fichier affûté au fil du projet.
- **`varlock-claude-skill`** — secrets hors sessions/logs/git (webhook Make, IDs GA4/GTM) — tâches 2, 3, 5.
- **`sanitize`** — masquer la PII (email/téléphone), conformité RGPD/AMF — tâche 4.
- **`review-implementing` + `test-fixing`** — évaluer un plan vs la spec, réparer les tests cassés.
- **`email-html-mjml`** — emails de confirmation Brevo (HTML responsive) quand le webhook → Brevo sera câblé.

**Outils (non-skills)** : `gitleaks` (hook pré-commit anti-secrets) — déjà cité en Tâche 5.

**Top 3 pour démarrer** : `using-git-worktrees`, `varlock`, `agnix`. Le reste s'ajoute quand la tâche correspondante arrive.

**Côté claude.ai (hors repo, stratégie / copy / CRO)** : `wondelai/skills` (hypothèses A/B, copy de conversion), `avoid-ai-writing` (humaniser la copy funnel/emails).
