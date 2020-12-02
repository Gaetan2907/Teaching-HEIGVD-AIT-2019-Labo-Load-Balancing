## AIT - Labo3 

**Auteurs:** Barros Henriques Chris, Daubresse Gaëtan, Noël Eric





### Introduction

Le but de ce laboratoire a été d'approfondir notre compréhension de la notion de load balancing ainsi que de mettre en application certaines des notions vues en cours. De ce fait, nous avons regardé le comportement de HAProxy dans différentes situations ainsi que le résultat obtenu en effectuant des tests de charge avec JMeter. Nous avons pour cela commencé par modifier le comportement de la sticky session, du round robin, puis il nous a finalement été demandé d'apporter nous-mêmes deux modifications de notre choix à la configuration afin d'améliorer la stratégie du load balancing.



### Task1: Install the tools



> 1. Explain how the load balancer behaves when you open and refresh the
>    URL <http://192.168.42.42> in your browser. Add screenshots to
>    complement your explanations. We expect that you take a deeper a
>    look at session management.

On voit que le load balancer nous fait passer alternativement d'un serveur à l'autre. On remarque aussi qu'une nouvelle session est créée à chaque requête. On le déduit du fait que la variable *sessionView* ne change pas et que l'id de session quant à lui change.

```
# Premier JSON obteu après accès à la page
{"hello":"world!","ip":"192.168.42.22","host":"dec2aacceecb","tag":"s2","sessionViews":1,"id":"St42YWoXZqLxqULOxNg6MKkdb6lrHfoI"}


# JSON obtenu après rafraîchissement de la page depuis le browser
{"hello":"world!","ip":"192.168.42.22","host":"dec2aacceecb","tag":"s2","sessionViews":1,"id":"St42YWoXZqLxqULOxNg6MKkdb6lrHfoI"}
```



On peut le prouver en réinjectant le cookie généré par S1 lorsque l'on refait une requête vers celui-ci. Nous avons utilisé Burp comme cela:

**Requête 1:** On se connecte pour la première fois à S1, celui-ci nous envoie un cookie que nous interceptons et sauvegardons grâce à Burp. On obtient l'affichage suivant:

```
{"hello":"world!","ip":"192.168.42.11","host":"ab6d4d299d1b","tag":"s1","sessionViews":1,"id":"TaVIsL0EBYPvq8hoC5MilEQa8rFbrOdF"}
```

**Requête 2:** On réactualise la page pour joindre S2 pour pouvoir atteindre S1 à la prochaine requête faite.

**Requête 3:** On intercepte la requête envoyée côté cliet et on réinjecte le cookie. On voit que S1 reconnaît la session grâce au cookie et a incrémenté *sessionView*

 ```
{"hello":"world!","ip":"192.168.42.11","host":"ab6d4d299d1b","tag":"s1","sessionViews":2,"id":"TaVIsL0EBYPvq8hoC5MilEQa8rFbrOdF"}
 ```





> 2. Explain what should be the correct behavior of the load balancer for
>    session management.

Lorsqu'une session est crée, le load balancer devrait rediriger le client vers le même serveur, ce qui implique que l'id de session ne devrait pas changer et *sessionView* s'incrémenter comme indiqué au point 1. Ceci pourrait se faire notamment à l'aide de Cookies.



> 3. Provide a sequence diagram to explain what is happening when one
>    requests the URL for the first time and then refreshes the page. We
>    want to see what is happening with the cookie. We want to see the
>    sequence of messages exchanged (1) between the browser and HAProxy
>    and (2) between HAProxy and the nodes S1 and S2. Here is an example:
>
>   ![Sequence diagram for part 1](img/seq-diag-1.png)

```mermaid
sequenceDiagram
    participant A as Browser
    participant B as HAProxy
    participant C as S1
    participant D as S2
    A->>B: GET /
    B->>C: GET /
    C->>B: Response + Cookie1
    B->>A: Response + Cookie1
    
    A->>B: GET / + Cookie1
    B->>D: GET / + Cookie1
    D->>B: Response + Cookie2
    B->>A: Response + Cookie2
    
    A->>B: GET / + Cookie2
    B->>C: ...
```



