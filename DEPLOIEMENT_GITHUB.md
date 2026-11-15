# Déploiement GitHub Pages — état et procédure

**État au 14 novembre 2026, 19 h 48 (heure locale).**

| Élément | Valeur |
|---|---|
| Fichier local | `index.html` — **392 463 octets** |
| SHA-256 local | `E4C1E2C79CC101F2056BC3A28DDAE1ABAC4B82FFC40DF1D8DC0177E5B99347AF` |
| Fichier servi | `https://mrchiealgerie-hub.github.io/cod-team-app/index.html` — **392 184 octets** (ancienne version) |
| Commit local prêt | dernier commit de la copie de travail (`HEAD`) |
| Push | **Refusé (HTTP 403)** : la session Git est authentifiée comme `chih36`, sans droit d'écriture sur `mrchiealgerie-hub/cod-team-app` |

Le dépôt local prêt à pousser se trouve dans `C:\Users\pc\AppData\Local\Temp\opencode\cod-team\repo-git` (clone du dépôt public). Le commit contient l'application patchée, les scripts SQL, le README, le CHANGELOG et le guide de reprise.

`gh` (GitHub CLI) n'est pas installé sur le poste. Aucun jeton `GITHUB_TOKEN` / `GH_TOKEN` n'est présent dans l'environnement.

---

## Option A — Donner l'accès écriture au compte `chih36` (le plus rapide)

1. Ouvrir <https://github.com/mrchiealgerie-hub/cod-team-app/settings/access>.
2. **Add people** → rechercher `chih36` → rôle **Write** → inviter.
3. Accepter l'invitation depuis le compte `chih36`.
4. Relancer :

```powershell
cd "C:\Users\pc\AppData\Local\Temp\opencode\cod-team\repo-git"
git push origin main
```

## Option B — Se reconnecter avec le compte propriétaire

Sur un poste où le navigateur peut s'ouvrir :

```powershell
git credential-manager logout https://github.com
cd "C:\Users\pc\AppData\Local\Temp\opencode\cod-team\repo-git"
git push origin main
```

Git Credential Manager ouvre une fenêtre de connexion GitHub : utiliser le compte **mrchiealgerie-hub**. Si un mot de passe est demandé, utiliser un **Personal Access Token** avec le scope `repo` (les mots de passe classiques ne fonctionnent plus pour Git).

## Option C — Envoi manuel par l'interface GitHub

1. Ouvrir <https://github.com/mrchiealgerie-hub/cod-team-app/upload/main>.
2. Glisser-déposer, depuis `cod-team-FINAL-v15/` :
   - `index.html` (obligatoire) ;
   - `.nojekyll` (si absent) ;
   - `bootstrap_base.sql`, `bootstrap_full.sql`, `migrations_combined.sql`, `v5_migration.sql` à `v12_migration.sql` ;
   - `README.md`, `CHANGELOG.md`, `RECOVERY_SUPABASE.md`, `DEPLOIEMENT_GITHUB.md`.
3. Valider le commit sur `main`.
4. Attendre 1 à 2 minutes, puis vérifier le déploiement.

---

## Vérification après déploiement

```powershell
$r = Invoke-WebRequest "https://mrchiealgerie-hub.github.io/cod-team-app/index.html" -UseBasicParsing
"HTTP $($r.StatusCode) - $($r.RawContentLength) octets"
```

Attendu : **HTTP 200** et **392 396 octets**. Si la taille reste 392 184, le déploiement n'a pas pris : vérifier le dernier commit sur `main`, l'onglet **Actions** (workflow Pages) et le cache du navigateur.

Forcer ensuite le rechargement avec **Ctrl + Shift + R**.

---

## Ce que change cette version du front

- Inscription : le sélecteur de rôle est retiré ; le formulaire envoie explicitement `role: 'confirmatrice'`.
- Message d'erreur réseau clarifié sur les 13 points d'entrée Supabase : « Serveur Supabase injoignable. Le projet est peut-être en pause - contactez l'administrateur. »
- Le reste de l'interface v15 est inchangé.

> Le déploiement du front ne rétablit pas la connexion : le projet Supabase doit d'abord être réactivé ou recréé, puis `bootstrap_full.sql` exécuté (voir `RECOVERY_SUPABASE.md`).
