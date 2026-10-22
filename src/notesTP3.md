Q1. INCR est une opération atomique. Qu'est-ce que ça veut dire concrètement, et pourquoi est-ce important si mille joueurs terminent une partie en même temps ? Comment auriez-vous fait la même chose en SQL, et quelle différence cela fait-il ?

    Ca veut dire que tout se fait d'un coup, sans que d'autres opérations puissent interférer. 
    C'est important car si mille joueurs terminent une partie en même temps, on veut être sûr que le compteur de parties jouées est correctement incrémenté pour chacun d'eux. 
    En SQL, on pourrait utiliser une transaction pour l'atomicité, mais cela peut être plus complexe et moins performant que l'opération INCR de MongoDB

Q2. On aurait pu sérialiser tout le joueur en JSON dans une simple String. Décrivez les étapes qu'il aurait alors fallu faire pour ajouter 250 XP, et le problème qui apparaît si deux requêtes le font en même temps.

    Il faudrait d'abord récupérer le joueur depuis la base de données, le désérialiser en objet, ajouter 250 XP à son compteur, puis le ré-sérialiser en JSON et le mettre à jour dans la base de données. 
    Si deux requêtes le font en même temps, elles pourraient lire la même valeur, ajouter 250 XP chacune, et écrire la même valeur à la fin, ce qui signifie que les 250 XP ne sont comptabilisés qu'une seule fois.

Q3. Les deux dernières commandes ne renvoient pas la même chose. Expliquez la différence, et dites laquelle vous utiliserez pour afficher un « top 3 ».

    ZREVRANGE affiche la liste triée dans l'ordre décroissant et ZRANGE dans l'ordre croissant.
    pour afficher le top 3 (donc les 3 plus gros scores) on utilisera ZREVRANGE.

Q4. ZREVRANK renvoie un nombre. Si vous affichez ce nombre tel quel au joueur, que va-t-il lire ? Que faut-il faire ?

    il va lire son index, pas son classement donc il faut ajouter 1.

Q5. Qui sort de la file au RPOP, et pourquoi ? Qu'aurait donné LPOP à la place ?

    Avec RPOP on enlève le dernier élément de la liste. 
    LPOP aurait enlevé le premier élément.

Q6. Quelles valeurs obtenez-vous ? Que renvoie TTL sur une clé sans expiration (testez avec compteur:parties) ? Que signifie chacune des valeurs de retour ?

    TTL    enligne:Nova     -> -2 : la clée n'existe pas
    EXISTS enligne:Nova     ->  0 : la clée n'existe pas
    TTL    compteur:parties -> -1 : la clée existe et n'a pas d'expiration
    EXISTS compteur:parties ->  1 : la clée existe

Q7. Personne n'a lancé de tâche de nettoyage. Quel code auriez-vous dû écrire pour obtenir le même comportement avec PostgreSQL, et quels risques ce code introduit-il ?

    PostgreSQL ne supprime pas les lignes tout seul. 
    Il faudrait ajouter une colonne avec la date d'expiration, filtrer chaque requête dessus, et lancer régulièrement une tâche qui supprime les lignes expirées.

    Risques : 
    oublier le filtre dans une requête (on affiche des joueurs déconnectés), des lignes expirées qui restent en attendant le prochain nettoyage, et si la tâche plante, plus rien n'est supprimé sans que personne ne le voie.

Q8. Ce service est enregistré en singleton. 
Cherchez dans la documentation de StackExchange.Redis ce qu'est un ConnectionMultiplexer et ce qui se passerait si on en créait un à chaque requête HTTP. 
Pourquoi ce bug serait-il invisible aujourd'hui en TP ?

    Le ConnectionMultiplexer partage une même connexion à Redis entre tous les appels ; il est fait pour être créé une seule fois et réutilisé.
    En créer un par requête ouvrirait une nouvelle connexion à chaque fois, ce qui finirait par saturer Redis et ralentir l'API.
    En TP c'est invisible car il n'y a que quelques requêtes, donc jamais assez de connexions pour poser problème.

Q9. Supprimez Order.Descending des deux méthodes, relancez, et regardez /leaderboard/top/5. 
Le résultat est-il visiblement faux ? 
Pourquoi ce bug est-il particulièrement dangereux ?

    Non ça n'est pas visiblement faux.
    C'est dangereux car c'est un bug silencieux car en voulant les meilleurs on a les pires.

Q12. Combien de lignes avez-vous modifiées dans vos endpoints pour ajouter ce cache ? Pourquoi ? 
Quel choix de la séance 2 rend cela possible ?


