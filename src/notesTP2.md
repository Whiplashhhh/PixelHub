Q1. Vous retrouvez le champ _id, déjà aperçu au TP 1 (étape 6.2), que personne n'a écrit dans games.json. 
Cette fois, intéressez-vous à qui l'a fabriqué : le serveur MongoDB, ou l'outil qui a envoyé les documents (mongoimport) ? 
Comparez avec l'Id 4 de Pixel au TP 1 (Q8) : qui l'avait choisi, et à quel moment le connaissiez-vous ?

    Le champ _id est généré automatiquement par mongoimport.
    Au contraire c'est PostgreSQL qui avait choisi l'Id 4 au moment de l'insertion dans la base.

Q2. À l'exercice 5, plateformes est un tableau et vous avez écrit { plateformes: "Switch" }, sans opérateur particulier. 
Comment auriez-vous fait la même chose en SQL, avec un modèle relationnel normalisé ?

    EN système relationnel on aurait du avoir une table jeux et une table plateformes avec dans la table jeux une liste des clés étrangères vers la table plateformes. 
    On aurait fait une jointure entre les deux tables.

Q3. Vous tapez --authenticationDatabase admin depuis le TP 1 sans qu'on vous ait dit pourquoi. 
Pourquoi faut-il l'ajouter aux commandes mongoimport et mongosh ? 
Où se trouve le compte pixelhub ?

    L'option --authenticationDatabase admin est nécessaire pour indiquer à MongoDB que l'utilisateur pixelhub est dans la base admin. 
    Le compte pixelhub se trouve dans la base de données admin.

Q4. Personne ne vous a arrêté. Est-ce une bonne nouvelle ou un problème ? 
Argumentez en imaginant un projet à quatre développeurs, six mois plus tard.

    Je trouve que c'est un problème car ça veut dire que ça peux laisser place à des fautes de frappes, des champs nommés différemment, et sur le long terme ça pouraait créer de gros problèmes de maintenance.
    Mais ça dépends de l'usage, pour certain document ça peut être pratique de ne pas avoir de contraintes.
    Si on immagine un projet à quatre développeurs, six mois plus tard, tout le monde n'aura pas forcément les mêmes habitudes et ça pourrai créer des incohérences dans la base de données.
    Il faudrait mettre en place des conventions.

Q5. Le résultat de l'exercice 9 vous surprend-il ? Que s'est-il passé exactement, et pourquoi n'y a-t-il eu aucun message d'erreur ?

    Je trouve ça surpennant qu'il n'y ai pas d'erreur, la collection s'appelle 'jeux' pas 'Jeux', donc la requête n'a rien trouvé.
    le fait de chercher une collection qui n'existe pas ne génère pas d'erreur, mais renvoie simplement un tableau vide.

Q6. L'exercice 16 échoue. Recopiez le message d'erreur, et expliquez pourquoi MongoDB refuse. Quelle commande utiliseriez-vous si vous vouliez réellement remplacer le document en entier ?

    MongoInvalidArgumentError: Update document requires atomic operators
    MongoDB refuse car la commande update nécessite des opérateurs atomiques pour modifier un document existant. 
    Si on veut remplacer le document en entier, on peut utiliser la commande replaceOne() à la place de updateOne().

Q7. À l'exercice 12, vous avez ajouté un champ nbVotes à un seul document, alors que les dix autres n'en ont pas. 
Quelle commande SQL aurait été nécessaire pour faire l'équivalent en relationnel, et quelle en aurait été la conséquence sur les autres lignes ?

    En SQL il aurait fallu faire un ALTER TABLE pour rajouter une colonne qui sera vide partout sauf à une seule ligne.
    Les autres lignes auraient toutes eu un champ nbVotes NULL.

Q8. Lisez IGameCatalog. Y a-t-il un seul endroit dans cette interface où le mot « Mongo » apparaît ? Pourquoi est-ce important, à votre avis ?

    Non, il n'y a pas de mention de Mongo dans l'interface IGameCatalog. 
    C'est important car cela permet de séparer la logique métier du SGBD. 
    Comme ça il n'y a pas de dépendance directe avec MongoDB, et on peut changer de SGBD sans avoir à modifier la logique métier.

