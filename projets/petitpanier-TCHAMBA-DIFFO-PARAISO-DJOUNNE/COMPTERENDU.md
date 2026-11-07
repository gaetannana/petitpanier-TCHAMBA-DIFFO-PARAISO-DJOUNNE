# Compte rendu TP PetitPanier — TCHAMBA / DIFFO / PARAISO / DJOUNNE

Systèmes utilisés : WSL (Ubuntu) avec Docker Desktop 4.95.0 sur Windows

## Palier 1

1.1 : Le bloc Client est la commande `docker` que je tape (le CLI). Le bloc Server est le démon Docker (Docker Engine 29.8.2, fourni ici par Docker Desktop) qui construit et exécute réellement les conteneurs. Si le bloc Server était absent, le démon ne tournerait pas (par exemple Docker Desktop fermé).

1.2 : Dans `-p 8080:80`, 8080 est le port de ma machine et 80 le port du conteneur. Le format est toujours `port-machine:port-conteneur`. Le navigateur appelle donc `localhost:8080`, et Docker redirige vers le port 80 de nginx dans le conteneur.

1.3 : L'image (`nginx:alpine`) est un modèle figé en lecture seule, le conteneur (`essai`) est une instance en cours d'exécution créée à partir de cette image. Je l'ai vu : après `docker stop`, le conteneur existait encore (statut Exited dans `docker ps -a`), puis après `docker rm` il a disparu, alors que l'image `nginx:alpine` était toujours présente dans `docker image ls`. On peut donc créer et supprimer autant de conteneurs qu'on veut à partir d'une même image.

## Palier 2

2.2 : Non, `EXPOSE 8080` ne suffit pas. Cette instruction documente seulement le port sur lequel l'application écoute dans le conteneur, elle ne l'ouvre pas vers la machine. Pour accéder à la page depuis le navigateur, il faut publier le port au lancement avec l'option `-p` de `docker run` (ou `ports:` dans Compose).

2.3 : Le `.` à la fin de `docker build` est le contexte de build : le dossier courant, envoyé à Docker, qui contient le Dockerfile et `index.html` dont le `COPY` a besoin. On utilise `-p 8080:8080` et non `-p 8080:80` car l'image `nginx-unprivileged` écoute sur le port 8080 dans le conteneur, alors que `nginx:alpine` (palier 1) écoute sur le port 80.

2.4 : Après le build, la page affichait encore la 1.0 parce qu'une image est figée et qu'un conteneur déjà lancé continue de tourner sur l'image avec laquelle il a démarré : construire la 2.0 ne l'a pas modifié. Pour passer à la 2.0, j'ai supprimé l'ancien conteneur avec `docker rm -f petitpanier`, puis j'en ai lancé un nouveau avec `docker run --name petitpanier -d -p 8080:8080 petitpanier:2.0`.

## Palier 3

3.2 : `Up` signifie seulement que le processus du conteneur est démarré. `Up (healthy)` signifie en plus que le healthcheck réussit, donc que le service répond réellement. Cette différence est produite par la section `healthcheck:` de `compose.yaml` : Compose exécute régulièrement la commande `wget` et le statut passe de `health: starting` à `healthy` quand elle réussit. Dans mon cas, le conteneur est resté `Up (unhealthy)` (`FailingStreak` à 31) : `docker inspect` montrait `wget: can't connect to remote host: Connection refused` à chaque sonde, alors que le site répondait dans le navigateur. Rejouée dans le conteneur, `wget http://localhost:8080/` échouait, mais `wget http://127.0.0.1:8080/` affichait la page. J'ai donc remplacé `localhost` par `127.0.0.1` dans le `test` du healthcheck de `compose.yaml` (une seule modification) et le statut est passé à `Up (healthy)`.

3.3 : La seconde commande échoue (`Connection refused`) parce que `localhost` désigne toujours le conteneur lui-même : dans le conteneur `busybox` temporaire, aucun serveur n'écoute sur le port 8080, alors que le site tourne dans un autre conteneur. `app` dans `http://app:8080` est le nom du service défini dans `compose.yaml` : Compose le déclare dans le DNS interne du réseau `petitpanier-tchamba-diffo-paraiso-djounne_default`, qui le résout vers l'adresse du conteneur du site.

## Palier 4