> 4. Provide a screenshot of the summary report from JMeter.

![image-20201118160907282](./img/image-20201118160907282.png)



> 5. Run the following command:
>
>   ```bash
>   $ docker stop s1
>   ```
>
>   Clear the results in JMeter and re-run the test plan. Explain what
>   is happening when only one node remains active. Provide another
>   sequence diagram using the same model as the previous one.

On voit que si un serveur est down, les requêtes sont redirigées vers l'autre serveur.

![image-20201118161133861](./img/image-20201118161133861.png)

```mermaid
sequenceDiagram
    participant A as Browser
    participant B as HAProxy
    participant C as S1
    participant D as S2
    
    Note over C: DOWN 
    A->>B: GET /
    B->>D: GET /
    D->>B: Response + Cookie1
    B->>A: Response+ Cookie1
    A->>B: GET / + Cookie1
    B->>D: ...
```

### Task 2: Sticky sessions



> 1. There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).
>
>   * Choose one of the both stickiness approach for the next tasks.

**NODESESSID:**

L'application se trouvant sur l'un des serveurs génèrent un Cookie contenant un NODESESSID, puis l'envoie dans sa réponse au load balancer. Le load balancer préfixe ensuite l'ID précédent avec un tout nouvel ID.

```mermaid
sequenceDiagram
    participant A as Browser
    participant B as HAProxy
    participant C as S1
    participant D as S2
    
    A->>B: GET /
    B->>C: GET /
    C->>B: Response + Cookie(NODESSID)
    B->>A: Response + Cookie(NODESESSID ~ SERVERID)
    A->>B: GET / + Cookie(NODESESSID ~ SERVERID)
    B->>C: ...
```

**SERVERID:**

L'application génère également un NODESESSID à la réponse destinée au load balancer. Mais cette fois-ci, le load balancer génère un nouveau cookie appelé SERVERID puis envoie les deux cookies lors de sa réponse au client. 

```mermaid
sequenceDiagram
    participant A as Browser
    participant B as HAProxy
    participant C as S1
    participant D as S2
    
    A->>B: GET /
    B->>C: GET /
    C->>B: Response + Cookie(NODESSID)
    B->>A: Response + Cookie(NODESESSID) + Cookie(SERVERID)
    A->>B: GET / + Cookie(NODESESSID) + Cookie(SERVERID)
    B->>C: ...
```

Dans les deux cas, le load balancer pourra savoir à quel serveur rediriger la requête grâce à une table qu'il aura généré, liant un SERVERID avec un serveur.





> 2. Provide the modified `haproxy.cfg` file with a short explanation of
>    the modifications you did to enable sticky session management.



On a choisi la méthode avec le SERVERID. HAProxy va donc générer un nouveau cookie qui sera envoyé au client. Cecci est spécifie grâce à `    cookie SERVERID insert indirect nocache`. Le load balancer sait vers qui rediriger la requête grâce à `check cookie sX`. 

```
    #Add cookie policy
    cookie SERVERID insert indirect nocache
    # Define the list of nodes to be in the balancing mechanism
    # http://cbonte.github.io/haproxy-dconv/2.2/configuration.html#4-server
    server s1 ${WEBAPP_1_IP}:3000 check cookie s1
    server s2 ${WEBAPP_2_IP}:3000 check cookie s2
```





> 3. Explain what is the behavior when you open and refresh the URL
>    <http://192.168.42.42> in your browser. Add screenshots to
>    complement your explanations. We expect that you take a deeper a
>    look at session management.

On voit que lorsque l'on se connecte pour la première fois, on arrive sur un serveur précis et que la variable *sessionViews* est à 1:

```
{"hello":"world!","ip":"192.168.42.22","host":"1c9799a4d16f","tag":"s2","sessionViews":1,"id":"vrufdNfGvhuq0XvLScBzTYTLqcKuZj7F"}
```



Puis lorsque l'on actualise la page, on voit que le *sessionView* s'incrémente et que l'id et le host reste identique:

