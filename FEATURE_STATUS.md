# État des fonctionnalités — Titan Download Manager

Dernière mise à jour : 2026-10-21

Légende :
- ✅ **Implémenté et testé** (tests automatisés ou vérification outillée)
- 🟡 **Implémenté, à valider en usage réel** (compilé et intégré, mais pas encore exercé manuellement)
- ⏳ **Non implémenté** (l'interface l'indique explicitement via « bientôt » ou un interrupteur désactivé)

## Preuves de vérification

| Vérification | Commande | Résultat |
|---|---|---|
| Compilation workspace | `cargo check --workspace` | 0 erreur, 0 avertissement |
| Tests unitaires | `cargo test --workspace -- --test-threads=2` | **126 tests verts** : 118 unitaires + 8 bout-en-bout (0 échec) |
| Tests bout-en-bout (moteur) | `cargo test -p titan-engine --test e2e_http` | 4 scénarios verts (serveur HTTP réel avec Range : téléchargement segmenté complet + empreinte identique, pause/reprise sans corruption, persistance après redémarrage, restauration des tâches terminées + historique) ; exécution sérialisée (mutex) pour éliminer les faux échecs sous charge |
| Tests bout-en-bout (capture) | `cargo test -p titan-capture` (yt-dlp réel dans le PATH) | 3 scénarios verts avec le vrai yt-dlp.exe : analyse de métadonnées, téléchargement octet à octet (2 Mio), pause/reprise sans corruption (4 Mio) |
| Tests bout-en-bout (torrent anonyme) | `cargo test -p titan-torrent --test proxy_e2e` | 1 scénario vert : serveur SOCKS5 local + tracker HTTP local ; l'annonce tracker transite bien par le proxy (CONNECT vérifié côté serveur) |
| Frontend TypeScript | `npx tsc` dans `apps/desktop` | 0 erreur (strict, noUnusedLocals) |
| Build frontend | `npx vite build` | OK (369 Ko JS, 43 Ko CSS) |
| Build installeur | `npm run tauri build` | MSI 15,98 Mio + NSIS 9,80 Mio (v1.4.0, `target\release\bundle\`), extension navigateur (Chromium + variante Firefox, `content.js` + `media.js`) + hôte natif embarqués |
| Sauvegarde automatique de la base | démarrage de l'application | `%LOCALAPPDATA%\Titan\backups\titan-<AAAAMMJJ>.db` créée au premier démarrage du jour (1/jour, 7 conservées, hors du dossier effacé par le désinstalleur) |
| Restauration automatique | `titan.db` supprimée puis démarrage | Base restaurée depuis la sauvegarde la plus récente avant ouverture SQLite ; tâche et historique retrouvés ; tests unitaires `latest_backup` / `restores_latest_when_database_is_missing` |
| Démarrage réel de l'exe | lancement de `titan-desktop.exe` (10 s) | Processus stable ; `/health` → 1.4.0 ; `titan.db` (WAL), `parts/`, `torrents/`, `bridge.json`, `Downloads\Titan` opérationnels |
| Mise à jour 1.4.0 | NSIS silencieux, application fermée | Base et réglages conservés, sauvegarde recréée, extension/hôte réinstallés |
| Syntaxe extension | `node --check` | OK |
| Overlay vidéo | Brave réel, extension chargée, page HTTP locale | Hôte `titan-media-capture` injecté et positionné sur la vidéo au survol (vérifié par CDP) ; clic réel vers yt-dlp non encore exercé |

## Moteur de téléchargement (crate `titan-engine`)

| Fonctionnalité | État | Détails |
|---|---|---|
| Téléchargement HTTP(S) segmenté | ✅ | Planification de segments, 206 validé octet à octet, test e2e 16 Mo avec empreinte. |
| Pause / reprise | ✅ | Jeton d'annulation, segments actifs remis en attente, reprise sans corruption (test e2e). |
| Reprise après redémarrage | ✅ | État + segments persistés en SQLite ; les tâches terminées sont restaurées visibles dans la liste (test e2e), les tâches interrompues reprennent sans corruption (vérifié en usage réel). |
| File d'attente (FIFO / priorité / petits fichiers / adaptatif) | 🟡 | `pump_queue` + `max_active_downloads` + politiques de tri ; couverture unitaire partielle. |
| Concurrence adaptative | 🟡 | Contrôleur basé sur erreurs/débit (tests unitaires titan-metrics) ; comportement en charge non mesuré. |
| Nouvel essai avec backoff + Retry-After | ✅ | Classification d'erreurs, respect des fenêtres de limitation par hôte (tests unitaires). |
| Limites de débit globales et par tâche | 🟡 | `BandwidthAllocator` câblé ; effet réel non mesuré. |
| Téléchargements sans Content-Length | 🟡 | Chemin `download_unbounded` avec reprise par `If-Range`. |
| Miroirs / sources alternatives | 🟡 | API + planification ; bascule automatique non exercée en e2e. |
| Vérification SHA-256 | ✅ | `verify_file` + vérification à la finalisation ; testée unitairement (`titan-storage`). |
| Renommage atomique part → destination | ✅ | Renommage même volume, jamais d'écrasement silencieux. |
| Réparation / redémarrage (`restart_task`) | 🟡 | Réinitialise segments et part file ; testé via correctif anti-double-exécution. |
| Torrents (magnet + .torrent) | 🟡 | Pont librqbit complet (ajout, pause, reprise, suppression, sélection de fichiers) ; non exercé en réseau réel (pairs). |
| Mode anonyme BitTorrent (proxy SOCKS5 Tor/I2P) | 🟡 | Réglage « Mode anonyme » : tout le trafic TCP pair/tracker passe par un proxy SOCKS5 (`session_options` dans `titan-torrent`) ; DHT, uTP, LSD et écoute entrante désactivés pour ne pas exposer l'IP. Validé par `proxy_e2e` (SOCKS5 local + tracker HTTP : annonce via proxy). À valider avec Tor/I2P réellement actif ; les trackers **UDP** ne passent pas par SOCKS5 (limitation) — préférer les trackers HTTP(S). |
| Détection de média (« Grabber ») | ⏳ | Réglage présent, moteur absent. |
| Planificateur horaire | ⏳ | Réglage présent, automatisation absente. |

## Capture média (crate `titan-capture`)

| Fonctionnalité | État | Détails |
|---|---|---|
| Téléchargement via yt-dlp (YouTube, TikTok, Twitch, X, Instagram…) | ✅ | Processus yt-dlp suivi (`--newline --progress`), progression/débit/ETA parsés ; validé par e2e réel (octets identiques). |
| Analyse d'URL (métadonnées sans téléchargement) | ✅ | `yt-dlp -J`, timeout 60 s, première entrée si playlist ; e2e réel. |
| Pause / reprise | ✅ | Arrêt de l'arbre de processus (`taskkill /T`), `.part` conservé, reprise avec `--continue` ; e2e réel (intégrité vérifiée). |
| Annulation + nettoyage des fichiers partiels | 🟡 | `cancel(delete_files)` + `cleanup_files` ; non couvert par les tests automatisés. |
| Installation de yt-dlp depuis l'interface | 🟡 | Téléchargement 1-clic depuis les releases GitHub, vérification MZ, écriture `.part` puis renommage ; validation manuelle à faire. |
| Choix de format (meilleure qualité / 1080p / 720p / audio) | 🟡 | Sélecteurs adaptés à la présence de ffmpeg ; seul « meilleure qualité » exercé en e2e. |
| Fonctionnement sans ffmpeg | ✅ | Sélecteurs progressifs `b[ext=mp4]/b`, repli documenté dans l'interface ; logique unitaire simple. |
| Validation anti-SSRF des URL de capture | ✅ | `validate_download_url` + `DnsPolicy` identiques au moteur. |

## Réseau et sécurité (crates `titan-http`, `titan-security`)

| Fonctionnalité | État | Détails |
|---|---|---|
| Anti-SSRF (hôtes privés bloqués par défaut) | ✅ | `DnsPolicy::PublicOnly` ; `allow_private_hosts` explicite (utilisé par les tests e2e). |
| Validation d'URL (schémas, ports, identifiants) | ✅ | Tests unitaires dédiés. |
| TLS vérifié | ✅ | Pas de contournement exposé. |
| Proxy (système / manuel / identifiants) | 🟡 | Construction reqwest testée unitairement ; proxy réel non exercé. |
| Cookies / en-têtes personnalisés | ✅ | Transmis par requête, chiffrés au repos via DPAPI (compte Windows) dans SQLite, rejoués uniquement pour la tâche concernée. |
| Délais, limites de taille, redirections contrôlées | 🟡 | Configurés ; cas extrêmes non exercés. |

## API locale (`titan-api`) et pont navigateur

| Fonctionnalité | État | Détails |
|---|---|---|
| API loopback authentifiée (jeton Bearer, `bridge.json`) | ✅ | Comparaison à temps constant, écoute 127.0.0.1 uniquement. |
| Allowlist d'origine (extensions) | ✅ | `chrome-extension://` / `moz-extension://` / `edge-extension://`. |
| Limiteur de débit (120 req/s) | ✅ | Tests unitaires. |
| Endpoints `/health`, `/v1/downloads` (+ pause/resume/cancel) | ✅ | Utilisés par l'hôte natif et l'extension. |
| Endpoints `/v1/capture` et `/v1/captures` | ✅ | Ajout d'une capture yt-dlp et liste des captures ; formats `best`, `1080p`/`p1080`, `720p`/`p720`, `audio`/`m4a` ; URL invalide refusée (400). |
| Hôte natif (`titan-native-host.exe`) | ✅ | Protocole Native Messaging complet, validation de taille, corrélation, erreurs structurées (tests unitaires). |
| `OpenApp` de l'hôte natif | ⏳ | Renvoie ok sans lancer l'application (stub honnête documenté). |

## Extension navigateur (`apps/browser-extension`)

| Fonctionnalité | État | Détails |
|---|---|---|
| Interception par taille + extensions + domaines exclus | 🟡 | Logique complète (`onCreated` → attente `totalBytes` → seuil) ; pas encore chargée dans un vrai Chrome/Edge. |
| Capture directe et discrète au clic | 🟡 | `content.js` (content script) : clic gauche sur un lien de fichier intercepté avant tout téléchargement navigateur ; vérification `HEAD` (type/taille/`Content-Disposition`), envoi direct à Titan, restauration transparente sinon (téléchargement ou navigation). Syntaxe et manifestes vérifiés ; à valider dans un vrai Brave/Chrome. |
| Restauration dans le navigateur si Titan indisponible | 🟡 | Re-téléchargement natif + notification en cas d'échec de l'hôte. |
| Bouton « Titan » au survol des vidéos | 🟡 | `media.js` (Shadow DOM, `all_frames`) : bouton positionné sur les lecteurs ≥ 240×135, envoi de `location.href` au moteur de capture, toast de résultat, domaines exclus et activation pris en compte à chaud. Injection/position vérifiées dans Brave (CDP) ; capture d'une vidéo réelle à valider. |
| Menus contextuels (lien, vidéo/audio) | 🟡 | Créés à l'installation ; le menu média envoie `add_media` avec l'URL de la page (repli sur `srcUrl`), transmis à `/v1/capture`. |
| Popup : statut, liste, pause/reprise, réglages | 🟡 | Interface complète ; à valider visuellement. |
| ID d'extension épinglé + script d'installation de l'hôte | ✅ | `key` dans le manifeste (ID stable), `install-host.ps1` (Chrome/Edge, HKCU). |
| Installation de l'hôte natif depuis l'application | 🟡 | Un bouton : copie de `titan-native-host.exe` dans `%LOCALAPPDATA%\Titan`, copie stable de l'extension (survit aux mises à jour), manifeste Chromium (ID épinglé) + manifeste Firefox (`allowed_extensions`, `%APPDATA%\Mozilla\NativeMessagingHosts`), clés HKCU Chrome/Edge/Brave via `reg.exe` ; à valider sur machine réelle. |
| Assistant d'installation par navigateur | 🟡 | Boutons Chrome / Edge / Brave / Firefox : ouverture du dossier de l'extension + page des extensions ; Chromium tente un chargement immédiat (`--load-extension`), Firefox ouvre `about:debugging` (module temporaire, non signé). |
| Brave | 🟡 | Même flux Chromium que Chrome/Edge (ID d'extension et schéma identiques) ; hôte natif enregistré sous `HKCU\Software\BraveSoftware\Brave-Browser` ; non encore chargé dans un vrai Brave. |
| Firefox (variante Gecko) | 🟡 | `browser-extension/firefox/` (manifeste MV3 Gecko, background event page, ID `titan@titan-download-manager`), hôte natif via fichier de profil ; installation permanente impossible sans signature AMO → chargement temporaire documenté. |
| Interception des liens magnet | ⏳ | Impossible via l'API `downloads` ; à ajouter depuis Titan. |

## Interface desktop (`apps/desktop`)

| Fonctionnalité | État | Détails |
|---|---|---|
| Vue Téléchargements (tableau, tri, filtres, recherche) | 🟡 | Complète dans le code ; première exécution visuelle en attente. |
| Vue Torrents (fichiers, sélection, pause/reprise) | 🟡 | Idem. |
| Panneau de détails (segments, sources, historique, vérification) | 🟡 | Idem. |
| Performances (débit global, anneaux CPU/RAM/disque, statistiques) | 🟡 | Graphiques SVG maison alimentés par `useEngine`. |
| Historique + benchmarks | ✅ | Lecture SQLite + exécution de benchmarks réels ; chaque téléchargement terminé est désormais enregistré dans `download_history` (correctif 1.1.2, test e2e). |
| Catégories (CRUD + règles) | 🟡 | Idem. |
| Réglages (profils, réseau, torrents, apparence…) | 🟡 | Les options non implémentées sont désactivées avec la mention « bientôt ». |
| Notifications système Windows | 🟡 | Plugin notification câblé aux événements du moteur (fin/échec si activé). |
| Icône de la zone de notification (tray) | ✅ | Icône permanente + menu (Afficher, Tout mettre en pause, Tout démarrer, Quitter), clic gauche = bascule ; « Réduire » masque la fenêtre et la restauration fonctionne (vérifié par UI Automation : fenêtre `iconic`, puis `SW_RESTORE`). |
| Lancement avec Windows | 🟡 | `tauri-plugin-autostart` synchronisé avec `start_with_windows` (démarrage masqué `--hidden`) ; validation sur machine réelle à faire. |
| Capture média (page dédiée) | 🟡 | Page « Capture média » : installation de yt-dlp, analyse d'URL, choix qualité/dossier, suivi des captures (pause/reprise/annulation, révélation du fichier). |
| Version de l'interface | ✅ | `vite.config.ts` lit désormais `package.json` (`__APP_VERSION__`) au lieu de la valeur codée en dur `1.0.0` ; la page Paramètres affiche `Titan v1.4.0`. |
| Optimisation matérielle | 🟡 | Commande Rust `system_profile` (`sysinfo`) : cœurs physiques/logiques, RAM totale/disponible et recommandation affichée dans Paramètres → Ressources système ; « Optimiser » applique le profil custom (connexions, segmentation). Détection vérifiée à l'écran ; application à valider par un téléchargement réel. |
| Thèmes d'interface | ✅ | 6 thèmes : Système, Sombre, Clair, Minuit, Océan, Braise — cartes avec aperçu, variables CSS `data-theme`, surcharges claires pour inputs, boutons, menus, dialogues, tableau et scrollbars. Thème clair vérifié par capture `PrintWindow` (plus de surfaces sombres résiduelles) ; application immédiate après enregistrement. |
| Planificateur | ⏳ | Entrée de navigation désactivée (« bientôt »). |
| Internationalisation | ⏳ | Français uniquement. |

## Distribution

| Élément | État | Détails |
|---|---|---|
| Configuration installeur MSI + NSIS | ✅ | `tauri.conf.json` (`bundle.targets`) + icônes complètes. |
| Installeur construit et vérifié | ✅ | v1.4.0 : `npm run tauri build` produit MSI (15,98 Mio) et NSIS (9,80 Mio) incluant `titan-native-host.exe`, l'extension Chromium (`content.js` + `media.js`) et la variante Firefox ; installation silencieuse NSIS vérifiée (FileVersion 1.4.0, `/health` → 1.4.0, auto-réparation de l'hôte/extension, sauvegarde quotidienne recréée, données conservées avec l'application fermée). Copies livrées dans `Titan download manager\`. |
| Test d'installation réelle du paquet | ⏳ | Non installé sur une machine vierge (WebView2 en `downloadBootstrapper`). |
| Signature de code | ⏳ | Nécessite un certificat ; non fourni. |
| Mise à jour automatique | ⏳ | Non implémentée. |

## Correctifs notables issus des tests bout-en-bout

1. **Double exécution d'une tâche** (`kick_task`) : le drapeau `running` était positionné à l'intérieur de la future asynchrone ; `start_task → pump_queue` pouvait lancer un second exécutant avant la planification du premier (téléchargement en double, pause inefficace, échec de vérification de taille). Correction : revendication synchrone du drapeau avant le spawn. Détecté et validé par `e2e_http`.
2. **Vitesses torrent** : `Speed.mbps` de librqbit converti en octets/s via `as_bytes()` (l'ancien code publiait des Mo/s comme des octets/s).
3. **Pause yt-dlp** : `yt-dlp.exe` est un bootloader PyInstaller qui télécharge dans un processus enfant ; tuer le seul parent laissait l'enfant actif, verrouillant le `.part` (échec du renommage à la reprise). Correction : arrêt de l'arbre complet (`taskkill /T /F`) puis attente de la fin effective avant de signaler la pause. Détecté et validé par `ytdlp_e2e` (pause/reprise de 4 Mio avec yt-dlp réel).
4. **Empaquetage des ressources** : le motif glob unique `browser-extension/**/*` aplatissait les sous-dossiers dans le MSI (collision ICE30 `popup.js` entre la racine et `firefox/`). Correction : entrées explicites par dossier (`icons/`, `native/`, `firefox/`, `firefox/icons/`).
5. **« Titan indisponible » à répétition** (1.1.1/1.1.2) : le `LocalApiHandle` était libéré juste après le démarrage de l'API locale (arrêt gracieux axum immédiat) alors que `bridge.json` annonçait un port mort. Correction : le handle est conservé dans l'état de l'application et `bridge.json` est supprimé proprement à la sortie (si port + jeton correspondent).
6. **Hôte natif sans lancement automatique** (1.1.1/1.1.2) : l'action `open_app` ne faisait rien et l'extension échouait en « Titan indisponible ». Correction : l'hôte vérifie `/health`, lance Titan si absent (`--hidden`, processus détaché sans fenêtre), réessaie une fois et attend jusqu'à 15 s.
7. **« error decoding response body » sur pause/reprise** (1.1.1/1.1.2) : l'API répond 200 avec un corps vide ; l'hôte échouait au décodage JSON. Correction : corps vide traité comme `{"ok": true}`.
8. **Copie de l'hôte verrouillée** (1.1.1/1.1.2) : la mise à jour échouait si un hôte natif tournait (Brave/Chrome gardent le processus). Correction : copie non bloquante si la cible existe + auto-réparation au démarrage (hôte, copie stable de l'extension, manifestes, clés de registre).
9. **Téléchargements terminés invisibles après redémarrage + historique vide** (1.1.2) : la restauration ignorait les tâches `completed`/`cancelled` et `insert_history` n'était jamais appelé. Correction : les tâches terminées sont restaurées sans être relancées, et chaque fin de téléchargement alimente `download_history` (test e2e dédié).
10. **Sonde initiale non retentée** (1.2.0) : si la requête de sondage (HEAD/GET initial) tombait sur une coupure transitoire (`ConnectionReset`), la tâche échouait définitivement même avec une erreur classée réessayable. Correction : boucle de nouvelle tentative appliquant la politique d'essais/backoff des réglages + `record_retry` ; les tests e2e moteur sont en outre sérialisés (mutex) pour supprimer les faux échecs en exécution parallèle (5 exécutions consécutives vertes).
11. **Auto-réparation de l'extension se copiait sur elle-même** (1.2.0 → corrigé en 1.2.1) : la source de copie préférait la copie stable puis la supprimait avant la copie — la copie stable de l'extension (`%LOCALAPPDATA%\Titan\browser-extension`) se vidait à chaque mise à jour. Correction : la source est désormais uniquement l'ensemble embarqué/ressources, copie dans `browser-extension.new` puis bascule (renommage) atomique — un échec ne détruit plus la copie existante. Vérifié sur installation réelle 1.2.1 (copie stable mise à jour intacte au démarrage).
12. **Mise à jour NSIS destructrice** (constat 1.3.0) : lors de l'installation de 1.3.0, le désinstalleur de l'ancienne version a poursuivi son nettoyage après la fin de l'installation (dossier d'installation et clé de désinstallation supprimés environ une minute plus tard) et a effacé les données utilisateur (`%APPDATA%\com.titan.downloadmanager` : liste des tâches, historique, réglages) — **les fichiers déjà téléchargés sont intacts**. Mitigation : sauvegarde quotidienne automatique dans `%LOCALAPPDATA%\Titan\backups` (7 copies, hors du dossier effacé) + **restauration automatique** si `titan.db` manque au démarrage (1.4.0). Testé : base supprimée puis application relancée → base et tâche restaurées. La mise à jour 1.4.0 avec l'application fermée a conservé les données ; la course du désinstalleur avec une application ouverte reste à corriger dans le template NSIS.
13. **Version UI figée à 1.0.0** (1.4.0) : `vite.config.ts` injectait `__APP_VERSION__` en dur. Correction : lecture de `package.json` ; l'interface et la page Paramètres affichent 1.4.0.
14. **Thème clair incohérent** (1.4.0) : inputs, boutons, menus, dialogues et tableau conservaient des fonds sombres codés en dur. Correction : variables `--input-*`, `--btn-*`, `--menu-*`, `--dialog-*`, `--thead-bg`, `--track-bg`, `--switch-off` avec surcharges `[data-theme="light"]` ; rendu vérifié par capture d'écran.
15. **Overlay vidéo absent** (1.4.0) : aucun bouton de capture au survol. Correction : `media.js` en Shadow DOM, positionné sur les vidéos, activable/désactivable et respectant les domaines exclus ; vérifié dans Brave sur une page de test.

## Audit sécurité / erreurs (vérifié)

- **API locale** : tous les points d'entrée exigent le jeton (comparaison à temps constant) et l'allowlist d'origine ; limite de débit (120 req/s) étendue à `pause`/`resume`/`cancel` (auparavant `add_download` seul).
- **Secrets au repos** : cookies, referrer, en-têtes personnalisés et mot de passe proxy chiffrés via DPAPI (compte Windows) ; migration transparente des bases existantes (valeurs en clair relues puis réécrites chiffrées). Test de régression `secrets_are_not_plaintext_at_rest` (titan-db).
- **Injection d'en-têtes** : en-têtes dangereux filtrés, CRLF rejetés (`HeaderValue`), cookies invalides refusés.
- **Anti-SSRF** : résolution DNS + refus des adresses privées par défaut (`DnsPolicy::PublicOnly`, option explicite sinon).
- **Chemins** : `sanitize_filename` (traversée, noms réservés Windows, 240 octets) ; `reveal_in_explorer` échappe désormais le chemin (`raw_arg`) et rejette les guillemets.
- **Erreurs** : aucun `unwrap`/`expect` de production hors entrée `run()` ; états restaurés normalisés (Queued/Paused), segments réconciliés au démarrage.
- **UI ↔ Rust** : les 47 commandes Tauri invoquées par l'interface existent avec les bons arguments (camelCase) ; événements `titan://event` alignés (9 variantes).
- **Registre de l'hôte natif** : écriture sous `HKCU` uniquement (aucun droit administrateur), valeurs citées via `reg.exe` (pas d'interpolation PowerShell), manifeste limité à l'ID d'extension épinglé ; Firefox utilise un manifeste de profil (`allowed_extensions`, ID Gecko) sans registre.
