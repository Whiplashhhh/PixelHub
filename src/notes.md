Q1. Le fichier déclare quatre services alors que le TP d'aujourd'hui n'en utilise qu'un. Pourquoi, à votre avis ?

    On utilisera uniquement le service Postgres car c'est là dessus qu'on greffera tout le reste.
    
Q2. Trois services ont une section volumes, un seul n'en a pas. Lequel, et qu'est-ce que ça implique pour ses données ?

    Le service api n'en a pas, car il ne contient aucune datas qu'on pourrait devoir sauvegarder.

Q3. Que signifie la ligne "15432:5432" ? Les deux nombres ne désignent pas la même chose.

    ça signifie que le port de PostgreSQL dans le conteneur est 5432; mais que sur la machine hôte ça sera 15432.

Q4. Les mots de passe sont écrits en clair dans le fichier. Est-ce acceptable ici ? Le serait-ce sur un serveur de production ?
    
    On est sur un environnement de dev donc c'est acceptable mais en production il faudra absolument éviter.

Q5. Les commandes ci-dessus utilisent docker compose exec. Qu'est-ce que ça fait exactement ? Où s'exécute la commande redis-cli ?

    exec permet d'éxécuter une commande diretement à l'intérieur du conteneur. redis-cli s'éxécute dans le conteneur redis.

Q6. La base de données tourne dans un conteneur Docker, et la chaîne de connexion dit Host=localhost. Pourquoi est-ce que ça fonctionne ? (Indice : relisez votre réponse à la Q3.)

    On accède au conteneur grâce au bind de port donc ici le port 15432 du localhost.

Q7. Arrêtez l'application (Ctrl+C), relancez-la. Les trois joueurs sont-ils dupliqués ? Pourquoi ?

    Ils ne sont pas dupliqués car les données sont stockées dans un volume qui est persistant même après l'arrêt du conteneur et qu'on vérifie si il n'y a aucun joueur avant de créer ceux là.

Q8. Dans votre code, que vaut player.Id avant l'appel à SaveChangesAsync() ? Et après ? Qui a choisi la valeur 4 : votre code, EF Core ou PostgreSQL ?

    Avant l'appel à SaveChangesAsync(), player.Id vaut 0 car il n'a pas encore été sauvegardé dans la base de données. Après l'appel c'est PostgreSQL qui a choisi la valeur 4.

Q9. Que valent les soldes après le ROLLBACK ? Imaginez maintenant que le serveur s'éteigne entre les deux UPDATE, sans transaction : dans quel état serait la base, et qui y perdrait ?

    Après le rollback, les soldes reviennent à leur état initial avant les deux UPDATE. Si le serveur s'éteint entre les deux UPDATE sans transaction, la base de données pourrait se retrouver dans un état incohérent où un seul des deux changements est appliqué. Cela pourrait entraîner une perte financière pour l'un des joueurs.

Q10. Pendant la transaction de A, quel solde de Nova voyaient B et l'API ? Pourquoi l'UPDATE de B a-t-il dû attendre ? Imaginez que A et B soient deux achats lancés au même instant par Nova sur deux téléphones : que pourrait-il se passer si la base ne bloquait pas B ?

    Pendant la trasaction de A, B et l'API voyaient 1200. 
    L'UPDATE de B a dû attendre car la base de données utilise un verrouillage pour garantir l'intégrité des données pendant la transaction de A. 
    Si A et B étaient deux achats lancés au même instant par Nova sur deux téléphones, sans verrouillage, il pourrait y avoir une condition de course où les deux transactions tenteraient de modifier le solde en même temps, ce qui pourrait entraîner des résultats incohérents ou incorrects dans la base de données.

