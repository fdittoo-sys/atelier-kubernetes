Rendu — Atelier Kubernetes
Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

Nom / Prénom : DITTOO Farhan
Groupe : RS1

---
## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > Il y a 3 nœuds, tous en `Ready`. `k3d-lab-server-0` a le rôle `control-plane`, c'est le master : c'est lui qui gère le cluster et décide où lancer les pods. Les deux autres, `k3d-lab-agent-0` et `k3d-lab-agent-1`, n'ont pas de rôle affiché (`<none>`) : ce sont les workers, ceux qui font vraiment tourner les applis.
   > J'ai aussi remarqué qu'avec `kubectl get pods -A`, le cluster n'est pas vraiment vide : il y a déjà des pods système dans le namespace `kube-system` (coredns pour le DNS, traefik, metrics-server, etc.).

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : un pour chaque nœud (le server et les deux agents, avec l'image `rancher/k3s`), plus un `k3d-lab-serverlb` qui sert de load balancer, mais qui n'est pas un nœud Kubernetes.
   > Concrètement, dans k3d, un nœud c'est juste un conteneur Docker dans lequel tourne k3s. Au lieu d'avoir 3 vraies machines, on simule tout dans le Codespace : les nœuds partagent le même noyau et communiquent par le réseau Docker `k3d-lab`. Et les pods qu'on lance tournent à l'intérieur de ces conteneurs, donc ce sont des conteneurs dans des conteneurs.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > La commande a créé un Deployment `web`, qui a lui-même créé un ReplicaSet `web-68d995574f`, qui a créé un pod `web-68d995574f-bj4w6`. On voit bien la chaîne avec les noms : le pod reprend le nom du ReplicaSet + un suffixe aléatoire. Le Deployment gère le ReplicaSet, et le ReplicaSet s'assure qu'il y a toujours 1 pod qui tourne (DESIRED 1 / CURRENT 1).
   > Le Service `web` n'est pas créé par `create deployment` mais par `kubectl expose` : il donne une adresse fixe pour joindre le pod depuis l'intérieur du cluster.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Dans le second terminal (`kubectl get pods -w`), j'ai vu l'ancien pod `web-68d995574f-bj4w6` passer en `Terminating` puis `Completed`. Dans la même seconde, un nouveau pod `web-68d995574f-sp82c` est apparu : `Pending`, puis `ContainerCreating`, puis `Running` en 2 secondes à peine.
   > Le nom a changé (`bj4w6` → `sp82c`), mais le début `web-68d995574f` est le même, parce que c'est toujours le même ReplicaSet qui l'a créé. Le ReplicaSet doit garder 1 replica en vie : dès qu'il a vu que le pod disparaissait, il en a recréé un nouveau. Le pod n'est pas redémarré, il est remplacé par un tout neuf. C'est l'auto-réparation de Kubernetes : on ne gère pas les pods un par un, on déclare un état voulu (1 replica) et Kubernetes s'arrange pour le maintenir.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non. Après avoir relancé le tunnel, je suis retombé sur la page par défaut « Welcome to nginx! », mon message avait disparu.
   > C'est parce que j'avais modifié le fichier directement dans le conteneur du pod `sp82c`. Quand le pod a été supprimé, son système de fichiers a été supprimé avec lui. Le nouveau pod créé par le ReplicaSet repart de l'image `nginx` d'origine, qui ne contient évidemment pas ma modif. Les conteneurs sont éphémères : tout ce qu'on change à la main dedans est perdu. Pour que le contenu persiste, il faut le mettre en dehors du conteneur (ConfigMap, volume…) ou dans l'image elle-même : c'est le but de l'exercice 4.

6. Pourquoi le port-forward a-t-il cessé de fonctionner après la suppression du pod ?
   > Quand j'ai rechargé la page, j'ai eu une erreur HTTP 502 : le proxy GitHub n'arrivait plus à joindre le port 8080. Dans `/tmp/web.log`, on voit que le tunnel marchait (`Handling connection for 8080`), puis l'erreur `network namespace for sandbox ... is closed` et `error: lost connection to pod`, et le processus s'est arrêté (`Exit 1`).
   > En fait, le `port-forward` ne passe pas vraiment par le Service : au lancement, il s'accroche à un seul pod précis. Quand ce pod a été supprimé, le tunnel s'est retrouvé sans destination et il est mort avec lui. C'est pour ça qu'il faut le relancer à chaque fois, et que ce n'est pas une solution pour la prod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est impératif : on donne un ordre précis (« crée ce Deployment »). Si l'objet existe déjà, la commande échoue (`AlreadyExists`). Et rien n'est gardé dans un fichier, donc c'est difficile à refaire à l'identique ou à versionner.
   > `kubectl apply -f` est déclaratif : on décrit l'état voulu dans un fichier YAML, et Kubernetes se débrouille pour y arriver. Si l'objet n'existe pas, il le crée ; s'il existe, il compare et ne modifie que ce qui a changé. Je l'ai vu au 4.4 : après avoir changé la ConfigMap, `apply` a affiché `configmap/site-html configured` mais `deployment.apps/site unchanged` et `service/site unchanged`. On peut le relancer autant de fois qu'on veut, le résultat est le même. En plus, les fichiers sont versionnés dans Git, donc on a un historique et le dépôt sert de référence (GitOps).

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` dit au Service vers quels pods envoyer le trafic : ici `app: site`, donc tous les pods qui ont ce label. Le Service ne connaît pas les pods par leur nom (qui change à chaque recréation, comme on l'a vu), il les retrouve par leurs labels. C'est pour ça que l'adresse du Service reste stable même quand les pods sont remplacés.
   > Si le selector ne correspond à aucun pod (par exemple une faute de frappe `app: sit`), le Service est quand même créé sans erreur, mais il n'a aucun endpoint : `kubectl get endpoints site` est vide. Le trafic n'arrive nulle part, et le site ne répond pas. C'est un piège classique parce que rien ne plante, il faut penser à vérifier les endpoints (chez moi : `10.42.1.8:80`, l'IP du pod).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
   site-df64cb7b8-cxhfg   1/1     Running   0          6s      10.42.2.5   k3d-lab-agent-0    <none>           <none>
   site-df64cb7b8-hv6r5   1/1     Running   0          5m43s   10.42.1.8   k3d-lab-server-0   <none>           <none>
   site-df64cb7b8-jkb6k   1/1     Running   0          6s      10.42.0.7   k3d-lab-agent-1    <none>           <none>
   ```
   > Les 3 réplicas sont répartis sur les 3 nœuds, un par nœud. Le scheduler a mis les 2 nouveaux pods sur les agents qui étaient vides pour équilibrer la charge. On remarque aussi qu'un pod tourne sur `k3d-lab-server-0` (le control plane) : avec k3s, le server n'a pas de taint `NoSchedule`, il accepte donc les pods comme un worker. Chaque nœud a sa propre plage d'IP de pods (10.42.0.x, 10.42.1.x, 10.42.2.x).

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod `site-68856667bc-5bhbn` est passé en `ErrImagePull`, puis en `ImagePullBackOff` (0/1, jamais Ready). C'est `kubectl describe pod site-68856667bc-5bhbn`, dans la section Events en bas, qui donne la cause exacte : `Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found`. Le tag 9.99 n'existe tout simplement pas sur Docker Hub.
    > On voit aussi que le pod a bien été placé sur `k3d-lab-agent-1` (le scheduling a marché), c'est le téléchargement de l'image par le kubelet qui échoue. Après 3 échecs, Kubernetes passe en BackOff : il réessaie mais en espaçant de plus en plus.

11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui, le site restait accessible. Les 3 anciens pods en nginx:1.28 sont restés `Running` pendant toute la panne.
    > C'est grâce à la stratégie de rolling update. Par défaut, `maxSurge: 25%` autorise 1 pod en plus des 3, donc Kubernetes a créé un seul nouveau pod en 9.99. Et `maxUnavailable: 25%` (arrondi à 0 avec 3 replicas) l'empêche de supprimer un ancien pod tant que le nouveau n'est pas Ready. Comme le nouveau ne devient jamais Ready, le rollout reste bloqué, mais les anciens pods continuent de servir le trafic. Le Service n'envoie d'ailleurs du trafic qu'aux pods Ready, donc jamais au pod cassé. Après `kubectl rollout undo`, `kubectl diff -f k8s/site/` n'affichait plus rien : le cluster était de nouveau aligné avec mes fichiers YAML.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://psychic-spoon-rr496477qpwfpw4p-8585.app.github.dev/

13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui, `kubectl get deploy mario -o yaml` affiche bien le Deployment créé depuis Portainer, dans le namespace `default`, à côté de `site` et `web`. Portainer et `kubectl` parlent tous les deux au même composant : l'API Server de Kubernetes (sur le control plane `k3d-lab-server-0`). Portainer n'est qu'une interface graphique qui envoie des requêtes à cette API, comme kubectl.
    > Le Deployment est presque identique à ceux que j'ai créés en ligne de commande : même structure, même stratégie RollingUpdate (maxSurge/maxUnavailable 25 %). Les seules différences sont des labels ajoutés par Portainer (`io.portainer.kubernetes.application.owner: admin`, `stackid`, etc.) pour qu'il retrouve ses applications, et l'absence de l'annotation `last-applied-configuration` puisque Portainer n'utilise pas `kubectl apply`. `imagePullPolicy: Always` vient juste du tag `:latest`.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **Bonus A — Ingress :** j'ai complété `k8s/bonus/ingress.yaml` (Service `site`, port 80). k3s l'a automatiquement associé à Traefik (`CLASS traefik`, `HOSTS *`). Avec `curl localhost:8081`, le site répond sans aucun port-forward : le trafic passe par le load balancer k3d, puis Traefik, puis le Service `site`, qui répartit sur les 3 pods. Quand j'ai supprimé les pods avec `kubectl delete pod -l app=site`, le site est resté accessible sur le 8081, alors qu'avec le port-forward le tunnel cassait à chaque fois. C'est parce que l'Ingress passe par le Service et pas par un pod précis.
    >
    > **Bonus B — Probes et ressources :** j'ai ajouté une `readinessProbe` HTTP sur `/` (port 80), et des `resources` (requests 50m CPU / 32Mi, limits 100m CPU / 64Mi). Pour tester, j'ai pointé la probe vers `/nope` avec un `kubectl patch` : le nouveau pod est resté en `0/1 Running`. Le conteneur tourne mais n'est jamais Ready, car nginx renvoie 404. Du coup, le rollout est resté bloqué et les 3 anciens pods ont continué à servir le site, comme avec `nginx:9.99`. Un `kubectl apply -f k8s/site/` a tout remis en ordre. La différence avec l'image cassée, c'est qu'ici le conteneur démarre bien : c'est l'appli qui n'est pas prête, et c'est la probe qui le détecte.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