```
{"hello":"world!","ip":"192.168.42.22","host":"1c9799a4d16f","tag":"s2","sessionViews":2,"id":"vrufdNfGvhuq0XvLScBzTYTLqcKuZj7F"}
```

Ceci nous montre que la confiuration du sticky session fonctionne correctement.



On remarque aussi que logiquement, si on ouvre une nouvelle navigation privée, une nouvelle session est créée (*sessionViews* à 1) et on a tout de même changé de serveur.



Nous remarquons que le client renvoie bien 2 cookies au serveur lors de ses requêtes suivantes:

![image-20201122145010128](./img/image-20201122145010128.png)



> 4. Provide a sequence diagram to explain what is happening when one
>    requests the URL for the first time and then refreshes the page. We
>    want to see what is happening with the cookie. We want to see the
>    sequence of messages exchanged (1) between the browser and HAProxy
>    and (2) between HAProxy and the nodes S1 and S2. We also want to see
>    what is happening when a second browser is used.



```mermaid
sequenceDiagram
    participant A as Browser
    participant B as HAProxy
    participant C as S1
    participant D as S2
    
    A->>B: GET /
    B->>C: GET /
    C->>B: Response + Cookie(NODESSID)
    B->>A: Response + Cookie(NODESESSID) + Cookie(SERVERID)
    A->>B: GET / + Cookie(NODESESSID) + Cookie(SERVERID)
    B->>C: ...
```



> 5. Provide a screenshot of JMeter's summary report. Is there a
>    difference with this run and the run of Task 1?
>
>   * Clear the results in JMeter.
>   * Now, update the JMeter script. Go in the HTTP Cookie Manager and
>     <del>uncheck</del><ins>verify that</ins> the box `Clear cookies each iteration?`
>     <ins>is unchecked</ins>.
>   * Go in `Thread Group` and update the `Number of threads`. Set the value to 2.



Sur cette capture, les cookies sont conservés entre chaque requête ce qui a pour conséquence qu'on atteint toujours le même serveur.

![image-20201122143857506](./img/image-20201122143857506.png)



> 6. Provide a screenshot of JMeter's summary report. Give a short
>    explanation of what the load balancer is doing.



On voit que le serveur réagit comme si un autre ordinateur s'était connecté et on obtient le fonctionnement attendu.

![image-20201122144044224](./img/image-20201122144044224.png)





### Task 3: Drain mode



> 1. Take a screenshot of the Step 5 and tell us which node is answering.
>

On peut voir que c'est le serveur s2 qui a répondu. 

![image-20201125152258335](img/image-20201125152258335.png)

> 2. Based on your previous answer, set the node in DRAIN mode. Take a screenshot of the HAProxy state page.

![image-20201125152543145](img/image-20201125152543145.png)



> 3. Refresh your browser and explain what is happening. Tell us if you stay on the same node or not. If yes, why? If no, why?

Oui, nous restons sur la même node. Le principe du mode drain consiste a accepter uniquement les connexions ayant déjà été enregistrée grâce à la persistance. Toute nouvel connexion ne sera pas rediriger vers le node ou ce mode à été activé. 

> 4. Open another browser and open `http://192.168.42.42`. What is happening?

Si nous ouvrons un autre browser par exemple chrome puis firefox, les cookies ne seront pas les mêmes. Nous sommes donc rediriger vers s1 puisqu'il n'est pas en drain mode et accepte donc les nouvelles connexions. 

> 5. Clear the cookies on the new browser and repeat these two steps multiple times. What is happening? Are you reaching the node in DRAIN mode?

Si les cookies sont supprimés toutes les requêtes ouvriront de nouvels sessions, elles seront donc toutes redirigée vers s1 qui n'est pas en drain mode. 

> 6. Reset the node in READY mode. Repeat the three previous steps and explain what is happening. Provide a screenshot of HAProxy's stats page.

