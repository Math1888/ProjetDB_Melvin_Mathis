XTI107 – Bases de données 1 : Concepts de base
Projet – Partie 1 : Modèle Conceptuel de Données (MCD)

Groupe : Melvin Davitian, Mathis Fortin Correia
Sujet choisi : Sujet n°1 – Plateforme de CTF (Capture The Flag)
Enseignante : Lena Trebaul

1. Choix du sujet

Nous avons choisi le sujet n°1 (plateforme de CTF). Une association étudiante de cybersécurité souhaite une plateforme permettant d'organiser des compétitions de CTF : les joueurs se regroupent en équipes, s'inscrivent à des événements, résolvent des épreuves classées par domaine et par difficulté en soumettant des flags, et peuvent débloquer des indices payants en points. Un classement par score est établi pour chaque événement.

2. Modèle Conceptuel de Données (MCD)

Le MCD a été modélisé avec Looping, un outil de modélisation MERISE dédié (plutôt qu'un schéma dessiné à la main) : entités, associations, attributs, identifiants et cardinalités sont saisis directement dans l'outil, qui gère le formalisme E/A. Le fichier source (MCD_CTF_Looping.loo) est versionné dans ce dépôt, ainsi qu'un export image (MCD_CTF_Looping.png, capture ci-dessous).

Une première version du modèle avait été construite avec Graphviz (notation DOT) pour itérer rapidement sur la structure ; elle reste disponible dans ce dépôt (MCD_CTF.dot et ses exports MCD_CTF.svg / .pdf) à titre de référence, mais c'est la version Looping ci-dessous qui constitue le rendu officiel.

MCD de la plateforme CTF (Looping)

Légende : les rectangles jaunes sont des entités — y compris INSCRIPTION, DISQUALIFICATION, SOUMISSION et DEBLOQUE, qui sont des associations réifiées en entités (voir justification 3.3) ; les ellipses cyan sont les associations porteuses de cardinalités.

Remarque sur la spécialisation PARTICIPANT : la contrainte « un participant est soit un joueur, soit un organisateur » est modélisée par les deux associations EST_JOUEUR et EST_ORGANISATEUR, chacune de cardinalité (0,1) côté PARTICIPANT. La version Graphviz représentait en plus explicitement cette spécialisation comme totale et exclusive (symbole CIF « T,X »), une notation que Looping n'exprime pas nativement de la même façon ; le caractère exclusif et obligatoire du rôle reste néanmoins une règle de gestion à vérifier applicativement (cf. section 4).

3. Dictionnaire des données et justifications de modélisation
3.1 Spécialisation de PARTICIPANT (contrainte T,X)

L'énoncé précise qu'« un participant est soit un joueur, soit un organisateur (un participant n'a qu'un seul rôle) ». Plutôt que d'ajouter un attribut role de type énuméré sur l'entité PARTICIPANT, nous avons modélisé une spécialisation totale et exclusive (T,X) entre PARTICIPANT et ses deux sous-types JOUEUR et ORGANISATEUR.

Utiliser une spécialisation plutôt qu'un simple attribut de rôle permet de rattacher directement à chaque sous-type les associations qui ne concernent que lui (un JOUEUR crée des équipes et soumet des flags, un ORGANISATEUR crée des événements et des épreuves) sans avoir à vérifier role = 'joueur' dans chaque contrainte applicative, et sans risquer qu'une association incohérente (ex. un organisateur capitaine d'équipe) soit représentable dans le schéma.

Note d'outillage : Looping ne propose pas de symbole graphique dédié pour une contrainte CIF totale/exclusive entre deux associations distinctes ; le modèle conserve donc ce choix de conception (spécialisation via deux associations EST_JOUEUR / EST_ORGANISATEUR), la totalité et l'exclusivité étant documentées ici et vérifiées applicativement (cf. section 4) plutôt que visibles sous forme de cercle « T,X » sur le diagramme.

3.2 Le capitaine d'équipe n'est pas un attribut redondant

Une équipe est « créée par un seul joueur, qui en devient le capitaine ». Nous avons choisi de ne pas dupliquer cette information par un attribut id_capitaine sur EQUIPE en plus de l'association d'appartenance : le capitaine est simplement le joueur lié à l'équipe par l'association CREE_EQUIPE (cardinalité 1,1 côté équipe : une équipe a exactement un créateur). Cela évite une redondance entre deux sources d'information (l'attribut et la relation d'appartenance) qui pourraient diverger.

L'appartenance « courante » à une équipe est modélisée par une association distincte, MEMBRE, portant l'attribut date_arrivee, avec la cardinalité (0,1) côté JOUEUR : « un joueur ne peut appartenir qu'à une seule équipe à la fois (ou à aucune) ». Deux associations différentes sont nécessaires ici car « créer une équipe » et « être membre actif » ne sont pas forcément synchrones dans le temps (un capitaine pourrait en théorie quitter son équipe) ; la contrainte « le capitaine est un membre de son équipe » reste une règle de gestion à vérifier par trigger (cf. section 4), le MCD ne pouvant pas exprimer de contrainte inter-associations.

La cardinalité (1,5) côté EQUIPE de l'association MEMBRE retranscrit la règle « une équipe compte entre 1 et 5 membres » ; comme toute cardinalité maximale dans un MCD, son maintien dynamique (interdire un 6ᵉ membre) relève d'un trigger au niveau du MLD/SQL, le MCD n'exprimant qu'une contrainte structurelle.

3.3 Réification de l'inscription et entité DISQUALIFICATION séparée

L'inscription d'une équipe à un événement est une association many-to-many porteuse de données (date_inscription), ce qui en MERISE classique suffirait à une simple association EQUIPE ↔ EVENEMENT. Cependant, l'énoncé ajoute qu'« un organisateur peut disqualifier une équipe d'un événement », avec une date et un motif conservés pour cette inscription précise.

Deux options étaient possibles :

ajouter des attributs date_disqualification et motif directement sur l'association d'inscription ;
réifier l'association d'inscription en entité INSCRIPTION (dotée de son propre identifiant id_inscription), puis créer une entité DISQUALIFICATION reliée à INSCRIPTION par une association (0,1)–(1,1).

Nous avons retenu la seconde solution : la première aurait introduit des attributs non renseignés pour la grande majorité des inscriptions (une équipe disqualifiée est l'exception), ce qui est contraire à l'esprit de la 3FN (un attribut doit décrire une propriété réellement caractéristique de l'entité/association qui le porte, pas un cas particulier rare) et aurait compliqué les requêtes de classement (il aurait fallu tester la nullité de plusieurs colonnes pour savoir si une équipe est toujours en course). En isolant la disqualification dans sa propre entité, une inscription « normale » n'a simplement aucune occurrence liée dans DISQUALIFICATION, ce qui est plus propre et plus proche de la sémantique métier (« disqualifiée » est un événement qui peut arriver à une inscription, pas une propriété intrinsèque de toute inscription).

3.4 SOUMISSION comme entité (et non comme simple association)

Un joueur « peut tenter plusieurs fois la même épreuve ». Une association classique entre JOUEUR et EPREUVE ne peut porter qu'une seule occurrence par couple (joueur, épreuve) : il est donc nécessaire de modéliser les tentatives comme une entité SOUMISSION dotée de son propre identifiant technique, reliée par deux associations (0,n)–(1,1) à JOUEUR et à EPREUVE. Cela permet un nombre illimité de tentatives par couple joueur/épreuve, chacune avec sa propre date, son flag proposé et son résultat.

3.5 DOMAINE comme entité plutôt qu'un attribut texte

Les domaines (Web, Cryptographie, Forensics…) sont « identifiés par un nom unique » et réutilisés par plusieurs épreuves : nous en avons fait une entité à part entière reliée à EPREUVE par une association (1,1)–(0,n), plutôt que de stocker le nom du domaine directement comme attribut texte de EPREUVE. Cela évite la duplication du libellé du domaine sur chaque épreuve (donc les incohérences de saisie, ex. « Web » vs « web ») et respecte la 3FN : le nom d'un domaine ne dépend que de la clé de DOMAINE, pas de celle de EPREUVE.

3.6 Score et classement : non modélisés comme attributs stockés

Le score d'une équipe à un événement (somme des points des épreuves validées, moins le coût des indices débloqués) est une donnée calculée, dérivable à partir de SOUMISSION, EPREUVE et DEBLOQUE. Nous ne l'avons donc pas représenté comme un attribut stocké sur INSCRIPTION ou EQUIPE : un attribut calculable à partir d'autres données du schéma est une redondance qui violerait la 3FN (dépendance fonctionnelle vers des données elles-mêmes dérivées d'autres entités) et risquerait de se désynchroniser. Le score sera calculé par une requête (vue SQL) lors du passage au MLD/SQL.

4. Règles de gestion non représentables structurellement dans le MCD

Certaines règles de gestion de l'énoncé sont transversales à plusieurs associations et ne peuvent pas être exprimées par de simples cardinalités ; elles seront implémentées par des triggers ou des contraintes applicatives lors du passage au modèle logique :

« Un participant est soit un joueur, soit un organisateur » (rôle exclusif et obligatoire) : dans la version Looping du MCD, cette règle est portée par les cardinalités (0,1) des associations EST_JOUEUR et EST_ORGANISATEUR, mais leur caractère exclusif (pas les deux à la fois) et total (toujours l'un des deux) n'est pas représentable par de simples cardinalités et sera vérifié par contrainte applicative (trigger ou CHECK) au niveau du MLD/SQL.
« Un joueur ne peut soumettre que pour les épreuves d'un événement auquel son équipe est inscrite » : nécessite de vérifier la chaîne JOUEUR → EQUIPE → INSCRIPTION → EVENEMENT → EPREUVE au moment de l'insertion d'une SOUMISSION.
« Une épreuve n'est comptabilisée qu'une seule fois dans le score d'une équipe, même si plusieurs de ses membres la valident » : règle de calcul du score, traitée au niveau de la requête de classement (agrégation par équipe avant somme, pas par joueur).
La taille d'équipe entre 1 et 5 membres (maintien dynamique du maximum).
Le capitaine (CREE_EQUIPE) doit rester cohérent avec l'appartenance (MEMBRE) : un organisateur ne peut pas être capitaine, et le capitaine est censé rester membre de son équipe.
5. Vérification de la 3ème forme normale (3FN)

Chaque entité du MCD respecte la 3FN :

Clé primaire unique et non décomposable pour chaque entité (identifiants techniques id_xxx), avec des clés candidates explicites quand elles existent (pseudo, email sur PARTICIPANT, nom sur EQUIPE et DOMAINE).
Aucune dépendance fonctionnelle partielle : toutes les entités ayant une clé simple (non composée), cette condition (propre à la 2FN) est trivialement vérifiée.
Aucune dépendance transitive : chaque attribut non-clé dépend directement et uniquement de la clé de son entité, jamais d'un autre attribut non-clé de la même entité. Les données qui décrivent un concept réutilisable et identifié indépendamment (le domaine d'une épreuve, le motif d'une disqualification) ont été extraites dans des entités séparées plutôt que dupliquées comme attributs (cf. sections 3.3 et 3.5), ce qui élimine les dépendances transitives potentielles.
Aucune donnée calculée stockée (score, classement), cf. section 3.6.
6. Pistes d'amélioration (bonus)

Quelques extensions pourraient enrichir la plateforme sans remettre en cause le cœur du modèle :

Historiser les équipes précédentes d'un joueur (garder une trace de tous les passages en équipe plutôt que seulement l'équipe courante), ce qui nécessiterait de transformer MEMBRE en entité historisée.
Ajouter une entité TAG/mots-clés multi-valués sur les épreuves en plus du domaine unique obligatoire, pour un classement secondaire transverse.
Tracer l'organisateur à l'origine d'une disqualification (ajout d'une association ORGANISATEUR ↔ DISQUALIFICATION).
Ajouter un système de badges/achievements pour les joueurs.
7. Fichiers du dépôt
Fichier	Description
README.md	Ce rapport
MCD_CTF_Looping.loo	Source Looping du MCD (modèle officiel)
MCD_CTF_Looping.png	Export image du MCD Looping (image utilisée ci-dessus)
MCD_CTF.dot	Source Graphviz de la première version du MCD (référence)
MCD_CTF.svg / MCD_CTF.png / MCD_CTF.pdf	Exports de la version Graphviz
