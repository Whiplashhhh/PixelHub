# TP 1 — Mise en place de l'environnement

**R5.A.10 — Nouveaux paradigmes de bases de données**
Durée : 3 h, installation du poste comprise · À rendre : rien aujourd'hui, mais **votre environnement doit fonctionner avant de partir**, et **toutes les étapes sont obligatoires**

---

## Objectif

À la fin de ce TP, vous aurez :

- **quatre moteurs de bases de données** qui tournent sur votre machine, sans en avoir installé aucun ;
- l'application **PixelHub** qui démarre et répond ;
- une première brique fonctionnelle : la gestion des joueurs dans **PostgreSQL**, en lecture et en création ;
- **manipulé chacun des quatre moteurs à la main** : une transaction PostgreSQL qui protège la monnaie virtuelle, des clés Redis qui expirent, des documents MongoDB qui n'ont pas tous les mêmes champs, un petit graphe Neo4j.

C'est le point de départ. Pendant les cinq séances suivantes, vous ajouterez un moteur de plus à *cette même application*. Les manipulations des étapes 5 et 6 sont un premier contact avec ce que les séances 2 à 4 approfondiront.

> **Important** : ne partez pas de la salle avec un environnement cassé. Les 15 heures suivantes reposent sur celui d'aujourd'hui. Si quelque chose ne marche pas, appelez — c'est le but de cette séance.

### Déroulé indicatif

| Étape | Contenu | Durée |
|---|---|---|
| 0 | Installer et vérifier le poste | 25 min |
| 1 | Lire le `docker-compose.yml` avant de lancer | 15 min |
| 2 | Démarrer et vérifier les quatre moteurs | 20 min |
| 3 | L'application PixelHub + PostgreSQL | 35 min |
| — | Pause | 10 min |
| 4 | Créer un joueur : `POST /players` | 15 min |
| 5 | ⭐ PostgreSQL : la transaction qui protège la monnaie | 25 min |
| 6 | Premiers pas dans Redis, MongoDB et Neo4j | 25 min |
| 7 | Les données survivent-elles ? Arrêter proprement | 10 min |

Répondez aux questions **Q1 à Q17** par écrit, au fil de l'eau, dans un fichier `notes.md`. Elles seront reprises à l'oral.

---

## Étape 0 — Installer et vérifier le poste (25 min)

Si ce n'est pas déjà fait, installez Docker et le SDK .NET en suivant l'**annexe d'installation** (`TP1_annexe_installation_etudiants.md`), section correspondant à votre système.

Vérifiez ensuite que vous avez tout. Dans un terminal :

```bash
docker --version
docker compose version
dotnet --version
```

Les trois commandes doivent répondre avec un numéro de version. Si l'une échoue, reportez-vous à l'annexe d'installation ou appelez l'enseignant.