4.1 : Le cluster kind tourne dans un conteneur Docker (`petitpanier-control-plane`) qui possède son propre runtime : il ne voit pas les images stockées dans le Docker de ma machine. `kind load docker-image` copie donc l'image dans le nœud. En entreprise, on pousse l'image dans un registre (GitLab Registry, Docker Hub…) et le cluster la télécharge depuis ce registre. Observation : juste après la création, le nœud était `NotReady`, puis il est passé à `Ready` au bout de quelques minutes.

4.2 : Les labels `app: petitpanier` relient les objets entre eux : le Deployment reconnaît ses Pods grâce à `selector.matchLabels` (qui doit correspondre aux labels du modèle de Pod), et le Service choisit les Pods vers lesquels envoyer le trafic grâce à son `selector`. Si le `selector` du Service était différent, il ne trouverait aucun Pod : la liste des endpoints serait vide et le site inaccessible, même si les Pods sont `Running`.

4.3 : L'adresse affichée dans ENDPOINTS (`10.244.0.5:8080`) est l'adresse IP du Pod (et son port) vers lequel le Service envoie le trafic : le Service l'a trouvée grâce à son `selector` (`app: petitpanier`). On a besoin de `port-forward` car le Service est de type ClusterIP : il n'est accessible que depuis l'intérieur du cluster. `kubectl port-forward` crée un tunnel temporaire entre le port 8080 de ma machine et le Service.

## Palier 5

5.1 : Après `kubectl scale deployment petitpanier --replicas=3`, j'obtiens 3 Pods `Running` et `1/1`. Ils tournent tous sur le même nœud, `petitpanier-control-plane`, car mon cluster kind n'a qu'un seul nœud. On ne peut donc pas parler de haute disponibilité complète : si ce nœud tombe, les 3 Pods tombent avec lui. En revanche, avoir 3 exemplaires protège contre la panne d'un Pod et répartit le trafic entre eux. Avec plusieurs nœuds, Kubernetes pourrait répartir les Pods sur des machines différentes.

5.2 : C'est le Deployment (via son ReplicaSet) qui a recréé le Pod. L'état désiré déclaré dans le Deployment est `replicas: 3`. Quand j'ai supprimé un Pod, l'état réel est passé à 2 Pods : Kubernetes compare en permanence l'état réel à l'état désiré et corrige l'écart, comme un thermostat. Dans `kubectl get pods -w`, j'ai vu le Pod `2tblh` passer en `Terminating`, et un nouveau Pod `vg2nx` apparaître aussitôt en `Pending`, `ContainerCreating`, puis `Running` (1/1) en environ 4 secondes.

5.3 : Dans `kubectl set image deployment/petitpanier web=petitpanier:2.0`, `web` est le nom du conteneur dans le Deployment (`containers[].name` de `deployment.yaml`), pas le nom du Pod ni de l'image : il indique quel conteneur du Pod change d'image. Après `set image`, le navigateur affichait « Version 2.0 » avec la ligne « Nouveau : livraison le dimanche ! », puis la « Version 1.0 » après `rollout undo` (confirmé par `kubectl get deployment … -o jsonpath`, qui renvoie `petitpanier:1.0`). La première fois, `rollout history` affichait 2 révisions (1 et 2), puis 2 et 3 après l'`undo` : le retour en arrière réutilise l'ancien modèle de Pod (image 1.0) et l'enregistre comme une nouvelle révision. Le `port-forward` s'arrêtait à chaque remplacement de Pods (`lost connection to pod`), car il est attaché à un Pod précis : il faut le relancer.

## Palier 6 — tableau de diagnostic

