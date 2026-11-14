# Incident : la page des promotions affichait « 403 Forbidden » et le stock « 502 Bad Gateway » après la publication de novembre

- **Date** : 14 novembre 2026
- **Durée** : environ 1 heure, de 16 h 28 à 17 h 28 (CET, UTC+1). Le 403 a duré environ 9 minutes (jusqu'à vers 16 h 37), le 502 environ 1 heure.
- **Rédigé par** : Saïd Mounpain

## Impact

Après la publication de la version de novembre sur le serveur de pré-production de Mboa Market, deux fonctions ont cessé de marcher en même temps pour les visiteurs de la boutique :

- la page des promotions (`/promo.html`) répondait **403 Forbidden**, de 16 h 28 à environ 16 h 37 ;
- le stock ne s'affichait plus : `/api/stock` répondait **502 Bad Gateway**, de 16 h 28 à 17 h 28.

Le reste du site (page d'accueil, feuille de style, anciennes images) continuait de fonctionner. Le service client a signalé la page des promotions en 403. Le nombre de visiteurs touchés n'a pas été mesuré.

## Chronologie

Toutes les heures sont en CET (UTC+1).

| Heure | Événement |
| --- | --- |
| 16:28:45 | Publication de la version de novembre avec `mboa-publier`. Le fichier `promo.html` est créé (`Last-Modified` de nginx : 16:28:45). |
| 16:28:46 – 16:28:47 | L'API du stock démarre cinq fois de suite et plante à chaque fois (`KeyError: 'STOCK_FICHIER'`). À 16:28:47, systemd abandonne : « Start request repeated too quickly ». Le service passe en `failed`. |
| 16:30:07 – 16:33:39 | Le journal d'erreurs de nginx (`/var/log/nginx/mboamarket.error.log`) montre deux types d'erreurs : `connect() failed (111: Connection refused)` vers `127.0.0.1:8090` pour `/api/stock`, et `open() "/srv/mboamarket/site/promo.html" failed (13: Permission denied)`. |
| 16:33:39 | Diagnostic du 403 : `curl -I http://localhost/promo.html` renvoie 403 ; `sudo tail -n 5 /var/log/nginx/mboamarket.error.log` indique « Permission denied » ; `ls -l /srv/mboamarket/site/promo.html` affiche `-rw------- 1 root web`. |
| vers 16:37 | Correction du 403 (`chmod 664`). `curl -I http://localhost/promo.html` renvoie 200 OK (en-tête `Date: 15:37:44 GMT`). |
| vers 17:07 (déduit de « 38min ago ») | Diagnostic du 502 : `systemctl status mboa-stock` affiche `failed (Result: exit-code)`, puis `sudo journalctl -u mboa-stock --no-pager -n 20` montre `KeyError: 'STOCK_FICHIER'` à la ligne 21 de `stock.py`. |
| 17:28:23 | Correction du 502 : ajout de la variable d'environnement dans l'unité systemd, `daemon-reload`, `restart`. Le service est `active (running)`. |
| heure non notée | Analyse de l'outil `mboa-publier` : `umask 077` trouvé et remplacé par `umask 002`. `sudo mboa-publier --essai` confirme des fichiers en 664 et des dossiers en 775. |

## Causes racines

### Le 403 sur `/promo.html`

- **Ce que les journaux ont montré** : nginx n'a pas pu ouvrir le fichier (`failed (13: Permission denied)`). `ls -l` donne `-rw------- root web` : seul root peut le lire. Les processus de travail de nginx tournent sous `www-data`, qui n'a aucun droit sur ce fichier.
- **Pourquoi** : l'outil `mboa-publier` s'exécute en root avec sudo et contient `umask 077` (« les fichiers de la livraison restent privés pendant la copie »). Tout fichier **nouveau** qu'il crée naît donc en 600 (et tout dossier en 700), propriétaire root. Le dossier du site a le SGID (`2775`), donc le groupe `web` est hérité, mais le mode 600 ne donne aucun droit au groupe. La première version du site (v1) fonctionnait parce que les fichiers avaient été copiés par Nadia avec des droits lisibles par tous.
- **Pourquoi personne ne l'a vu avant** : la publication ne vérifie pas que nginx peut lire ce qui vient d'être publié, et rien n'alerte quand une page répond 403.

### Le 502 sur `/api/stock`

- **Ce que les journaux ont montré** : nginx n'arrivait pas à joindre l'API (`connect() failed (111: Connection refused)` vers `127.0.0.1:8090`). Le service `mboa-stock` était en `failed`. Son journal contient `KeyError: 'STOCK_FICHIER'` : le code lit `os.environ["STOCK_FICHIER"]` sans valeur par défaut, et cette variable n'était pas définie.
- **Pourquoi** : l'unité systemd `mboa-stock.service` ne définissait pas `STOCK_FICHIER`, que la nouvelle version de l'API exige. L'API plantait donc dès son démarrage. `Restart=on-failure` a relancé le service cinq fois en moins d'une seconde, puis systemd a renoncé (« Start request repeated too quickly »). Une relance automatique ne peut pas réparer une erreur de configuration qui se reproduit à chaque démarrage.
- **Pourquoi personne ne l'a vu avant** : la publication ne contrôle ni l'état du service ni la réponse de l'API après la mise à jour, et rien ne surveille l'état `failed`.

## Correction

**403 : rendre le fichier lisible par nginx**

```bash
sudo chmod 664 /srv/mboamarket/site/promo.html
curl -I http://localhost/promo.html
```

Résultat : `HTTP/1.1 200 OK`.

**502 : fournir la variable d'environnement à l'API**

```bash
sudo nano /etc/systemd/system/mboa-stock.service
```

Ajout dans la section `[Service]` :

```ini
Environment="STOCK_FICHIER=/srv/mboamarket/stock/stock.csv"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart mboa-stock
systemctl status mboa-stock
```

Résultat : `active (running)`. Vérification de la réponse de l'API : `curl http://localhost/api/stock` (à lancer et à coller ici, la sortie n'est pas dans le journal de bord).

**Cause commune côté outil : `mboa-publier`**

```bash
sudo nano /usr/local/bin/mboa-publier   # umask 077 remplacé par umask 002
sudo mboa-publier --essai
```

Résultat de l'essai : fichiers en `-rw-rw-r--`, dossier `images` en `drwxrwxr-x`.

## Pour que ça ne revienne pas

Les responsables sont proposés et à confirmer avec l'équipe.

| Action | Responsable |
| --- | --- |
| Vérifier que tous les fichiers créés par la publication de novembre sont lisibles par nginx (par exemple `images/novembre.svg`) : `ls -l /srv/mboamarket/site /srv/mboamarket/site/images` | Saïd |
| Dans `mboa-publier`, fixer explicitement les droits après la copie (fichiers 644 ou 664, dossiers 755 ou 775) au lieu de dépendre du `umask` | Koffi |
| Ajouter à `mboa-publier` une vérification finale : chaque page publiée est lisible par `www-data`, `systemctl is-active mboa-stock` est vrai, `curl -f http://localhost/api/stock` répond 200. En cas d'échec, l'outil se termine avec un code d'erreur | Koffi |
| Versionner l'unité `mboa-stock.service` dans le dépôt `mboa-market-serveur` et y documenter les variables d'environnement obligatoires (`STOCK_FICHIER`) | Saïd |
| Faire échouer l'API avec un message clair quand `STOCK_FICHIER` manque (ou prévoir une valeur par défaut) | Koffi |
| Surveiller les réponses 5xx et l'état `failed` des services (par exemple une vérification de `/api/stock` toutes les minutes) | Saïd |
| Avant chaque publication, lancer `sudo mboa-publier --essai` et relire les droits affichés | Koffi et Nadia |

---

Ce post-mortem ne cherche pas de coupable : il cherche ce qui, dans les outils et les habitudes, a permis l'incident.