Une fois le state ready reactivé c'est à nouveau la politique du round robin avec sticky session qui s'applique. S'il il n'y a pas de session on est redirigé alternativement d'une node à l'autre. Si une session est enregistrée on sera redirigé vers la node concernée.  

> 7. Finally, set the node in MAINT mode. Redo the three same steps and explain what is happening. Provide a screenshot of HAProxy's stats page.

Le mode maint n'accepte plus aucune connexion. Toutes les requêtes seront donc redirigées vers l'autre node. 

### Task 4: Round robin in degraded mode

> 1.  Make sure a delay of 0 milliseconds is set on `s1`. Do a run to have a baseline to compare with in the next experiments.

Après avoir fixé un délai de 0 millisecondes sur chacune des nodes nous lançons un test Jmeter afin d'avoir une mesure de comparaison pour les étapes suivantes. 

![](img/image_2020-11-25_16-01-27.png)

> 2. Set a delay of 250 milliseconds on `s1`. Relaunch a run with the
>    JMeter script and explain what is happening.

Nous avons set le délai avec la commande suivante : 

```
curl -H "Content-Type: application/json" -X POST -d '{"delay": 250}' http://192.168.42.11:3000/delay
```

Le test suivant nous montre que les requêtes sur s1 mettent effectivement plus de temps. 

![](image/7adb8818-fd9c-4c62-8ff3-ad9482b9522d.png)

> 3. Set a delay of 2500 milliseconds on `s1`. Same than previous step.

Nous voyons que presque toutes les requêtes sont redirigées vers s2. 

![](image/image_2020-11-25_16-18-31.png)

> 4. In the two previous steps, are there any errors? Why?

Avec le délai de 250ms sur s1, il n'y a pas d'erreur. 

Avec le délai de 2500ms sur s1, il y a des erreurs. A cause du délai trop grand, HAProxy considère le serveur s1 comme étant down. C'est pour cette raison que toutes les requêtes sont redirigées vers s2

![](image/image_2020-11-25_16-19-24.png)

> 5. Update the HAProxy configuration to add a weight to your nodes. For
>    that, add `weight [1-256]` where the value of weight is between the
>    two values (inclusive). Set `s1` to 2 and `s2` to 1. Redo a run with a 250ms delay.

Nous configurons `hoproxy.cgf` avec les instructions suivantes. 

```
server s1 ${WEBAPP_1_IP}:3000 weight 2 check cookie s1
server s2 ${WEBAPP_2_IP}:3000 weight 1 check cookie s2
```

Nous voyons grâce au test que toutes les requêtes sont redirigées vers s2, ceci à cause du poids plus élevé.  

> 6. Now, what happens when the cookies are cleared between each request and the delay is set to 250ms? We expect just one or two sentence to summarize your observations of the behavior with/without cookies.

Avec les cookies : 

Il n'y a pas des différences car à cause de la sticky sessions toutes les requêtes sont redirigées vers s1. 

![](image/image_2020-11-25_16-38-24.png)

Sans les cookies: 

Une nouvelle sessions est créé à chaque requête. s1 reçoit deux fois plus de requête que s2 car son poids est double. Le temps de réponse est donc légèrement meilleur car s2 a un délai de 0 ms.  

![](image/image_2020-11-25_16-45-22.png)



### Task 5: Balancing strategies



> 1. Briefly explain the strategies you have chosen and why you have chosen them.

La première stratégie qu'on a choisie est la dénommée "first". C'est une stratégie assez simple dans sa logique. On sélectionne le premier serveur avec des emplacements de connexion disponibles et dirigeons toutes les requêtes vers l'un d'eux jusqu'à ce qu'il atteigne son maximum de connexions. La sélection des serveurs se fait par leur id, du plus petit au plus grand. De par sa logique, cette stratégie ignore les poids des serveurs.

C'est une stratégie qui fonctionne mieux avec de longue session comme en RDP ou IMAP. Elle est donc généralement moins efficace en HTTP mais peut quand même être utile.