Récupérez ensuite le **dépôt de départ** (lien fourni par l'enseignant), et placez-vous dans le dossier :

```bash
cd pixelhub
```

---

## Étape 1 — Lire avant de lancer (15 min)

Ouvrez le fichier `docker-compose.yml`. **Ne lancez rien encore.**

Ce fichier décrit quatre services, un par moteur de base de données. Lisez-le, puis répondez à ces questions par écrit (sur papier ou dans votre fichier `notes.md`) :

**Q1.** Le fichier déclare quatre services alors que le TP d'aujourd'hui n'en utilise qu'un. Pourquoi, à votre avis ?

**Q2.** Trois services ont une section `volumes`, un seul n'en a pas. Lequel, et qu'est-ce que ça implique pour ses données ?

**Q3.** Que signifie la ligne `"15432:5432"` ? Les deux nombres ne désignent pas la même chose.

**Q4.** Les mots de passe sont écrits en clair dans le fichier. Est-ce acceptable ici ? Le serait-ce sur un serveur de production ?

> Ces questions seront reprises à l'oral. Prenez-les au sérieux : la question 3 en particulier vous évitera une demi-heure de blocage plus tard.

---

## Étape 2 — Démarrer les quatre moteurs (20 min)

### 2.1 Lancer

```bash
docker compose up -d
```

`-d` signifie *detached* : les conteneurs tournent en arrière-plan et vous récupérez votre terminal.

**Le premier lancement télécharge plusieurs gigaoctets d'images.** C'est normal que ce soit long. Si vous voyez « Pulling » pendant plusieurs minutes, laissez faire.

### 2.2 Vérifier que les quatre tournent

```bash
docker compose ps
```

Vous devez voir quatre services à l'état `running`. Si l'un est en `exited` ou `restarting`, notez son nom et consultez la section **Problèmes fréquents** à la fin de ce sujet.

### 2.3 Vérifier chaque moteur individuellement

**Ne sautez pas cette étape.** « Le conteneur tourne » ne veut pas dire « le moteur répond ».

```bash
# PostgreSQL → doit répondre "accepting connections"
docker compose exec postgres pg_isready -U pixelhub

# Redis → doit répondre PONG
docker compose exec redis redis-cli ping

# MongoDB → doit répondre { ok: 1 }
docker compose exec mongo mongosh -u pixelhub -p pixelhub_dev \
  --authenticationDatabase admin --eval "db.runCommand({ping:1})"
```

**Neo4j** : ouvrez `http://localhost:17474` dans votre navigateur. Dans le formulaire de connexion :

- **Connect URL** : `neo4j://localhost:17687` — le Browser propose `7687`, **corrigez-le en `17687`** ;
- utilisateur `neo4j`, mot de passe `pixelhub_dev`.

> **Pourquoi deux ports ?** Neo4j en ouvre deux. Le **17474** (HTTP) sert seulement à charger la page web du Browser. Le **17687** (protocole **Bolt**) transporte vos requêtes et leurs résultats : c'est par lui que la page se connecte ensuite à la base, et c'est lui que votre code C# utilisera en séance 4. Le Browser vous propose `7687` parce que c'est le port *à l'intérieur* du conteneur, que Neo4j croit être le sien : relisez votre réponse à la Q3.

> Neo4j met **20 à 40 secondes** à démarrer, bien plus que les autres. Si la page ne répond pas tout de suite, attendez et réessayez. Pour suivre son démarrage : `docker compose logs neo4j`.

### 2.4 À noter dans vos notes

**Q5.** Les commandes ci-dessus utilisent `docker compose exec`. Qu'est-ce que ça fait exactement ? Où s'exécute la commande `redis-cli` ?

---

## Étape 3 — L'application PixelHub (35 min)

On part de ce que vous maîtrisez : une API et une base relationnelle. Rien de nouveau aujourd'hui — c'est voulu, c'est la base sur laquelle on va greffer le reste.

### 3.1 Installer les paquets

```bash
cd src/PixelHub.Api
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### 3.2 Le modèle

Créez `Models/Player.cs` :

```csharp
namespace PixelHub.Api.Models;

public class Player
{
    public int Id { get; set; }
    public string Pseudo { get; set; } = "";
    public int Coins { get; set; }          // monnaie virtuelle
}
```

### 3.3 Le contexte

Créez `Data/PixelHubContext.cs` :

```csharp
using Microsoft.EntityFrameworkCore;
using PixelHub.Api.Models;

namespace PixelHub.Api.Data;

public class PixelHubContext : DbContext
{
    public PixelHubContext(DbContextOptions<PixelHubContext> options)
        : base(options) { }

    public DbSet<Player> Players => Set<Player>();
}
```

### 3.4 La chaîne de connexion

Dans `appsettings.json`, ajoutez :

```json
{
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Port=15432;Database=pixelhub;Username=pixelhub;Password=pixelhub_dev"
  }
}
```

**Q6.** La base de données tourne dans un conteneur Docker, et la chaîne de connexion dit `Host=localhost`. Pourquoi est-ce que ça fonctionne ? (Indice : relisez votre réponse à la Q3.)

### 3.5 Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using PixelHub.Api.Data;
using PixelHub.Api.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<PixelHubContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")));

var app = builder.Build();

// Crée la base et les tables au démarrage.
// Suffisant en TP ; dans un vrai projet, on utilise les migrations EF Core.
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<PixelHubContext>();
    db.Database.EnsureCreated();

    if (!db.Players.Any())
    {
        db.Players.AddRange(
            new Player { Pseudo = "Nova",  Coins = 1200 },
            new Player { Pseudo = "Krayz", Coins = 350  },
            new Player { Pseudo = "Ombre", Coins = 90   }
        );
        db.SaveChanges();
    }
}

app.MapGet("/", () => "PixelHub API — séance 1 OK");

app.MapGet("/players", async (PixelHubContext db) =>
    await db.Players.ToListAsync());

app.Run();
```

### 3.6 Lancer et tester

```bash
dotnet run
```