| Panne | Symptôme | Hypothèse | Commande de preuve et résultat | Correction |
|---|---|---|---|---|
| promo-1 | Pod `promo-1-…` en `0/1` `ImagePullBackOff` (précédé de `ErrImagePull`), 0 redémarrage | Le cluster ne trouve pas l'image : tag faux | `kubectl describe pod promo-1-78b84b5b9c-vqnpb` : Events `Failed to pull image "petitpanier:1.O"`. `crictl images` ne contient que `petitpanier:1.0` et `2.0` : le tag `1.O` (lettre O) ne correspond à aucune image | Remplacé `1.O` par `1.0` (chiffre zéro) dans `pannes/panne-1.yaml`, puis `kubectl apply` : le Pod est `Running` et `1/1` |
| promo-2 | Pod `promo-2-…` en `0/1` `CrashLoopBackOff`, `RESTARTS` qui augmente (3 puis 4) | Le conteneur démarre puis s'arrête aussitôt : le processus lancé échoue | `kubectl describe pod promo-2-5fb657dc47-8ln2b` : `Last State: Terminated`, `Exit Code: 1`, démarré et fini à la même seconde. `kubectl logs … --previous` : `ERREUR : variable PROMO_CODE absente` | Supprimé la ligne `command:` de `pannes/panne-2.yaml` (elle affichait l'erreur et faisait `exit 1` sans condition), puis `kubectl apply` : le Pod est `Running` et `1/1` |
| promo-3 | Les 2 Pods `promo-3-…` sont `Running` et `1/1`, mais le site est inaccessible via `port-forward` sur le port 8083 | Le Service ne trouve pas les Pods : son `selector` ne correspond pas aux labels des Pods | `kubectl get endpoints promo-3` : `<none>`. `kubectl describe service promo-3` : `Selector: app=promo-three`, alors que `kubectl get pods --show-labels` montre `app=promo-3` | Remplacé `promo-three` par `promo-3` dans le `selector` du Service (`pannes/panne-3.yaml`), puis `kubectl apply` : 2 adresses dans les endpoints et la page s'ouvre sur le port 8083 |

6.2 (--previous) : `kubectl logs` lit le conteneur actuel. Quand un Pod est en `CrashLoopBackOff`, le conteneur actuel est souvent en attente de redémarrage ou vient de redémarrer, et il peut n'avoir rien écrit : la sortie est alors vide. `--previous` lit les logs de l'instance précédente, celle qui a planté, où se trouve le message d'erreur. Dans mon cas, les deux commandes affichaient `ERREUR : variable PROMO_CODE absente`, car le conteneur écrit ce message puis s'arrête immédiatement ; `--previous` reste le réflexe sûr, puisqu'il cible l'instance qui s'est arrêtée.

6.3 (pourquoi Running mais inaccessible) : `Running` et `1/1` signifient seulement que les conteneurs tournent et sont prêts. Le Service, lui, choisit ses Pods par leur label : son `selector` (`app: promo-three`) ne correspondait pas au label des Pods (`app: promo-3`). Il n'avait donc aucun endpoint (`<none>`) et ne pouvait envoyer le trafic nulle part, même si les Pods étaient sains.

## Palier 7

| Étape | Commande | Si échec |
|---|---|---|
| build | `docker build -t petitpanier:1.0 .` | Pas d'image : le pipeline s'arrête, rien n'est publié ni déployé |
| test | `docker run --rm petitpanier:2.0 nginx -t` | La configuration nginx est invalide : on ne publie pas l'image |
| publish | `kind load docker-image petitpanier:2.0 --name petitpanier` (en entreprise : `docker push` vers un registre) | Le cluster ne peut pas récupérer l'image : le déploiement échouerait en `ImagePullBackOff` |
| deploy | `kubectl set image deployment/petitpanier web=petitpanier:2.0` | Les anciens Pods continuent de servir le site ; on peut revenir en arrière avec `kubectl rollout undo` |

7.1 : Un pipeline tague l'image avec l'identifiant du commit plutôt qu'avec `latest`, car chaque image est ainsi traçable : on sait exactement quel code tourne. Au 5.3, `kubectl set image` a changé le tag de 1.0 à 2.0, ce qui a déclenché un rollout, et `rollout undo` a pu revenir à la 1.0. Avec `latest`, le tag ne change jamais : `set image` ne verrait aucune différence et ne déclencherait rien. Un retour en arrière serait aussi trompeur : même si Kubernetes garde un historique, toutes les révisions pointeraient vers `latest`, qui désigne toujours la dernière image publiée, donc `rollout undo` ne garantirait pas de retrouver l'ancien code.

7.2 (workflow n8n) : Webhook GitLab (déclencheur : GitLab envoie un POST à chaque fin de pipeline) → IF `statut = failed` (condition : `body.object_attributes.status` vaut `failed`) → message Slack dans `#petitpanier-ops` (action : prévient l'équipe DevOps avec le nom du projet, la branche, l'identifiant du commit et le lien du pipeline) ; en parallèle, un e-mail de secours à l'équipe. Si le statut n'est pas `failed`, le workflow s'arrête sans rien envoyer. Les identifiants (token Slack, identifiants e-mail, secret du webhook GitLab) sont rangés dans les identifiants (credentials) de n8n, chiffrés par n8n, jamais en clair dans les nœuds, dans le workflow partagé ni dans Git.

