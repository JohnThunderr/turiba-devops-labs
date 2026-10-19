# Lab 4 · Multi-service app with Compose

GitHub user: <johnthunderr> · Reference files used (Lab 2 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Fill in every blank (four underscores) and your GitHub user name: check.sh writeup flags anything left. -->

## 1 · Architecture

```mermaid
flowchart LR
  B["Browser on your laptop"]
  subgraph front["network: front"]
    W["web: nginx :80"]
  end
  A["api :5000 · networks: front, back"]
  subgraph back["network: back"]
    D[("db :5432")]
  end
  V[("volume pgdata")]
  B -->|"HTTP localhost:8080"| W
  W -->|"HTTP api:5000"| A
  A -->|"PostgreSQL db:5432"| D
  D --- V
```

Which arrow crosses from your laptop into Docker: B --> W

## 2 · Predictions and what happened

Part 1: fill in "I predicted" (the letter and a one-line reason) before you write any YAML. Part 3: fill in the rest. A wrong prediction that you explain earns full marks.

| # | I predicted | What I saw (command → the deciding output line) | Match? | If not: why |
|---|-------------|--------------------------------------------------|--------|-------------|
| P1 | B - web does not see the DB, since db sits in back, but web can only see front | thunder@BSW:~/turiba-devops-labs$ docker compose exec web getent hosts db -> No output | Yes | |
| P2 | B - while port 5432 is used by PostgreSQL, it is used within the container, and by sending request from outside the container, the request gets refused by OS| Db can be reached from web, but refused if accessed directly | Yes | |
| P3 | A - down only removes the database, not the volume where todo sits | after docker up, I got the message 'survived' | Yes | |
| P4 | C - the same as P3, but in this case -v removes the volume also | docker compose up -d --wait
curl -s localhost:8080/api/todos ERROR:  relation "todos" does not exist, SO yes, it deletes both db and volumes and nothing comes back up | Yes | |
| P5 | A - health check only tests /healthz, which doesn’t use the database, so it stays healthy, the stopped db fails inside the API, which answers 500.| I see 2 services at the beginning app-api and app-web, and afterwards just app-api with status healthy. | Yes | |
| P6 | B - app.js runs in browser not in Docker; the laptop isn’t on any Docker network, so it can’t resolve the name api. The lookup fails before CORS could even apply.| even web can’t resolve names outside its networks; my browser runs app.js on the laptop, which is on no Docker network, so api can’t be resolved | Yes | |

## 3 · Backup and restore

```text
(paste the whole output of step 3.3: pg_dump, down -v, the count query, the restore, and the GET with your todo back)
```

Why the restore must run before `api` starts: ____

## 4 · Compose clinic

### c1 (worked example)
- Error line (copied): `connect ECONNREFUSED 127.0.0.1:5432`
- Checkpoint: port, inside `api`
- Fix: `DB_HOST: db`
- Why this fix, and not the tempting one: inside `api`, `localhost` is `api` itself, and nothing listens on 5432 there. Docker's DNS answers `db` with the database container's address. Publishing `db`'s port would "work" by opening a door into the database.
- Hints used: 0

<!-- Copy the block below once for every mission you solved and change cN to c2, c3 … Then delete the cN block itself. -->

### cN
- Committed fix: `lab4/clinic/cN/`
- Error line (copied): ____
- Checkpoint (name / path / port / app / config / data): ____
- Fix: ____
- Why this fix, and not the tempting one: ____
- Hints used: ____

## 5 · Run in one command

```bash
git clone https://github.com/<you>/turiba-devops-labs.git && cd turiba-devops-labs
____
____
```

## 6 · Answers

### A1
In c1 the API logged `ECONNREFUSED 127.0.0.1:5432`. Which container was `127.0.0.1`, and why does `db` work instead?

____

### A2
Trace `POST /api/todos` from your browser to PostgreSQL: each hop, with the name and port used. Why does `db` have no `ports:`?

____

### A3
`docker compose down` kept your todos, and `docker compose down -v` deleted them. Where were they stored, and what exactly does `-v` remove?

____

### A4
With `db` stopped, `docker compose ps` showed `api` as healthy, but `/api/todos` answered 500. Explain both. What does `service_healthy` protect against, and what not?

____