Notez le **port** affiché au démarrage (une ligne du type `Now listening on: http://localhost:5199`). Il est propre à votre projet.

Pour tester, ouvrez le fichier `PixelHub.Api.http` fourni dans le dépôt, ajustez la ligne `@host` avec votre port, puis cliquez sur **Send request** au-dessus de la requête `GET /players`.

> **Attention** : il n'y a **pas de page Swagger**. Depuis .NET 9, l'interface Swagger n'est plus incluse dans les modèles de projet. On utilise les fichiers `.http`, qui sont intégrés à Visual Studio.

**Résultat attendu** : trois joueurs en JSON.

```json
[
  { "id": 1, "pseudo": "Nova",  "coins": 1200 },
  { "id": 2, "pseudo": "Krayz", "coins": 350  },
  { "id": 3, "pseudo": "Ombre", "coins": 90   }
]
```

### 3.7 Vérifier côté base

Les données sont-elles vraiment dans PostgreSQL, ou seulement en mémoire ? Vérifiez :

```bash
docker compose exec postgres psql -U pixelhub -d pixelhub -c "SELECT * FROM \"Players\";"
```

**Q7.** Arrêtez l'application (Ctrl+C), relancez-la. Les trois joueurs sont-ils dupliqués ? Pourquoi ?

---

## Étape 4 — Créer un joueur : `POST /players` (15 min)

Ajoutez dans `Program.cs`, **avant** `app.Run()`, un endpoint `POST /players` qui reçoit un joueur en JSON, l'enregistre dans PostgreSQL et répond **201 Created**.

Quelques pistes, sans tout donner :

- c'est un `app.MapPost(...)`, construit comme le `MapGet` juste au-dessus ;
- le joueur reçu se déclare comme un paramètre de type `Player`, à côté du `PixelHubContext` : ASP.NET Core le lit dans le corps de la requête ;
- on ajoute avec `db.Players.Add(...)`, on enregistre avec `await db.SaveChangesAsync()` ;
- on répond avec `Results.Created($"/players/{player.Id}", player)`.

**Arrêtez l'application (Ctrl+C) et relancez-la** : `dotnet run` compile votre code *au démarrage*, il ne voit pas les modifications faites ensuite. Pour ne plus y penser, lancez plutôt `dotnet watch run`, qui redémarre l'API à chaque enregistrement.

Testez avec le bloc `POST {{host}}/players` déjà présent dans le fichier `.http` :

```json
{ "pseudo": "Pixel", "coins": 500 }
```

**Résultat attendu** : `201 Created`, un en-tête `Location: /players/4`, et le joueur créé avec `"id": 4`. Un `GET /players` renvoie maintenant quatre joueurs.

> **Envoyez la requête une seule fois.** Rien n'empêche aujourd'hui deux joueurs d'avoir le même pseudo : un second envoi créerait un deuxième *Pixel*.

**Q8.** Dans votre code, que vaut `player.Id` *avant* l'appel à `SaveChangesAsync()` ? Et après ? Qui a choisi la valeur 4 : votre code, EF Core ou PostgreSQL ?

---

## Étape 5 — ⭐ PostgreSQL : la transaction qui protège la monnaie (25 min)

Les `coins` sont de la monnaie virtuelle : les joueurs la paient avec de l'argent réel. Une pièce perdue ou créée par erreur, c'est un joueur lésé ou de l'argent offert. Cette étape vous montre ce que PostgreSQL garantit à cette donnée, et c'est la raison pour laquelle **la monnaie restera dans PostgreSQL jusqu'à la fin du module**, même quand les trois autres moteurs seront branchés.

