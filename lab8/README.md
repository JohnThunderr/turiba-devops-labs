# Lab 8 · Run the whole app

GitHub user: JohnThunderr · Fallback image used: no · Fallback Mission 7 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password, not even the base64 form. -->

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted yes because the data folder `/var/lib/postgresql/data` is on the PVC `db-data`, not in the container's own filesystem, so a new Pod mounts the same volume and finds the same data. What happened: the todo was still listed after the new Pod started.

```text
$ kubectl -n course get pvc,pv
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-346e9297-38f2-4358-b930-d0511d344a3a   1Gi        RWO            standard       <unset>                 8m49s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-346e9297-38f2-4358-b930-d0511d344a3a   1Gi        RWO            Delete           Bound    course/db-data   standard       <unset>                          8m22s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8084/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T18:23:47.113Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-654c6df4b8-7scnt   1/1     Running   0          26s

$ kubectl -n course delete pod db-654c6df4b8-7scnt
pod "db-654c6df4b8-7scnt" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-654c6df4b8-qxlw7   1/1     Running   0          11s

$ curl -s localhost:8084/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T18:23:47.113Z"}]
```

The PVC was `Pending` right after the first `apply` (age 0s) and `Bound` as soon as the `db` Pod started: kind binds the volume only when a Pod uses it.

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
course-api 0.1.0 starting (Node.js v24.21.0)
API listening on port 5000
Database ready
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted I would not see it as readable text, only a base64 string, because a Secret stores its values base64-encoded. What I saw, in words (don't paste the value): the Secret has one key called `password` and its value is an encoded string, not the readable word. I only counted the key and did not print the value. Base64 is encoding, not encryption, so anyone who can read the Secret can decode it; that is why the Secret is created by a command and is never a file in Git.

My first version of the init container had `-h "" -p ""` instead of `$DB_HOST` and `$DB_PORT`, because my shell expanded the variables when I wrote the file. The init container then waited for ever (`Init:0/1`, log `: - no response`). I rewrote the file so the YAML contains the literal `$DB_HOST` and `$DB_PORT`.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted that they still get traffic and are restarted over and over, because traffic depends on the readiness probe (still `/healthz`, so the Pods stay ready), while a failing liveness probe makes the kubelet kill the container. What happened: the Pods showed `1/1 Running` but with growing `RESTARTS`, as predicted.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                   READY   STATUS    RESTARTS     AGE
api-5574bf5cc9-2pztj   1/1     Running   2 (5s ago)   76s
api-5574bf5cc9-4cgtb   1/1     Running   2 (3s ago)   73s
api-5574bf5cc9-r2l2q   1/1     Running   2 (1s ago)   71s

$ kubectl -n course describe pod api-5574bf5cc9-2pztj         (only the Events)
  Normal   Started    80s                kubelet            spec.initContainers{wait-for-db}: Started container wait-for-db
  Normal   Started    10s (x3 over 79s)  kubelet            spec.containers{api}: Started container api
  Normal   Killing    10s (x2 over 40s)  kubelet            spec.containers{api}: Container api failed liveness probe, will be restarted
  Warning  Unhealthy  0s (x7 over 60s)   kubelet            spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

- How I made Git and the cluster agree again afterwards: I ran `git restore k8s/` to drop the uncommitted `/nope` edit and then `kubectl apply -f k8s/`. The rollout finished and `git diff` was empty.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: the new API Pods would not become ready, while the three old Pods from `main` kept serving, because a rolling update does not remove an old Pod until a new one is ready. Faults would show one at a time, each hiding the next.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | A new API Pod stuck in `Init:0/1` | `kubectl -n course logs api-84db49b8cc-mh9p4 -c wait-for-db`: `database:5432 - no response`. The ConfigMap had `DB_HOST: database`, and no Service has that name | fix(k8s): DB_HOST names the Service db again (then `rollout restart deployment/api`, so the Pod reads the new value) |
| 2 | After fix 1 the new Pod showed `CreateContainerConfigError` | `kubectl -n course describe pod api-777c67d968-95w4f`: `Error: couldn't find key pasword in Secret course/db-secret` | fix(k8s): read the Secret key password |
| 3 | After fix 2 the new Pod was `0/1 Running` and never ready | `kubectl -n course describe pod api-6cc788bd47-fbz9w`: `Readiness probe failed: Get "http://10.244.0.18:5001/healthz": dial tcp 10.244.0.18:5001: connect: connection refused` | fix(k8s): readiness probe checks port 5000 |

After the third fix every Pod was `1/1 Running` and `GET /api/todos` answered through the port-forward.
