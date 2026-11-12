# RDP Windows via GitHub Actions + Tailscale

Un bureau Windows complet, accessible à la demande en Bureau à distance (RDP), sur un runner GitHub Actions. Le port RDP n'est **jamais** exposé sur Internet : tout passe par votre réseau privé Tailscale.

> **Utilisez un dépôt privé** (ou au minimum un compte GitHub dédié). Les journaux d'un dépôt public sont lisibles par tout le monde (IP Tailscale, nom d'utilisateur).

## Ce qui a été corrigé par rapport à la version d'origine

| Problème | Correction |
|---|---|
| **RDP qui tombe après 1 à 2 h** | Tailscale est lancé avec `--unattended` : le tunnel reste actif même sans session Windows ouverte. |
| Sessions coupées par Windows | Plus aucune limite d'inactivité, de déconnexion ou de durée de session ; keep-alive RDP activé ; veille, hibernation et verrouillage d'écran désactivés. |
| Service RDP qui peut s'arrêter | Redémarrage automatique du service (`sc failure`) + surveillance toutes les 30 s. |
| Tunnel Tailscale qui peut tomber | La boucle de maintien détecte un tunnel hors ligne et le reconnecte toute seule. |
| `timeout-minutes: 3600` | Valeur invalide : GitHub limite les runners hébergés à 360 min (6 h). Remplacée par `360`, avec une durée de session réglable. |
| Mot de passe affiché dans les logs et écrit dans `GITHUB_ENV` | Supprimé : le mot de passe n'est jamais affiché ni exporté. |
| NLA et couche de sécurité désactivés | NLA + TLS activés. |
| Port 3389 ouvert à tout le monde | Autorisé uniquement depuis `100.64.0.0/10` (réseau Tailscale). |
| Tailscale 1.82.0 figé | Dernière version stable téléchargée automatiquement, avec nouvelles tentatives. |
| Machine qui reste dans la console Tailscale | `tailscale logout` en fin de job (même en cas d'échec ou d'annulation). |
| **Session limitée à 6 h** | **Relais automatique** : en fin de session, le workflow se relance tout seul (voir ci-dessous). |

## Installation

1. Créez un **dépôt privé** et placez les fichiers ainsi :
   ```
   .github/workflows/rdp.yml
   README.md
   .gitignore
   ```
2. Créez une clé d'authentification Tailscale : <https://login.tailscale.com/admin/settings/keys>
   - cochez **Reusable** ;
   - cochez **Ephemeral** (la machine disparaît toute seule du Tailnet à la fin) ;
   - copiez la clé `tskey-auth-...`.
3. Dans le dépôt : **Settings > Secrets and variables > Actions > New repository secret** :

   | Secret | Contenu |
   |---|---|
   | `TAILSCALE_AUTH_KEY` | la clé `tskey-auth-...` |
   | `RDP_USERNAME` | votre nom d'utilisateur (20 caractères max). Optionnel : `kipre10` par défaut |
   | `RDP_PASSWORD` | un mot de passe long : **14 caractères minimum**, avec au moins 3 types (minuscules, majuscules, chiffres, symboles). Une phrase de passe fonctionne très bien. |

## Utilisation

