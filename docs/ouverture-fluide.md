# Ouverture de profils — chemin critique

Avant ce passage, `launch_profile_synced` attendait le réseau **avant** de spawn ShardX :

- sonde UDP SOCKS5 (`probe_udp`) : timeout TCP 8 s + STUN
- `geo_check` / `geo_check_via` : jusqu’à 3 fournisseurs × 8 s, même si un test proxy était déjà en cache
- `touch_launched` (verrou fichier + rewrite JSON) sur le retour IPC
- flotte et sync : ouvertures strictement séquentielles
- UI : le bouton restait occupé et la ligne ne passait « running » qu’au poll de 2 s

Le spawn n’est plus tenu par ça.

- UDP / IP publique : cache du dernier test. Sans cache, budget 280 ms, puis UDP laissé actif (un faux négatif coupait QUIC/WebRTC).
- timezone / langue / géoloc `auto` : snapshot en cache, sinon le même budget, sinon tag pays / UTC.
- Les deux résolutions partagent un `tokio::join!`.
- `SingletonLock` orphelin (pid mort, y compris symlink cassé) est retiré avant spawn.
- `last_launched_at` est écrit après le spawn, hors du retour IPC.
- Poll CDP : 12 ms puis backoff jusqu’à 80 ms.
- Settings : cache mémoire invalidé par mtime.
- Sync / bulk : 3 profils en parallèle.
- UI : la ligne passe running au clic ; poll à 800 ms.

Rebuild : `npm install && npm run tauri build` (ou `tauri dev`) depuis la racine du projet.

# Profils headless

L'UI ne lançait que des fenêtres. Le headless n'existait que sur `POST /profiles/:id/start`.

- Commande `launch_headless` : `--headless=new` + CDP, même bus que l'ouverture normale.
- Le tracker expose `headless` et l'endpoint CDP dans `process_list`.
- Tableau : menu « Launch headless », badge, filtre Headless, action groupée, copie de l'URL CDP.
- Un profil ouvert headless par l'API locale apparaît de la même façon.
