# Lab 7 · Run it on a cluster

GitHub user: JohnThunderr · Fallback image used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. -->

## 1 · Deploy

- 🔮 Before my first `apply`: how many Pods, and what their names start with? I predicted 3 Pods, all named `api-` followed by the ReplicaSet hash and a random suffix, because the Deployment asks for `replicas: 3` and a Deployment creates a ReplicaSet named `<deployment>-<hash>`, which names its Pods. What happened: exactly that, 3 of 3 ready, and the ReplicaSet was `api-6999c4877f`.

```text
$ kubectl -n course get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           68s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/api-6999c4877f   3         3         3       68s

NAME                       READY   STATUS    RESTARTS   AGE
pod/api-6999c4877f-fbpfx   1/1     Running   0          68s
pod/api-6999c4877f-lb98k   1/1     Running   0          68s
pod/api-6999c4877f-wzqcc   1/1     Running   0          68s

$ curl -s localhost:8080/
{"name":"course-api","version":"0.1.0"}
```

## 2 · Heal and scale

- 🔮 After deleting one Pod, two seconds later I expected to see a new Pod already starting, so still 3 Pods, because the ReplicaSet keeps comparing desired (3) with actual and creates a replacement at once. What happened: the deleted Pod `api-6999c4877f-fbpfx` was gone and a new one, `api-6999c4877f-kwft9`, was already `Running` at age 3s.

```text
$ kubectl -n course get pods        (after the delete)
NAME                   READY   STATUS    RESTARTS   AGE
api-6999c4877f-kwft9   1/1     Running   0          3s
api-6999c4877f-lb98k   1/1     Running   0          110s
api-6999c4877f-wzqcc   1/1     Running   0          110s

$ kubectl -n course get pods        (with replicas: 5)
NAME                   READY   STATUS    RESTARTS   AGE
api-6999c4877f-fbpfx   1/1     Running   0          92s
api-6999c4877f-glkfx   1/1     Running   0          7s
api-6999c4877f-lb98k   1/1     Running   0          92s
api-6999c4877f-mjb97   1/1     Running   0          7s
api-6999c4877f-wzqcc   1/1     Running   0          92s
```

I set `replicas` back to 3 and applied again; `git diff` was empty and the 5 was never committed.

## 3 · Roll and roll back

- 3a: I rolled from the tag `0.2.0` to the tag `8deeee9383398af15dd13212cf85cb9d69fb034b` (the SHA of the newest `main` commit, built by the green `ci` run on `main`), through a PR.

```text
$ kubectl -n course rollout status deployment/api
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
deployment "api" successfully rolled out
```

- 3b: 🔮 With the tag 9.9.9, I predicted that 3 old Pods keep running, because a rolling update by default may not take any Pod away (maxUnavailable rounds down to 0 for 3 replicas) and may add only one extra (maxSurge 25% rounds up to 1), so the single new Pod stalls and the old ones stay. What happened: all 3 old Pods stayed `Running`, 1 new Pod was in `ErrImagePull`, and `rollout status` timed out after 30s.

```text
$ kubectl -n course get pods        (during the stall)
NAME                   READY   STATUS         RESTARTS   AGE
api-576d499c9b-q2r7x   0/1     ErrImagePull   0          33s
api-785668bc44-5rh8d   1/1     Running        0          45s
api-785668bc44-rqsq6   1/1     Running        0          43s
api-785668bc44-v4qss   1/1     Running        0          51s

$ kubectl -n course rollout undo deployment/api
Warning: resource deployments/api was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/api rolled back
```

- What the `rollout undo` warning means, in my own words, and what I did so that Git and the cluster agree again: `kubectl apply` stores the applied manifest in an annotation (`last-applied-configuration`) and uses it to work out what changed next time. `rollout undo` changes the live Deployment but does not update that annotation, so the next `apply` could compare against stale data. The cluster was then back on the old image while my file in Git was never changed. I ran `git restore k8s/` to throw away the 9.9.9 edit and `kubectl apply -f k8s/` again, so that the file, the annotation and the cluster all say the same thing. `rollout history` showed revisions 1, 3 and 4.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: the Pods, because a bad image tag stops the rollout (ImagePullBackOff) before anyone looks at the Service, while the Service mistakes would show up only as a service that answers nothing.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | A new Pod stuck in `ImagePullBackOff`, old Pods still running | `kubectl -n course describe pod api-7b859b8fd4-wjcg8`: `Failed to pull image "ghcr.io/johnthunderr/turiba-devops-labs/api:0.2.O": ... not found` (the tag has the letter O) | `919b4f2` fix(k8s): correct the image tag 0.2.O to 0.2.0 |
| 2 | Pods all ready, but the Service had no backends and `port-forward` gave nothing | `kubectl -n course get endpointslices -l kubernetes.io/service-name=api`: `api-rfxpc   IPv4   <unset>   <unset>` (ENDPOINTS empty); the Service selector was `app.kubernetes.io/name: api` while Pods are labelled `app: api` | `a87254f` fix(k8s): select the pods by app: api |
| 3 | Endpoints existed, but `curl` through the port-forward returned nothing (empty reply) | `kubectl -n course get endpointslices ...` showed `PORTS 8080`, and port-forward printed `Forwarding from 127.0.0.1:8081 -> 8080`, but the API listens on 5000 | `a515832` fix(k8s): point the service at port 5000 |

After the third fix, `kubectl apply -f k8s/` from `main` gave 3/3 ready Pods and `curl` through the port-forward answered `{"name":"course-api","version":"0.1.0"}`.
