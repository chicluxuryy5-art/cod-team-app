# Reprise de Supabase — COD Team v15

**Constat au 14 novembre 2026.** L'URL du projet `https://sfsnyppydvhvnjiqjgjt.supabase.co` ne résout plus (NXDOMAIN). L'application en ligne s'ouvre, mais aucune connexion ni donnée n'est disponible. La cause la plus probable est un projet **en pause**, pas une suppression : Supabase endort les projets gratuits inactifs et permet de les réactiver depuis le dashboard.

Aucun jeton d'accès (`SUPABASE_ACCESS_TOKEN`), CLI `supabase`, `psql` ou Docker n'est disponible sur le poste. Toutes les opérations ci-dessous se font donc **manuellement dans le dashboard Supabase**.

> Ce guide ne contient aucune donnée réelle et aucune clé secrète. La clé `anon` présente dans `index.html` est publique par conception.

---

## 1. Remettre le projet en service

### Option A — Le projet apparaît encore dans le dashboard (à essayer en premier)

1. Ouvrir <https://supabase.com/dashboard> et se connecter avec le compte propriétaire.
2. Sélectionner l'organisation, puis le projet **COD Team** (`sfsnyppydvhvnjiqjgjt`).
3. Si le projet affiche **Paused** : cliquer sur **Resume project** et attendre 1 à 3 minutes.
4. Vérifier ensuite **Project Settings → API** :
   - `Project URL` doit être `https://sfsnyppydvhvnjiqjgjt.supabase.co` ;
   - noter la clé `anon` / `publishable` affichée.
5. Ouvrir l'application et tenter une connexion. Si les données réapparaissent, les migrations v5 → v12 étaient déjà appliquées : passer directement au § 4 (vérifications).

### Option B — Le projet n'apparaît plus (supprimé ou irrécupérable)

1. **New project** : nom `cod-team`, mot de passe de base de données fort et conservé, région la plus proche de l'Algérie (`eu-west-3` Paris ou `eu-central-1` Francfort).
2. Attendre la fin du provisionnement (2 à 5 minutes).
3. **Project Settings → API** : copier `Project URL` et la clé `anon` / `publishable`.
4. **SQL Editor → New query** : coller l'intégralité de `bootstrap_full.sql`, puis **Run**.
   - Le script est idempotent : il peut être relancé sans perte de données.
   - Il se termine par `Bootstrap complet OK` avec `tables = 28`, `wilayas = 58`, `settings = 10`.
5. Passer au § 2 (clés dans le front), puis au § 3 (comptes).

> Si le projet a été supprimé, les anciennes données (produits, commandes, conversations) sont perdues. Les données de démonstration de l'ancien projet étaient fictives ; aucune donnée réelle n'est stockée dans cette archive.

---

## 2. Mettre à jour les clés dans l'application

Les deux constantes se trouvent lignes 183–184 de `index.html` :

```js
var SUPABASE_URL='https://sfsnyppydvhvnjiqjgjt.supabase.co';
var SUPABASE_ANON_KEY='eyJ...';
```

À remplacer par les valeurs du § 1, puis :

1. Committer et pousser `index.html` sur la branche `main` du dépôt `mrchiealgerie-hub/cod-team-app`.
2. Attendre 1 à 2 minutes (publication GitHub Pages).
3. Forcer le rechargement : **Ctrl + Shift + R**.
4. Contrôler que le fichier servi est bien le nouveau (taille et hash, cf. § 5).

> Le CDN chargé est `@supabase/supabase-js@2` (non épinglé). Les clés `publishable` récentes (`sb_publishable_...`) sont acceptées par les versions v2 récentes ; si la connexion échoue avec une clé publishable, vérifier la console du navigateur avant de conclure.

---

## 3. Créer le premier admin et les comptes de l'équipe

### 3.1 Confirmation d'e-mail

Par défaut, un projet Supabase neuf exige la confirmation d'e-mail. Deux choix :