1. Onglet **Actions** > **RDP Tailscale (stable)** > **Run workflow**.
   - `nom_machine` : nom affiché dans Tailscale (défaut `gh-runner`).
   - `duree_minutes` : durée de chaque session (défaut 335, maximum 340).
   - `relais` : `true` pour enchaîner automatiquement les sessions, `false` pour une seule session.
   - `max_relais` : nombre maximum de sessions enchaînées (défaut 4, soit environ 24 h).
   - `logiciels` : logiciels installés à chaque session, noms [Chocolatey](https://community.chocolatey.org/packages) séparés par des virgules (défaut `googlechrome,notepadplusplus,7zip,vlc`). Laissez vide pour ne rien installer. Un nom invalide est ignoré sans couper la session.
   - `rang` et `precedent` : remplis automatiquement, **ne pas toucher**.
2. Ouvrez l'exécution : l'adresse apparaît dans le résumé (**Summary**) et dans le journal de l'étape *Maintenir la session*.
3. Sur votre PC, Tailscale doit être lancé et connecté au même Tailnet.
4. Ouvrez *Connexion Bureau à distance*, saisissez l'IP `100.x.x.x` (ou simplement le nom `gh-runner`, grâce à MagicDNS), puis votre utilisateur et mot de passe.
5. Pour arrêter : **Cancel workflow** dans GitHub (la machine est retirée du Tailnet automatiquement). Annuler la session en cours arrête aussi la chaîne : aucun relais n'est lancé.

## Le relais automatique

Une session GitHub ne peut pas dépasser 6 h. Pour continuer au-delà, le workflow enchaîne les sessions :

1. 10 minutes avant la fin, un message s'affiche sur le bureau pour prévenir.
2. À la fin, le dossier `C:\Persist` est sauvegardé (artefact GitHub, conservé 1 jour).
3. Une nouvelle session est lancée automatiquement ; elle démarre dès que l'ancienne est terminée, restaure `C:\Persist` et reprend le même nom (`gh-runner`).
4. **Reconnectez-vous après 3 à 6 minutes** : la session RDP en cours est coupée pendant le changement.

**Ce qui est conservé** : uniquement les fichiers placés dans `C:\Persist` (raccourci sur le bureau).
**Ce qui est perdu** : logiciels installés, fenêtres ouvertes, fichiers hors de `C:\Persist`.

**Pré-requis** : dans le dépôt, **Settings > Actions > General > Workflow permissions** doit être sur **Read and write permissions**.

## Le gardien et les alertes (optionnel)

**Gardien** (`.github/workflows/gardien.yml`) : toutes les 30 minutes, il vérifie qu'une session tourne. S'il n'y en a aucune (coupure imprévue, fin de chaîne), il en relance une.

- **Interrupteur** : il ne fait rien tant que la variable `GARDIEN_ACTIF` n'est pas `true` (Settings > Secrets and variables > Actions > onglet **Variables**).
- **Pour s'arrêter pour de bon** : mettez d'abord `GARDIEN_ACTIF` sur `false`, puis annulez la session. Sinon le gardien relance.
- **Plafond** : par défaut 4 sessions par 24 h (variable `GARDIEN_MAX_PAR_JOUR`), pour protéger votre quota.

**Alertes sur le téléphone** (ntfy.sh, gratuit) :
1. Installez l'application **ntfy** (Android ou iPhone).
2. Inventez un nom de sujet long et impossible à deviner, par exemple `rdp-kipre-x7k29fqz` (le nom sert de mot de passe).
3. Dans l'application, abonnez-vous à ce sujet.
4. Dans le dépôt, ajoutez un secret `NTFY_TOPIC` contenant exactement ce nom.

Vous recevez : session prête (avec l'adresse), session terminée, annulée ou en échec, et relance par le gardien. Sans le secret `NTFY_TOPIC`, rien n'est envoyé et tout fonctionne quand même.

## Limites à connaître

- **6 heures maximum par session** : limite de GitHub. Le relais enchaîne des sessions, mais chaque changement coupe la connexion 3 à 6 minutes.
- Le runner est **éphémère** : tout ce qui est installé ou enregistré hors de `C:\Persist` disparaît à chaque relais.
- **Dépôt privé obligatoire** avec le relais : la sauvegarde de `C:\Persist` passe par GitHub, donc un dépôt public l'exposerait.
- Les minutes Windows consomment votre quota Actions plus vite que Linux (multiplicateur x2 sur les dépôts privés). Avec un quota limité, la chaîne s'arrête quand il est épuisé. Surveillez **Settings > Billing**.
- **Conditions d'utilisation** : GitHub Actions est prévu pour des tâches de développement. Enchaîner des sessions de bureau à distance peut entraîner la suspension du compte. Utilisez un compte dont vous pouvez vous passer.

## Dépannage

| Symptôme | Piste |
|---|---|
| « Secret ... manquant » | Vérifiez le nom exact des 3 secrets. |
| Connexion RDP impossible | Tailscale est-il connecté sur votre PC ? Utilisez bien l'IP `100.x.x.x` affichée. |
| Identifiants refusés | Mot de passe trop simple ou mal copié ; relancez après avoir corrigé le secret. |
| La machine n'apparaît pas dans Tailscale | Clé expirée ou révoquée : régénérez-la. |
| Session coupée au bout de ~6 h | Normal : c'est le relais. Reconnectez-vous après 3 à 6 minutes. Si aucune nouvelle session n'apparaît, vérifiez que `relais` est `true` et que le quota Actions n'est pas épuisé. |
| Le relais ne se lance pas | Vérifiez **Settings > Actions > General > Workflow permissions** : choisissez *Read and write permissions*. |
| La machine s'appelle `gh-runner-1` après un relais | L'ancienne machine n'avait pas encore quitté le Tailnet. Utilisez l'IP `100.x.x.x` du nouveau résumé, ou supprimez l'ancienne machine dans la console Tailscale. |
| `C:\Persist` est vide après un relais | La sauvegarde dure 1 jour et ne couvre que ce dossier. Vérifiez que vos fichiers y étaient avant la fin de session. |
