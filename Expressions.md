**Vous trouverez dans cette page des exemples d'expressions Power Automate contruites pour des besoins spécifiques.**

- [Filtrer les résultats de requète SharePoint](#filtrer-les-résultats-de-requète-sharepoint)
  - [A la source via le connecteur natif "Obtenir les éléments"](#a-la-source-via-le-connecteur-natif-obtenir-les-éléments)
  - [En filtrant via l'opération de donnée "Fitrer un tableau"](#en-filtrant-via-lopération-de-donnée-fitrer-un-tableau)
  - [En filtrant avec une requête HTTP SharePoint](#en-filtrant-avec-une-requête-http-sharepoint)
    - [Si on utilise OData](#si-on-utilise-odata)
    - [Si on utilise CAML](#si-on-utilise-caml)
- [Contrôler si le résultat d'une requête, d'un filtrage ou d'une propriété est vide](#contrôler-si-le-résultat-dune-requête-dun-filtrage-ou-dune-propriété-est-vide)
- [Éviter les boucles sur les requêtes ne renvoyant qu'un seul résultat](#éviter-les-boucles-sur-les-requêtes-ne-renvoyant-quun-seul-résultat)
- [Forcer la mise à jour d'un champ de type Lookup d'un élément SharePoint pour le remettre vide](#forcer-la-mise-à-jour-dun-champ-de-type-lookup-dun-élément-sharepoint-pour-le-remettre-vide)
- [Formatter une date en provenance d'un champ texte pour mettre à jour un élément SharePoint](#formatter-une-date-en-provenance-dun-champ-texte-pour-mettre-à-jour-un-élément-sharepoint)
- [Formater un tableau HTML](#formater-un-tableau-html)


# Filtrer les résultats de requète SharePoint

## A la source via le connecteur natif "Obtenir les éléments"

C'est la méthode recommandé, Il suffit d'ajouter une requête de filtre, cette requête est du type OData.
Il faut utiliser les noms internes des champs notamment en cas d'utilisation de caractère spéciaux ou d'espace et il faut toujours mettre entre simple quote les valeurs comparatives de type String par exemple :

    MyInternalColumnName eq '@{variables('MyVariable')}'

La syntaxe n'est pas clairement documentée et dépend des api interrogées toutefois on trouve des exemples simples sur la documentation dynamics 365 https://docs.microsoft.com/en-us/previous-versions/dynamicsnav-2016/hh169248(v=nav.90)

Une petite synthèse des opérateurs les plus courants :

* Equals : eq
* Does not equal : ne
* Greater than : gt
* Greater than or equal to : ge
* Less than	: lt
* Less than or equal to : le
* And operator : and
* Or operator :	or
* Search for substring ‘abc’ in field ‘Title’ :	substringof(‘abc’, Title)

Le champs de requète peut aussi contenir une epxression Power automate ce qui nous permet d'utiliser toutes les expressions disponibles dans Power Automate. On peut donc simplement retravailler notre valeur comparative sans utiliser la syntaxe OData.

Voici un exemple concret pour un flow d'archivage :

    (P_x00e9_riode eq '@{variables('PeriodPlus3Ans')}') or (Modified lt '@{addDays(utcnow(), -1100)}')

Et un autre pour faire une relance sur les documents non reçus à J-1,J et J+1 de leur deadline de réception :

    (Deadline eq '@{formatDateTime(addDays(utcnow(), +1), 'yyyy-MM-dd')}' and Received eq 'False') or (Deadline lt '@{formatDateTime(utcnow(), 'yyyy-MM-dd')}') or (Deadline eq '@{formatDateTime(utcnow(), 'yyyy-MM-dd')}')

On peut aussi travailler sur des sous-valeurs d'une colonne en ajoutant un '/':

    Monitoring0/Id eq '5'


## En filtrant via l'opération de donnée "Fitrer un tableau"

Pour filtrer on choisit la sortie Value de notre requête SharePoint

    outputs('Obtenir_les_éléments_SharePoint')?['body/value']

Puis on peut choisir la propriété à comparer :

    item()?['MycolumToCompare']

Si besoin et pour éviter une boucle on peut aussi aller comparer une sous-valeur du json en entrant soi même l'expression, par exemple avec l'email du créateur :

    item()?['Author/Email']

Puis on ajoute note comparateur et la valeur attendue.

## En filtrant avec une requête HTTP SharePoint

On utilise le composant "Envoyer une requête HTTP à SharePoint".

Il suffit ensuite de choisir notre site, d'utiliser la méthode POST avec l'URI de la méthode souhaité de l'API REST SharePoint.

Avec l'API on peut utiliser les filtres OData ou CAML.

### Si on utilise OData

Le filtre sera appelé via $filter mais on utilisera aussi utiliser $select pour choisir les champs à récupérer et $expand pour obtenir des propriétés sous-jacentes.

Par exemple je veux récupérer les élements d'une seul utilisateur en fonction de son email :

    _api/web/lists/getbytitle('MyList')/items?$select=ID,PersonOrGroupField/Id,PersonOrGroupField/Name,PersonOrGroupField/Title&$expand=PersonOrGroupField&$filter=substringof('username@email.com',PersonOrGroupField/Name)

La documentation  Microsoft sur l'api REST SharePoint n'est pas exhausitive mais elle permet de comprendre rapidement ce qu'il est possible de faire :
 https://docs.microsoft.com/en-us/sharepoint/dev/sp-add-ins/use-OData-query-operations-in-sharepoint-rest-requests#OData-query-operators-supported-in-the-sharepoint-rest-service

 On trouve facilement des exemples de requète d'api en OData sur Internet que ce soit dans le contexte Power Automate ou purement API Rest SharePoint. 

### Si on utilise CAML

Le langage CAML va permettre d'aller beaucoup plus loin dans les requêtes et de répondre à des scénarios avancés.

Voici le point d'entrée de la documentation Microsoft :
https://docs.microsoft.com/en-us/sharepoint/dev/schema/introduction-to-collaborative-application-markup-language-caml


J'ai par exemple utiliser la syntaxe suivante pour trouver dans une bibliothèque de document tous les fichiers qui avaient une colonne de métadonnée vide :

    _api/Web/lists/getByTitle('@{variables('Library')}')/GetItems(query=@v1)?@v1={"ViewXml":"<View Scope='Recursive'><Query><Where><And><Eq><FieldRef Name='FSObjType' /><Value Type='Lookup'>0</Value></Eq><IsNull><FieldRef Name='@{variables('MetadataInternalColumnName')}'/></IsNull></And></Where></Query></View>"}

La requète CAML est entre accolade et appelé via 'GetItems(query=@v1)?@v1='
C'est une requète récursive grâce à l'utilisation de l'option 'scope' de la balise 'View', j'utilise ici deux variables Power Automate, une pour la liste et une autre pour le nom interne de la colonne.

On trouve facilement des exemples de requète CAML sur Internet en se focalisant sur le développement SharePoint et l'interrogation de son API Rest plus que sur le scripting ou les flows Power Automate.


# Contrôler si le résultat d'une requête, d'un filtrage ou d'une propriété est vide

Le plus simple est d'utiliser l'expression 'empty' sur notre Body de sortie du filtrage, de la requête ou du champ soit par exemple :

    empty(body('Filtrer_mon_résultat SharePoint'))
    empty(items('Appliquer_à_chaque_élement')?['InternalColumnName'])

Parfois lorsqu'un élement SharePoint est renvoyé la propriété que nous cherchons n'est pas présente dans le résultat json. Cela peut avoir des impacts sur vos expressions 'empty' qui vont alors renvoyées des erreurs.

Une alternative est alors de regarder si le résultat de la requête contient une référence à la colonne :

    contains(items('Pour_chaque_élement'), 'InternalColumnName')

Si le résultat est égale à l'expression 'true' alors vous pouvez en déduire que la colonne contient une valeur et l'utiliser dans les expressions suivantes.


# Éviter les boucles sur les requêtes ne renvoyant qu'un seul résultat

Pour éviter les boucles ajouter automatiquement par Power Automate le plus simple est d'indiquer que l'on veut travailler sur le premier élement de l'array en ajoutant l'index. Soit par exemple :

    body('Filtrer_mon_résultat SharePoint')[0]?['InternalColumnName]


# Forcer la mise à jour d'un champ de type Lookup d'un élément SharePoint pour le remettre vide

Lorsque l'on met à jour un élement SharePoint il faut parfois forcer la remise à vide d'un champ de type LookupUp par exemple si on retrouve pas la valeur souhaité dans le référentiel lié. Pour cela j'ajoute une expression conditionnel dans la valeur du champ.
Ce qui peut donner par exemple l'expression suivante :

    if(empty(body('Filtrer_mon_résultat SharePoint')),null,body('Filtrer_mon_résultat SharePoint')[0]?['Title'])


# Formatter une date en provenance d'un champ texte pour mettre à jour un élément SharePoint

Lorsque l'on souhaite créer ou mettre à jour un élement SharePoint avec un champ date, il faut la mettre au format 'yyyy-MM-dd'.

Dans l'exemple ci-dessous, j'ai un fichier csv avec une colonne 'Date de signature' au format français et je souhaite que la colonne correspondante dans SharePoint soit null si la colonne est vide ou si elle contient '00/00/0000" sinon qu'elle soit mise à jour avec ma date de signature que je vais correctemment reformaté :

    if(or(equals(items('Appliquer_à_chaque_lignes_de_l''import')?['Date de signature'], '00/00/0000'), empty(items('Appliquer_à_chaque_lignes_de_l''import')?['Date de signature'])), null, formatDateTime(concat(substring(items('Appliquer_à_chaque_lignes_de_l''import')?['Date de signature'], 6, 4), '-', substring(items('Appliquer_à_chaque_lignes_de_l''import')?['Date de signature'], 3, 2), '-', substring(items('Appliquer_à_chaque_lignes_de_l''import')?['Date de signature'], 0, 2)), 'yyyy-MM-dd'))


# Formater un tableau HTML

Dans cet exemple mon flow Power Automate traite des documents et renvoie le résultat de chaque traitement dans deux variables de type tableau initialisés au début du flow.
Une de variables va contenir les résultats OK et l'autre les résultats KO.


Chaque résultat de traitement est ajouté via l'action "Ajouter à la variable de tableau" avec 4 propriétés configurés sur des valeurs de variable définies pendant la traitement :

    {
    "Status": "@{variables('ResultatIdentificationPJ')}",
    "Attachment": "@{items('Appliquer_à_chaque_pièce_jointe')?['name']}",
    "Entity": "@{variables('ResultatIdentificationSociete')}",
    "Document Type": "@{variables('ResultatIdentificationDocument')}"
    }


Je vais ensuite constuire un tableau HTML pour chacune de ces variables via l'action d'opération de donnée du même nom en laissant les colonnes en automatique.


Puis je vais procéder en deux temps pour formater chacun de ces tableaux via l'action d'opération de donnée "Message".

Première étape, je vais formater le statut OK ou KO pour celui-ci apparaisse en vert ou en rouge en remplaçant les balises.

    replace(body('Créer_un_tableau_HTML_des_résultats_OK'),'<td>OK</td>','<td style="background-color:green;color: white;text-align:center">OK</td>')

    replace(body('Créer_un_tableau_HTML_des_résultats_KO'), '<td>KO</td>', '<td style="background-color:red;color: white;text-align:center">KO</td>')


Seconde étape, je vais mettre en forme les tableaux pourqu'ils soient plus lisibles et colorés (Lignes visible de 1px, entêtes en gras, fond gris et un peu plus focné pour l'entête). Le format est identique pour les deux tableaux.

    replace(outputs('Format_du_tableau_HTML_des_résultats_OK_(Status)'),'<table>','<style>table{color:#333;border-collapse:collapse;border-spacing:0;}td,th{border:1px solid #CCC;height:30px;padding:10px}th{background:#F3F3F3;font-weight:bold;}td{background:#FAFAFA;text-align:center;}</style><table>')
 
    replace(outputs('Format_du_tableau_HTML_des_résultats_KO_(Status)'),'<table>','<style>table{color:#333;border-collapse:collapse;border-spacing:0;}td,th{border:1px solid #CCC;height:30px;padding:10px}th{background:#F3F3F3;font-weight:bold;}td{background:#FAFAFA;text-align:center;}</style><table>')



Pour finir les tableaux vont être ajoutés dans des mails selon 4 cas de figures conditonnés par le fait que les variables soit vides:

* Pas de tableau (Variables des tableaux OK et KO vide)
* Tableau de résultats OK (Variable de tableau KO vide)
* Tableau de résultats KO (Variable de tableau OK vide)
* Tableaux des résultats OK et KO (Variables des tableaux OK et KO ne sont pas vides)

Le contenu du message est adapté à chaque cas de figure et pour certain je passe en mode Code pour ajouter des messages spécifiques.

On pourra dans chaque mails ajouter le ou les tableaux en prenant la dernière sortie de formatage, dans mon cas :

    outputs('Format_du_tableau_HTML_des_résultats_OK_(Table)')

    outputs('Format_du_tableau_HTML_des_résultats_KO_(Table)')


**Attention, lorsque l'on ajoute plusieurs tableaux dans le corps du message brut, le style de ceux-ci n'est plus pris en compte. Il faut alors ajouter le style directemment dans le corps du mail en mode code avant la balise \<p> :**


    <style>table{color:#333;border-collapse:collapse;border-spacing:0;}td,th{border:1px solid #CCC;padding:10px}th{background:#F3F3F3;font-weight:bold;}td{background:#FAFAFA;text-align:center}</style>