- **Le plus simple pour une équipe interne** : **Authentication → Sign In / Providers → Email** → désactiver **Confirm email**. Les inscriptions sont alors utilisables immédiatement.
- **Ou** garder la confirmation et confirmer chaque compte dans **Authentication → Users** (l'icône de validation), ou créer les comptes directement avec **Add user** (compte déjà confirmé).

### 3.2 Premier administrateur

1. Ouvrir l'application et créer un compte via l'écran d'inscription.
2. Le trigger d'inscription force le rôle **`confirmatrice`**, quelles que soient les métadonnées envoyées par le formulaire. C'est le comportement attendu.
3. Dans **SQL Editor**, promouvoir ce compte :

```sql
update public.profiles
   set role = 'admin'
 where id = (select id from auth.users where email = 'VOTRE_EMAIL');
```

Le SQL Editor s'exécute sans `auth.uid()` : le trigger anti-escalade ne bloque donc pas cette promotion. C'est le seul chemin prévu pour créer le premier admin.

### 3.3 Autres membres

Créer chaque compte (inscription ou **Add user**), puis renseigner rôle, coût horaire et numéro WhatsApp :

```sql
update public.profiles p
   set role            = 'media_buyer',      -- admin | media_buyer | video_editor | confirmatrice | production | sourcing
       hourly_cost_dzd = 400,                -- coût horaire réel en DZD
       whatsapp_phone  = '+213XXXXXXXXX'
 where p.id = (select id from auth.users u where u.email = 'prenom@exemple.dz');
```

Rôles disponibles : `admin`, `media_buyer`, `video_editor`, `confirmatrice`, `production`, `sourcing`.

> Une inscription libre ne donne jamais plus que `confirmatrice`. Seul un admin peut promouvoir, via l'interface Équipe ou par SQL.

### 3.4 Fermer les inscriptions (optionnel)

Une fois l'équipe créée : **Authentication → Sign In / Providers** → désactiver les inscriptions anonymes, ou restreindre les domaines autorisés. Le rôle minimal reste `confirmatrice` même si les inscriptions restent ouvertes.

---

## 4. Vérifications après exécution

À lancer dans **SQL Editor** :

```sql
-- 1) Tables, politiques, triggers
select
  (select count(*) from information_schema.tables
    where table_schema='public' and table_type='BASE TABLE') as tables,      -- attendu 28
  (select count(*) from pg_policies where schemaname='public') as policies,  -- attendu 84
  (select count(*) from information_schema.triggers
    where trigger_schema='public') as triggers;                              -- attendu 16

-- 2) Référentiels
select (select count(*) from public.wilayas)  as wilayas,   -- attendu 58
       (select count(*) from public.settings) as settings;  -- attendu 10

-- 3) Tables publiées en Realtime
select count(*) as realtime_tables
  from pg_publication_tables
 where pubname='supabase_realtime' and schemaname='public';

-- 4) Forçage du rôle à l'inscription (le trigger doit exister)
select tgname from pg_trigger
 where tgrelid='auth.users'::regclass and not tgisinternal;
```

Résultats mesurés lors de la validation locale (PGlite + stubs Supabase, 14/11/2026) : **28 tables, 84 politiques, 16 triggers, 58 wilayas, 10 réglages**. Les tests de comportement couvrent l'inscription forcée, l'anti-escalade de rôle, le score produit pondéré, les commandes COD, l'isolation des notifications et des tâches, le stock, le temps, WhatsApp et la RPC de rémunération. Une seconde exécution complète du bootstrap ne produit ni erreur ni doublon.

### Realtime

L'application écoute 20 tables : `products`, `tasks`, `workshop_offers`, `product_tests`, `cod_kpis`, `orders`, `customer_blacklist`, `creative_tests`, `wa_campaigns`, `wa_campaign_recipients`, `wa_agent_flows`, `wa_relay_status`, `wa_conversations`, `wa_messages`, `time_entries`, `timesheets`, `sla_definitions`, `comp_runs`, `comments`, `notifications`.

Le bootstrap les ajoute toutes à la publication `supabase_realtime`. Si une table manque :

```sql
alter publication supabase_realtime add table public.NOM_TABLE;
```

### Storage

Le bootstrap crée le bucket public `media` et ses politiques. **La v15 du front ne contient aucun code d'upload** : le bucket est prêt mais inutilisé. Les vidéos et créas sont référencées par URL externes.

### RLS

Les politiques sont créées par le bootstrap. Un contrôle rapide :

```sql
select tablename, policyname, cmd
  from pg_policies
 where schemaname='public'
 order by tablename, policyname;
```

---

## 5. Vérifier le déploiement du front

Depuis un poste avec accès au dépôt :

```powershell
# Taille du fichier local
(Get-Item .\index.html).Length

# Hash local
Get-FileHash .\index.html -Algorithm SHA256

# Taille servie par GitHub Pages
(Invoke-WebRequest https://mrchiealgerie-hub.github.io/cod-team-app/index.html -UseBasicParsing).RawContentLength
```

Les deux tailles doivent correspondre. Si le site sert encore l'ancien fichier :

1. vérifier que le commit est bien sur `main` et que le workflow Pages est terminé ;
2. vider le cache du navigateur ou ouvrir une fenêtre privée ;
3. attendre 1 à 2 minutes.

---

## 6. Sécurité — à faire avant usage réel

| Point | Action |
|---|---|
| Compte de test `test.admin@codteam.dz` | Changer le mot de passe ou supprimer le compte |
| Clé `service_role` | Ne jamais la mettre dans `index.html`, le dépôt ou un navigateur |
| `relay_token` / `relay_url` | Stockés dans `settings`, lisibles par tout utilisateur authentifié : utiliser un jeton dédié et le renouveler si le relais est abandonné |
| Inscriptions | Fermer après création de l'équipe, ou garder le rôle minimal `confirmatrice` |
| Données de démo | Nettoyer avant production si les tableaux de bord doivent être vierges |
| Mot de passe base de données | Conservé hors dépôt (gestionnaire de mots de passe) |
| Relais Baileys | Non officiel : numéro dédié, risque de suspension |

---

## 7. Ce qui reste indisponible tant que Supabase est hors ligne

Même avec le front à jour, restent inactifs : connexion, données partagées, Realtime, stock, wilayas, tests A/B, WhatsApp CRM, suivi du temps, feuilles de temps et rémunération variable. L'interface s'affiche, mais les appels échouent.

Après exécution de `bootstrap_full.sql` et mise à jour des clés, l'ensemble de ces modules est opérationnel, sous réserve de renseigner les données réelles (produits, commandes, profils, tarifs).

---

## 8. Checklist de reprise

- [ ] Projet réactivé (ou recréé) et URL Supabase accessible
- [ ] `bootstrap_full.sql` exécuté : 28 tables, 58 wilayas, 10 réglages
- [ ] `SUPABASE_URL` et `SUPABASE_ANON_KEY` mis à jour dans `index.html`
- [ ] `index.html` poussé sur `main`, GitHub Pages à jour, hash vérifié
- [ ] Confirmation d'e-mail désactivée ou comptes confirmés
- [ ] Premier compte promu `admin`
- [ ] Comptes de l'équipe créés avec rôle, coût horaire et WhatsApp
- [ ] Mot de passe du compte de test changé ou compte supprimé
- [ ] Realtime vérifié sur les 20 tables
- [ ] Inscriptions fermées (si équipe stabilisée)
- [ ] Relais WhatsApp configuré uniquement si nécessaire (sinon mode manuel `wa.me`)

---

*Guide de reprise — COD Team v15. Les valeurs « attendues » proviennent du banc de test local ; elles ne remplacent pas une vérification sur le projet réel.*