La raison pour avoir choisie c'est stratégie est l'exemple intéressant donné par HAProxy. En effet, si au première abord cette stratégie paraît plutôt mauvaise elle peut être très utile dans la bonne situation. Durant les périodes avec très peu de requêtes ou d'utilisateurs, plutôt que de distribuer quelques requêtes sur chaque serveur, on peut utiliser la "first" stratégie pour concentrer les requêtes. Associée à d'autres programmes, cela permet d'éteindre un grand nombre de serveurs non utilisés. Cela permet d'économiser de l'énergie et donc de l'argent.

En résumé, l'objectif de cette stratégie est d'utiliser le moins de serveur possible.



La deuxième stratégie qu'on a choisie est la "random". Comme son nom l'indique, on va tirer un nombre aléatoire qui va être utilisé dans la fonction de hachage. Contrairement à la première, cette stratégie utilise et respecte les poids des serveurs.

Cette stratégie est intéressante car elle permet de dynamiquement changer les poids des serveurs et ces changements sont immédiatement pris en compte. C'est aussi une stratégie qui supporte bien le fait d'ajouter ou d'enlever des serveurs. Si on est dans une situation où on va fréquemment changer/ajouter/enlever des serveurs, c'est une bonne alternative à "roundrobin" ou "leastconn" qui support ça moins bien.



> 2. Provide evidence that you have played with the two strategies (configuration done, screenshots, ...)

**First :**

Pour la stratégie, on a changé deux configuration. La première, évidemment, est de changer le paramètre "balance" pour lui dire d'utiliser la stratégie "first". La deuxième a été l'ajout du paramètre "maxconn" sur chaque serveur. Ce paramètre sert à définir le nombre maximum de connexions simultanées que le serveur accepte.

![Task5_first_conf_max1](img/Task5_first_conf_max1.png)

Pour un premier test, on a limité à une connexion par serveur. Comme on a deux utilisateurs simultanés, on a logiquement 1000 connexions sur chaque serveur.

![Task5_first_res_max1](img/Task5_first_res_max1.png)

Dans un deuxième test, on a augmenté les connexions maximums de s1 au nombre de 2. Puisque maintenant le serveur 1 peut s'occuper de toutes les requêtes, elles sont toutes redirigées vers lui.

![Task5_first_res_max2](img/Task5_first_res_max2.png)



**Random :**

Pour la configuration, on a évidemment changé le paramètre balance pour appliquer la stratégie "random". Puisque les poids sont pris en compte, on a commencé en configurant des poids qui devraient répartir les requêtes équitablement.

![Task5_random_conf](img/Task5_random_conf.png)

Avec cette configuration, on obtient des résultats équilibrés entre les serveurs comme attendu.

![Task5_random_res_w21](img/Task5_random_res_w21.png)

Comme deuxième test on a augmenté le poids du "s2" à 6. On devrait donc avoir toute les requêtes vers le serveur 2 et c'est ce qu'on obtient.

![Task5_random_res_w26](img/Task5_random_res_w26.png)



> 3. Compare the two strategies and conclude which is the best for this lab (not necessary the best at all).

Ces deux stratégies sont très différentes. La "first" a pour but de concentrer les requêtes sur un serveur et d'utiliser le moins de serveurs possibles. Au contraire la "random" a plutôt pour but d'utiliser tous les serveurs et de répartir les requêtes en fonction des poids qui leurs sont attribués.

Dans la situation de notre labo, puisqu'on a que deux utilisateurs et des requêtes légères nous pensons que la stratégie "first" est la meilleure. On peut clairement voir dans le deuxième test avec la stratégie "first" qu'un serveur suffit pour s'occuper de toutes les requêtes. On pourrait donc imaginer utiliser la stratégie "first" et éteindre le deuxième serveur jusqu'à ce qu'il soit nécessaire.





### Conclusion

Nous avons beaucoup appris de ce labo car il a mis en lumière des aspects de base, néanmoins très importants, du load balancing tels que le round robin, la sticky session et diférentes stratégies de load balancing. Il nous a aussi permis d'utiliser un outil très utile qu'est JMeter afin de procéder à des tests de charges et aussi d'analyser comment la transmission des cookies et des paquets se fait. 