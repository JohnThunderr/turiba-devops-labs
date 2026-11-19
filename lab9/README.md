# Lab 9 · Package it with Helm

GitHub user: JohnThunderr · Fallback image used: no · Fallback Mission 8 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password. -->

## 1 · The chart

```text
$ helm lint charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
  replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/johnthunderr/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- 🔮 Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted that only the API Deployment's `replicas:` line changes to 2 (the database stays at 1 and the images stay the same), and that without `--set` the number comes from `values.yaml`, because `--set` overrides the value that `values.yaml` supplies. What happened: exactly that. Without `--set` the API says `replicas: 3`.
- In my own words, why the Namespace and the Secret are **not** in the chart: Helm only manages objects it made itself and refuses to take over ones that already exist, and the Namespace has to exist before the release because `helm install -n course` puts everything into it. The Secret holds a password, and anything in the chart is a file in Git, so it is created with a command instead. Also, `helm uninstall` would delete anything the chart made, including a Namespace or Secret that other things use.

My first PR for the chart move only carried the `git mv` renames: a failed `git add` stopped my script halfway, so `Chart.yaml` and the removal of the `namespace:` lines were missing. I noticed in the output, and a second small PR finished it before I did anything else.

## 2 · Install, upgrade, rollback

```text
$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Fri Nov 20 00:09:40 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Fri Nov 20 00:10:48 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-5f7d9c974b-7552b   0/1     ErrImagePull   0          35s
api-5fcf87d647-kpsqf   1/1     Running        0          101s
api-5fcf87d647-n8rdm   1/1     Running        0          101s
api-5fcf87d647-qshw5   1/1     Running        0          101s

$ helm history course-app -n course            (before the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Fri Nov 20 00:09:40 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Fri Nov 20 00:10:48 2026	deployed  	course-app-0.1.0	0.2.0      	Upgrade complete

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ helm history course-app -n course            (after the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Fri Nov 20 00:09:40 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Fri Nov 20 00:10:48 2026	superseded	course-app-0.1.0	0.2.0      	Upgrade complete
3       	Fri Nov 20 00:11:25 2026	deployed  	course-app-0.1.0	0.2.0      	Rollback to 1   
```

- 🔮 Before the upgrade with a tag that doesn't exist: will `helm upgrade` say it failed? Will `GET /api/todos` still answer? I predicted that `helm upgrade` would say it succeeded and that the API would still answer, because Helm only reports that the cluster accepted the change and does not wait for the new Pod, and a rolling update keeps the old Pods until the new one is ready. What happened: `STATUS: deployed`, the new Pod stuck in `ErrImagePull`, three old Pods still `Running`, and `GET /api/todos` still returned my todo.
- 🔮 Before the rollback: which revision number will `helm rollback course-app 1` create? I predicted revision 3, because a rollback never rewrites history; it creates a new revision with the old settings. What happened: revision 3, described `Rollback to 1`. `helm get values` then showed `USER-SUPPLIED VALUES: null`, so the bad tag was no longer remembered.
- 🔮 Before `helm uninstall` and a new `helm install`: will your todo still be there? I predicted no, because the PVC `db-data` belongs to the release and is deleted with it, while the Secret stays. What happened: after `helm uninstall` the PVC went `Terminating` and the Secret stayed. I waited until the PVC was really gone, installed again, and the list was empty.

```text
$ curl -s localhost:8087/api/todos             (after uninstall and install)
[]
```

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: Helm would stop while rendering the templates, before anything reached the cluster, because the commit touched a template's value names. The other faults would only show up one at a time after that.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | `helm upgrade` failed at once, nothing changed in the cluster | `helm upgrade course-app charts/course-app -n course`: `Error: UPGRADE FAILED: course-app/templates/api-deployment.yaml:23:28 … at <.Values.api.images.repository>: nil pointer evaluating interface {}.repository`. `values.yaml` has `image`, not `images` | `fix(chart): read api.image.repository again` |
| 2 | After fix 1, `helm upgrade` failed again, this time against the cluster | `helm upgrade …`: `Namespace "course" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata`. The branch added `templates/namespace.yaml`, but I made the Namespace with `kubectl` | `fix(chart): remove the Namespace template, kubectl owns it` |
| 3 | After fix 2, `helm upgrade` said `deployed` (revision 2), but the API had one Pod | `kubectl -n course get deployment api`: `1/1`, and `helm template course-app charts/course-app -n course \| grep replicas:` printed `replicas: ` with nothing after it. `values.yaml` had been renamed to `replicaCount`, so the template's `api.replicas` was empty and Kubernetes used its default of 1 | `fix(chart): name the value api.replicas again` |

After the third fix, `helm lint` passed, `helm upgrade` made revision 3, `kubectl -n course get deployment api` showed `3/3`, and `GET /api/todos` answered through the port-forward.