Q11. Qui a refusé l'achat d'Ombre : votre code C# ou la base ? Pourquoi le SELECT suivant a-t-il échoué lui aussi ? Après le ROLLBACK, la contrainte existe-t-elle toujours, et qu'est-ce que ça vous apprend sur PostgreSQL ?

    C'est la base qui a refusé l'achat.
    Le SELECT a échoué car la contrainte d'intégrité a été violée, empêchant l'insertion de données qui ne respectent pas les règles définies. 
    Après le ROLLBACK, la contrainte n'existe plus, ce qui montre que PostgreSQL gère les transactions de manière à maintenir l'intégrité des données et à revenir à l'état précédent en cas d'échec d'une transaction.

Q12. Que renvoie TTL temporaire au fil des secondes, puis une fois les 10 secondes écoulées ? Et GET temporaire ? Citez une donnée de PixelHub qui gagnerait à disparaître toute seule.

    TTL temporaire renvoie le temps restant avant l'expiration de la clé. 
    Une fois les 10 secondes écoulées, la clé n'existe plus et TTL renvoie -2. 
    GET temporaire renvoie la valeur associée à la clé tant qu'elle n'a pas expiré, puis renvoie nil une fois la clé expirée. 
    Une donnée de PixelHub qui gagnerait à disparaître toute seule pourrait être un token d'authentification temporaire ou un code de vérification envoyé par email pour confirmer une action.

Q13. INCR lit, ajoute 1 et réécrit en une seule commande. Pourquoi est-ce plus sûr que de faire un GET, d'ajouter 1 dans le code C#, puis un SET, si 200 joueurs déclenchent le compteur au même moment ? (Pensez à ce que vous avez vu en 5.2.)

    INCR est plus sûr car il est atomique, ce qui signifie que toutes les opérations se font en une seule étape. 
    Cela empêche les conditions de course où plusieurs joueurs pourraient lire la même valeur avant qu'elle ne soit mise à jour, ce qui pourrait entraîner des résultats incorrects ou incohérents. 
    En utilisant INCR, chaque joueur obtient une valeur unique et correcte du compteur, même si plusieurs joueurs déclenchent l'opération en même temps.

Q14. Les deux jeux sont dans la même collection, mais n'ont pas les mêmes champs (plateformes est une liste, multijoueur un objet imbriqué). Aurait-on pu ranger les deux dans une même table PostgreSQL ? À quel prix ? Rapprochez votre réponse de l'exercice du catalogue de jeux (section 2 du cours CM1_etudiants_Panorama.md).

    Oui, on aurait pu ranger les deux jeux dans une même table PostgreSQL, mais cela aurait nécessité de créer des colonnes supplémentaires pour chaque champ spécifique à chaque jeu, ce qui pourrait entraîner beaucoup de colonnes vides pour les jeux qui n'ont pas ces champs. 
    Cela pourrait également compliquer les requêtes et la gestion des données, car il faudrait gérer les valeurs nulles et les types de données différents pour chaque jeu.

Q15. La requête « ami d'un ami » se lit presque comme un dessin. Comment l'écririez-vous en SQL, avec une table Amities(joueur_id, ami_id) ? Combien de jointures faudrait-il pour « ami d'un ami d'un ami » ?

    SELECT a2.ami_id
    FROM Amities a1
    JOIN Amities a2 ON a1.ami_id = a2.joueur_id
    WHERE a1.joueur_id = :joueur_id;

    Pour « ami d'un ami d'un ami », il faudrait deux jointures supplémentaires, soit un total de trois jointures pour atteindre le troisième niveau d'amitié.

Q16. Pourquoi DELETE seul a-t-il été refusé ? Que fait DETACH de plus ?

    Il faut d'abord supprimer les relations entre les entités. 
    DETACH supprime également les relations entre les entités, mais ne supprime pas l'entité elle-même.

Q17. La clé survivant a-t-elle survécu au stop ? Au down ? Et les joueurs de PostgreSQL ? Expliquez la différence avec le mot volume.

    La clé survivant a survécu au stop mais pas au down, le down supprime les conteneurs et redis n'as pas de volume.
    Les joueurs de PostgreSQL ont survécu au stop et au down car ils sont stockés dans un volume