7.3 : J'ai supprimé les objets Kubernetes (`kubectl delete -f k8s/`), puis le cluster (`kind delete cluster --name petitpanier`). `kind get clusters` n'affiche plus `petitpanier` (il reste seulement `m1`, un ancien cluster d'un autre TP que je n'ai pas touché). Les images `petitpanier:1.0` et `petitpanier:2.0` sont conservées, ainsi que tous les fichiers du projet.

## Défis réalisés

- Défi 1 : j'ai lancé deux conteneurs nginx sur les ports 8081 et 8082 de ma machine. Il est impossible de les mettre tous les deux sur 8081, car un port de la machine ne peut être attribué qu'à un seul programme à la fois. Chaque conteneur a son propre réseau, donc les deux peuvent écouter sur le port 80 à l'intérieur ; seul le port côté machine doit être différent.
- Défi 2 : En relançant `docker build -t petitpanier:2.0 .` sans rien modifier, l'étape `[2/2] COPY index.html …` est marquée `CACHED` et le build ne dure que 2,2 s, contre 15,6 s pour le tout premier build (qui téléchargeait l'image de base). Dans ma sortie, l'étape `[1/2] FROM` n'est pas marquée `CACHED` mais ne prend que 0,1 s. Docker réutilise une couche tant que l'instruction et les fichiers qu'elle utilise sont identiques ; quand j'avais modifié `index.html` pour la version 2.0, seule l'étape `COPY` avait été refaite.
- Défi 3 : j'ai ajouté un service `stats` (image `nginx:alpine`, ports `8082:80`) à `compose.yaml`. La commande `docker compose exec stats wget -qO- http://app:8080` affiche la page de PetitPanier : `stats` joint `app` par son nom de service, grâce au DNS interne du réseau Compose.
- Défi 4 : Dans les Events du Pod : `Scheduled` (le scheduler choisit le nœud), `Pulled … already present on machine` (l'image est déjà dans le nœud grâce à `kind load` et à `imagePullPolicy: IfNotPresent`, aucun téléchargement), `Created` (le conteneur est créé), `Started` (il démarre). Un avertissement `Readiness probe failed … connection refused` apparaît juste après `Started` : nginx n'écoutait pas encore, puis le Pod est passé `Ready`. Tant que la sonde échoue, le Service n'envoie pas de trafic au Pod.
- Défi 5 : J'ai modifié `replicas` dans `k8s/deployment.yaml` puis fait `kubectl apply -f k8s/` : le cluster est passé à 4 Pods. Une équipe préfère modifier le fichier car il est la source de vérité : versionné dans Git (historique, relecture, retour en arrière), reproductible et applicable par un pipeline. `kubectl scale` ne change que le cluster en cours : le fichier resterait à l'ancienne valeur, et le prochain `kubectl apply` annulerait le changement. L'avertissement de `rollout undo` va dans le même sens : `set image`, `scale` et `undo` modifient le cluster sans mettre à jour l'annotation `last-applied-configuration` utilisée par `apply`, ce qui peut donner des résultats inattendus.
- Défi 6 : En remplaçant le `path` de la `readinessProbe` par `/sante`, le nouveau Pod `petitpanier-…` est resté `Running` mais `0/1`, alors que les 3 anciens Pods restaient `1/1` et continuaient de servir le site : Kubernetes ne retire pas les anciens Pods tant que le nouveau n'est pas prêt. Le `describe pod` montre `Readiness: http-get http://:8080/sante`, et dans les Events une première erreur `connection refused` (nginx démarrait) puis, répétée, `HTTP probe failed with statuscode: 404` : nginx répond mais la page `/sante` n'existe pas. `kubectl rollout status` a fini en `timed out waiting for the condition`. Après avoir remis `/` et refait `kubectl apply`, le Pod en échec a disparu et les Pods sont revenus à `1/1`.
- Défi 7 : j'ai écrit `.gitlab-ci.yml` avec 4 jobs (build, test, publish, deploy) qui reprennent les commandes tapées pendant le TP. L'image est taguée avec le SHA du commit, les identifiants du registre viennent des variables protégées de GitLab, et le déploiement est manuel et limité à la branche `main`. Ce fichier ne s'exécute que sur GitLab : sur GitHub il sert uniquement de livrable.