Ouvrez une session `psql` **interactive** (l'invite devient `pixelhub=#`) :

```bash
docker compose exec postgres psql -U pixelhub -d pixelhub
```

> EF Core a créé la table et les colonnes avec des majuscules (`"Players"`, `"Pseudo"`, `"Coins"`) : en SQL, il faut alors les écrire **entre guillemets doubles**. Sans guillemets, PostgreSQL cherche `players` en minuscules et répond que la table n'existe pas.

### 5.1 Atomicité : tout ou rien

Nova donne 100 pièces à Krayz. C'est **deux** `UPDATE` : un débit, un crédit. Tapez ligne par ligne :

```sql
BEGIN;
UPDATE "Players" SET "Coins" = "Coins" - 100 WHERE "Pseudo" = 'Nova';
UPDATE "Players" SET "Coins" = "Coins" + 100 WHERE "Pseudo" = 'Krayz';
SELECT "Pseudo", "Coins" FROM "Players" ORDER BY "Id";
```

Nova a 1100, Krayz 450. Changement d'avis :

```sql
ROLLBACK;
SELECT "Pseudo", "Coins" FROM "Players" ORDER BY "Id";
```

**Q9.** Que valent les soldes après le `ROLLBACK` ? Imaginez maintenant que le serveur s'éteigne entre les deux `UPDATE`, *sans* transaction : dans quel état serait la base, et qui y perdrait ?

### 5.2 Isolation : les autres ne voient pas le travail en cours

Il vous faut **deux terminaux** côte à côte, chacun avec sa propre session `psql` (la commande ci-dessus, lancée deux fois). Appelons-les **A** et **B**.

Dans **A**, commencez un débit **sans le terminer** :

```sql
BEGIN;
UPDATE "Players" SET "Coins" = "Coins" - 100 WHERE "Pseudo" = 'Nova';
SELECT "Coins" FROM "Players" WHERE "Pseudo" = 'Nova';   -- 1100
```

Dans **B** :

```sql
SELECT "Coins" FROM "Players" WHERE "Pseudo" = 'Nova';
```

Envoyez aussi `GET /players` depuis le fichier `.http` : que voit l'API pour Nova ?

Toujours dans **B**, essayez de modifier le même joueur :

```sql
BEGIN;
UPDATE "Players" SET "Coins" = "Coins" + 1 WHERE "Pseudo" = 'Nova';
```

**B ne rend pas la main.** C'est normal : attendez quelques secondes, puis tapez dans **A** :

```sql
ROLLBACK;
```

B se débloque aussitôt et affiche `UPDATE 1`. Terminez-le aussi, pour ne rien laisser traîner :

```sql
ROLLBACK;
```

**Q10.** Pendant la transaction de A, quel solde de Nova voyaient B et l'API ? Pourquoi l'`UPDATE` de B a-t-il dû attendre ? Imaginez que A et B soient deux achats lancés au même instant par Nova sur deux téléphones : que pourrait-il se passer si la base ne bloquait pas B ?

### 5.3 Cohérence : la base refuse ce qui viole une règle

Ombre n'a que 90 pièces. On ajoute une règle « jamais de solde négatif », **à l'intérieur d'une transaction** pour pouvoir tout annuler ensuite, puis on tente un achat à 100 pièces :

```sql
BEGIN;
ALTER TABLE "Players" ADD CONSTRAINT coins_positifs CHECK ("Coins" >= 0);
UPDATE "Players" SET "Coins" = "Coins" - 100 WHERE "Pseudo" = 'Ombre';
```

Lisez le message d'erreur. Essayez ensuite n'importe quelle requête, par exemple :

```sql
SELECT "Coins" FROM "Players" WHERE "Pseudo" = 'Ombre';
```

Puis :

```sql
ROLLBACK;
\d "Players"
```

`\d` décrit la table : regardez si la contrainte `coins_positifs` y figure encore. Quittez `psql` avec `\q`.

**Q11.** Qui a refusé l'achat d'Ombre : votre code C# ou la base ? Pourquoi le `SELECT` suivant a-t-il échoué lui aussi ? Après le `ROLLBACK`, la contrainte existe-t-elle toujours, et qu'est-ce que ça vous apprend sur PostgreSQL ?

> **Vérifiez avant de continuer** : `GET /players` doit renvoyer Nova 1200, Krayz 350, Ombre 90, Pixel 500. Si un solde a changé, un `COMMIT` a été tapé à la place d'un `ROLLBACK` : appelez l'enseignant.

---

## Étape 6 — Premiers pas dans Redis, MongoDB et Neo4j (25 min)

Trois moteurs, trois façons de ranger la donnée. On ne fait que les **toucher** aujourd'hui : chacun aura sa séance. Tout ce que vous créez ici est **supprimé à la fin de chaque sous-étape**, pour démarrer les séances 2 à 4 avec des bases propres.

### 6.1 Redis : des clés, des valeurs, et le temps qui passe (8 min)

```bash
docker compose exec redis redis-cli
```

L'invite devient `127.0.0.1:6379>`. Une clé, une valeur :

```
SET essai "bonjour"
GET essai
```

Une clé qui **s'autodétruit** au bout de 10 secondes. Tapez `TTL temporaire` plusieurs fois de suite, jusqu'à dépasser les 10 secondes, puis `GET temporaire` :

```
SET temporaire "vite" EX 10
TTL temporaire
TTL temporaire
GET temporaire
```

Un compteur :

```
INCR visites
INCR visites
INCR visites
GET visites
```

Nettoyez, puis quittez :

```
DEL essai visites
exit
```

**Q12.** Que renvoie `TTL temporaire` au fil des secondes, puis une fois les 10 secondes écoulées ? Et `GET temporaire` ? Citez une donnée de PixelHub qui gagnerait à disparaître toute seule.

**Q13.** `INCR` lit, ajoute 1 et réécrit en **une seule** commande. Pourquoi est-ce plus sûr que de faire un `GET`, d'ajouter 1 dans le code C#, puis un `SET`, si 200 joueurs déclenchent le compteur au même moment ? (Pensez à ce que vous avez vu en 5.2.)

### 6.2 MongoDB : des documents qui n'ont pas tous la même forme (7 min)

```bash
docker compose exec mongo mongosh -u pixelhub -p pixelhub_dev --authenticationDatabase admin
```

Travaillez dans une base d'essai, **pas** dans la base `pixelhub` qui servira en séance 2 :

```javascript
use essais
db.jeux.insertOne({ titre: "Hades II", genre: "Roguelike", plateformes: ["PC", "Switch"] })
db.jeux.insertOne({ titre: "Stardew Valley", genre: "Simulation", multijoueur: { joueursMax: 8 } })
db.jeux.find()
db.jeux.find({ genre: "Roguelike" })
db.jeux.countDocuments()
```

Observez les deux documents affichés par `db.jeux.find()` : leurs champs, et le champ `_id` que vous n'avez pas écrit.

Nettoyez, puis quittez :

```javascript
db.dropDatabase()
exit
```

**Q14.** Les deux jeux sont dans la même collection, mais n'ont pas les mêmes champs (`plateformes` est une liste, `multijoueur` un objet imbriqué). Aurait-on pu ranger les deux dans une même table PostgreSQL ? À quel prix ? Rapprochez votre réponse de l'exercice du catalogue de jeux (section 2 du cours `CM1_etudiants_Panorama.md`).

### 6.3 Neo4j : un graphe, des relations qu'on suit (10 min)

Dans le Neo4j Browser (`http://localhost:17474`, connecté sur `neo4j://localhost:17687`), tapez dans la barre de requête en haut, puis **Ctrl+Entrée** (ou le bouton ▶) :

```cypher
CREATE (ana:Essai {pseudo: 'Ana'}),
       (bob:Essai {pseudo: 'Bob'}),
       (cleo:Essai {pseudo: 'Cleo'}),
       (ana)-[:AMI_DE]->(bob),
       (bob)-[:AMI_DE]->(cleo)
RETURN ana, bob, cleo
```

Le résultat s'affiche **en graphe** : trois ronds (des **nœuds**) reliés par des flèches (des **relations**). Déplacez-les à la souris, cliquez sur un nœud pour voir ses propriétés.

Qui est l'ami d'un ami d'Ana ? On décrit le **chemin** à suivre, avec des flèches :

```cypher
MATCH (:Essai {pseudo: 'Ana'})-[:AMI_DE]->()-[:AMI_DE]->(amiDAmi)
RETURN amiDAmi.pseudo
```

Supprimez maintenant les nœuds :

```cypher
MATCH (n:Essai) DELETE n
```

Lisez le message d'erreur, puis :

```cypher
MATCH (n:Essai) DETACH DELETE n
```

Vérifiez qu'il ne reste rien (le résultat doit être `0`) :

```cypher
MATCH (n:Essai) RETURN count(n)
```

**Q15.** La requête « ami d'un ami » se lit presque comme un dessin. Comment l'écririez-vous en SQL, avec une table `Amities(joueur_id, ami_id)` ? Combien de jointures faudrait-il pour « ami d'un ami d'un ami » ?

**Q16.** Pourquoi `DELETE` seul a-t-il été refusé ? Que fait `DETACH` de plus ?

---

## Étape 7 — Les données survivent-elles ? Arrêter proprement (10 min)

Revenez à votre réponse à la Q2, et vérifiez-la. Posez une clé dans Redis :

```bash
docker compose exec redis redis-cli SET survivant "toujours là ?"
```

**Premier test : `stop` puis `start`.**

```bash
docker compose stop
docker compose start
docker compose exec redis redis-cli GET survivant
```

**Second test : `down` puis `up`.**

```bash
docker compose down
docker compose up -d
docker compose exec redis redis-cli GET survivant
docker compose exec postgres psql -U pixelhub -d pixelhub -c "SELECT \"Pseudo\" FROM \"Players\";"
```

> Si `psql` répond que la connexion est refusée, PostgreSQL n'a pas fini de redémarrer : attendez quelques secondes et relancez la commande.

**Q17.** La clé `survivant` a-t-elle survécu au `stop` ? Au `down` ? Et les joueurs de PostgreSQL ? Expliquez la différence avec le mot **volume**.

Prenez maintenant la bonne habitude, à faire à **chaque** fin de séance :

```bash
docker compose stop
```

Et retenez bien la différence, parce que la confusion vous coûtera vos données en séance 3 ou 4 :

| Commande | Effet |
|---|---|
| `docker compose stop` | Éteint les conteneurs. **Données conservées.** À utiliser en fin de séance. |
| `docker compose down` | Supprime les conteneurs. Données conservées *si elles vivent dans un volume* — vous venez de voir ce que devient Redis, qui n'en a pas. |
| `docker compose down -v` | Supprime les conteneurs **et les volumes**. ⚠️ **Tout est perdu.** |

`down -v` est parfois la solution à un problème (voir plus bas), mais ne l'utilisez jamais « pour faire propre ».

---

## Avant de partir — checklist

Faites valider par l'enseignant :

- [ ] `docker compose ps` affiche les services attendus en `running`
- [ ] `docker compose exec redis redis-cli ping` répond `PONG`
- [ ] Le Neo4j Browser est **connecté** sur `neo4j://localhost:17687`
- [ ] L'application démarre sans erreur
- [ ] `/players` renvoie les **quatre** joueurs en JSON, avec leurs soldes d'origine (Nova 1200, Krayz 350, Ombre 90, Pixel 500)
- [ ] `POST /players` répond `201 Created`
- [ ] Les essais sont nettoyés : base `essais` supprimée dans MongoDB, `MATCH (n:Essai) RETURN count(n)` renvoie `0` dans Neo4j
- [ ] Les questions **Q1 à Q17** ont une réponse dans `notes.md`
- [ ] Je sais dire ce que `ROLLBACK` annule, et la différence entre `docker compose stop` et `down -v`
- [ ] **Mon travail est sauvegardé ailleurs qu'en local** (dépôt Git, ou archive personnelle)

Le dernier point est important : sur un poste de salle, un profil réinitialisé entre deux séances fait perdre tout le travail.

---

## Problèmes fréquents

### `port already in use` / un conteneur refuse de démarrer

Un service occupe déjà le port sur votre machine — typiquement un PostgreSQL installé en dur.

Pour identifier le coupable :
```bash
# Windows (PowerShell)
netstat -ano | findstr :15432
# macOS / Linux
lsof -i :15432
```

**Solution** : modifiez le port dans le fichier `.env` (par exemple `PG_PORT=15433`), **et n'oubliez pas de modifier `appsettings.json` en conséquence.** L'incohérence entre les deux fichiers est la première cause d'erreur de connexion.

### `permission denied ... docker.sock` (Linux)

Votre compte n'est pas dans le groupe `docker`.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Ne préfixez pas vos commandes par `sudo` pour contourner : ça crée des fichiers appartenant à `root` dans votre projet, et ça vous causera des erreurs incompréhensibles plus tard.

### Docker Desktop reste bloqué sur « Starting » (Windows)

```powershell
wsl --update
```
puis redémarrez. Si le problème persiste, la virtualisation est peut-être désactivée dans le BIOS — dans ce cas, signalez-le à l'enseignant, vous ne pouvez pas le résoudre seul sur un poste de l'IUT.

### `Failed to connect to 127.0.0.1:15432`

À vérifier dans cet ordre :

1. les conteneurs tournent-ils ? → `docker compose ps`
2. PostgreSQL est-il prêt ? → `docker compose exec postgres pg_isready -U pixelhub`
3. le port de `appsettings.json` correspond-il à celui du `.env` ?
4. le mot de passe est-il le même dans les deux fichiers ?
5. si vous avez fait un essai précédent avec un **autre mot de passe** : les variables du compose ne s'appliquent qu'à la *première* initialisation du volume. Il faut réinitialiser :
   ```bash
   docker compose down -v
   docker compose up -d
   ```

### Le Neo4j Browser s'affiche, mais la connexion échoue

**Symptôme** : la page `http://localhost:17474` se charge, mais le bouton *Connect* échoue (`ServiceUnavailable`, « Could not perform discovery »…), alors que `docker compose ps` montre Neo4j en `running`.

**Cause** : le formulaire propose `neo4j://localhost:7687`. C'est le port Bolt *dans le conteneur* ; sur votre machine, il est publié sur **17687**. Rien n'écoute sur 7687.

**Solution** : dans *Connect URL*, remplacez `7687` par `17687` (`neo4j://localhost:17687`), puis `neo4j` / `pixelhub_dev`. Si un port différent est défini dans votre `.env` (`NEO4J_BOLT_PORT`), utilisez celui-là.

### Neo4j refuse le mot de passe

Neo4j 5 exige un mot de passe d'au moins 8 caractères. Utilisez celui du fichier compose. Si le volume a déjà été initialisé avec un mot de passe refusé : `docker compose down -v`.

### L'API ne démarre pas sur un poste de l'IUT (`PixelHub.Api.exe` bloqué)

**Symptôme** : `dotnet run` compile, puis échoue au moment de lancer l'API ; le message met en cause `PixelHub.Api.exe`.

**Cause** : par défaut, `dotnet run` fabrique un exécutable `PixelHub.Api.exe` et le lance. Sur certains postes, cet `.exe` est bloqué.

**Solution** : lancez `dotnet run -p:UseAppHost=false`, qui démarre l'API via `dotnet` sans produire d'`.exe`. Pour ne plus avoir à le retaper, ajoutez `<UseAppHost>false</UseAppHost>` dans le `<PropertyGroup>` de `src/PixelHub.Api/PixelHub.Api.csproj` : le réglage vaut alors aussi pour `dotnet watch run`.

### `405 Method Not Allowed` sur `POST /players`

**Symptôme** : le `GET /players` fonctionne, mais le `POST` répond `405`. Dans les en-têtes de la réponse : `Allow: GET`.

**Cause** : l'API qui tourne a été lancée **avant** que vous ajoutiez le `MapPost`. `dotnet run` compile au démarrage, puis ne relit plus vos fichiers. L'API connaît donc la route `/players`, mais seulement en `GET` — d'où `405` (« méthode non autorisée ») et pas `404` (« route inconnue »).

**Solution** : dans le terminal de l'API, **Ctrl+C**, puis `dotnet run`. Ou lancez `dotnet watch run`, qui redémarre tout seul à chaque enregistrement. Si l'erreur persiste après redémarrage, vérifiez que le `MapPost` est bien écrit **avant** `app.Run()`.

### `psql` ne rend plus la main

Une commande `UPDATE` (ou `ALTER TABLE`) reste figée, sans message. Une **autre** session `psql` a une transaction ouverte sur les mêmes lignes : c'est le verrou de l'étape 5.2. Retrouvez l'autre terminal et terminez sa transaction par `ROLLBACK;`. Si vous ne le retrouvez plus, fermez-le : une session fermée annule sa transaction.

### `ERROR: current transaction is aborted, commands ignored until end of transaction block`

Une commande précédente de la transaction a échoué (étape 5.3). Plus rien ne passe tant que la transaction n'est pas terminée : tapez `ROLLBACK;`.

### `ERROR: relation "players" does not exist`

Les guillemets doubles manquent : écrivez `"Players"`, `"Pseudo"`, `"Coins"`. Sans guillemets, PostgreSQL met le nom en minuscules.

### Le téléchargement des images n'en finit pas

30 postes qui téléchargent en même temps saturent le réseau. Pour démarrer l'étape 3, **seul PostgreSQL est nécessaire** :

```bash
docker compose up -d postgres
```

Lancez `docker compose up -d` dès que le réseau se libère : les trois autres moteurs sont nécessaires à l'étape 6.

---

## Pour la prochaine séance

Gardez sous les yeux votre modélisation relationnelle du catalogue de jeux (l'exercice ⭐ de `CM1_etudiants_Panorama.md`) — celle avec les colonnes vides ou les tables par genre. La séance 2 en est la résolution directe, et on la comparera à ce que vous aurez produit. Relisez aussi votre réponse à la Q14 : elle en est le premier pas.