Q9. Quelle exception obtenez-vous, et sur quel champ ? 
Expliquez la cause : qu'est-ce que le driver a essayé de faire, et pourquoi n'a-t-il pas su ?

    System.FormatException: Element 'nombreArmes' does not match any field...
    Le driver a essayé d'assigner un champ 'nombreArmes' à un attribut de la classe Game mais cet attribut n'existe pas.

Q10. [BsonIgnoreExtraElements] fait disparaître l'erreur, mais au prix de quoi ? Que devient joueursParEquipe quand vous appelez /games ?

    Tous les champs qui ne sont pas des attributs de la classe Game sont ignorés, donc on perd l'information contenue dans ces champs.

Q11. Comparez les trois approches possibles pour gérer un schéma variable en C# : [BsonIgnoreExtraElements], [BsonExtraElements], et une hiérarchie de classes (FpsGame : Game, etc.). 
Quel est l'avantage et l'inconvénient de chacune ?

    [BsonIgnoreExtraElements] : Avantage : simple à mettre en place, Inconvénient : on perd les données supplémentaires.
    [BsonExtraElements] : Avantage : on peut récupérer les données supplémentaires, Inconvénient : il faut gérer un dictionnaire pour accéder aux données.
    Hiérarchie de classes : Avantage : on peut avoir des classes spécialisées pour chaque type de jeu, Inconvénient : il faut connaître à l'avance tous les types de jeux et créer une classe pour chacun.

Q12. Écrivez la requête SQL qui produirait le même résultat sur une table jeux(titre, genre, note). Comparez les deux. 
Est-ce ici que MongoDB apporte quelque chose par rapport à PostgreSQL ? Si non, où l'apport se situe-t-il dans ce TP ?

    SELECT genre, AVG(note) AS note_moyenne, COUNT(*) AS nombre
    FROM jeux
    GROUP BY genre
    ORDER BY note_moyenne DESC;

    Les deux font exactement la même chose : $group = GROUP BY, $sort = ORDER BY
    Donc non, MongoDB n'apporte rien ici.
    L'apport de MongoDB dans ce TP est un schéma flexible : chaque jeu peut avoir ses propres champs
    et des tableaux directement dans le document sans créer de tables ou faire des jointures.

Q13. Les requêtes d et e ne renvoient pas la même chose. Laquelle a raison ? Que demande exactement la requête d, et pourquoi trouve-t-elle Stardew Valley alors que Krayz lui a mis 4 ?

    C'est la requête e qui a raison car Krayz n'a mis 5 à aucun jeu, donc le résultat vide est correct.
    La requête d demande : "un jeu qui a un avis écrit par Krayz ET un avis noté 5", mais pas forcément le même avis.
    Stardew Valley a un avis de Krayz a 4 et un autre avis a 5 par Nova, donc les deux conditions sont vraies mais pas en même temps
    $elemMatch oblige les deux conditions à être vraies sur le même élément

Q14. Combien d'octets coûte un avis, en moyenne ? Un document MongoDB ne peut pas dépasser 16 Mo (16 777 216 octets) : à partir de combien d'avis environ Stardew Valley ne pourrait-il plus en accueillir ? Est-ce un risque réel pour un jeu populaire ? Rapprochez votre réponse des critères du cours.

    On est passé de 273 octets a 516 octets pour 3 avis, donc 81 octets par avis.
    Il reste 16 777 216 - 516 = 16 776 700 octets, donc 16 776 700 / 81 = environ 207 000 avis maximum.
    Oui c'est un risque réel car un jeu populaire peut avoir des centaines de milliers d'avis qui peuvent aussi être plus conséquents
    Il vaut donc mieux référencer les avis dans une collection séparée plutôt que de les imbriquer dans le jeu.



aller jusqu'à la question 14, le reste est bonus