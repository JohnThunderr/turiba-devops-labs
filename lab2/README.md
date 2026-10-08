# Lab 2 · Containerize it, then debug it

- **Image:** `ghcr.io/johnthunderr/course-api:lab2` (public, linux/amd64 + linux/arm64)
- **Base image digest:** `node:24-alpine@sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1`

## Part 1 · Images and layers

| Image | Size | Distro | Default user |
|-------|-----:|--------|--------------|
| node:24        | 1.66Gb | Debian GNU/Linux 12 (bookworm) | root -> User=|
| node:24-slim   | 242Mb  | Debian GNU/Linux 12 (bookworm) | root -> User= |
| node:24-alpine | 332MB  | Alpine Linux v3.24             | root -> uid=0(root) |

cow:bad = 65.5 MB · cow:good = 12.9 MB

| Build | RUN step CACHED? | Build time |
|-------|------------------|-----------:|
| b · app.txt changed         | Yes | 3.8s for the whole build and about 0.2s for the RUN |
| c · deps.txt changed        | No | 19s for the whole build and about 15.2s for the RUN |
| d · app.txt changed, wrong order | No | 19s for the whole build and about 15.2s for the RUN |

## Part 2 · The course API image


| Step | Image | Size |
|------|-------|-----:|
| Naive | course-api:naive | 1.75 Gb |
| + .dockerignore, npm ci --omit=dev, exec form | course-api:step1 | 1.65 Gb |
| Multi-stage, node:24-alpine, non-root | course-api:lab2 | 248 Mb |
| Reduction against naive | | ~85 % |

docker stop before the SIGTERM handler: 10.4 s, exit code 137
                                 after:  3.0 s, exit code 0

Output of `docker run --rm course-api:lab2 id`: uid=1000(node) gid=1000(node) groups=1000(node),1000(node)

Output of `docker ps` showing (healthy): Yes -> Up 43 seconds (healthy)

## Part 3 · Linux drills

    3.1 
        root@6d356cbcb2c9:/# cat /etc/os-release
        PRETTY_NAME="Ubuntu 24.04.5 LTS"
        NAME="Ubuntu"
        VERSION_ID="24.04"
        VERSION="24.04.5 LTS (Noble Numbat)"
        VERSION_CODENAME=noble
        ID=ubuntu
        ID_LIKE=debian
        HOME_URL="https://www.ubuntu.com/"
        SUPPORT_URL="https://help.ubuntu.com/"
        BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
        PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
        UBUNTU_CODENAME=noble
        LOGO=ubuntu-logo
        root@6d356cbcb2c9:/# uname -r
        6.18.40.1-microsoft-standard-WSL2

        Cat is a command that reads the existing file in containers filesystem, where uname is a command that directly asks the running kernel.


    3.2 
        root@6d356cbcb2c9:/# sed 's/tech/tools/g' /lab/devopstools > /lab/newtools.txt
        cat /lab/newtools.txt
        chef tools
        ansible tools
        docker tools

    3.3 
        ji@LVAGPLTP5174:/mnt/c/WINDOWS/system32$ docker logs web 2>/dev/null | grep -c '" 404 '
        20
    
    3.4
        root@6d356cbcb2c9:/# su - student -c 'cat /lab/f'
        cat: /lab/f: Permission denied
        
        root@6d356cbcb2c9:/# ls -l
        total 72
        lrwxrwxrwx   1 root root    7 Apr 22  2024 bin -> usr/bin
        drwxr-xr-x   2 root root 4096 Apr 15 06:53 bin.usr-is-merged
        drwxr-xr-x   2 root root 4096 Apr 22  2024 boot
        drwxr-xr-x   5 root root  360 Oct  7 21:20 dev
        drwxr-xr-x   1 root root 4096 Oct  7 22:57 etc
        drwxr-xr-x   1 root root 4096 Oct  7 22:57 home
        drwxr-xr-x   2 root root 4096 Oct  7 22:57 lab
        lrwxrwxrwx   1 root root    7 Apr 22  2024 lib -> usr/lib
        lrwxrwxrwx   1 root root    9 Apr 22  2024 lib64 -> usr/lib64
        drwxr-xr-x   2 root root 4096 Sep 17 02:20 media
        drwxr-xr-x   2 root root 4096 Sep 17 02:20 mnt
        drwxr-xr-x   2 root root 4096 Sep 17 02:20 opt
        dr-xr-xr-x 237 root root    0 Oct  7 21:20 proc
        drwx------   1 root root 4096 Oct  7 22:46 root
        drwxr-xr-x   4 root root 4096 Sep 17 02:29 run
        lrwxrwxrwx   1 root root    8 Apr 22  2024 sbin -> usr/sbin
        drwxr-xr-x   2 root root 4096 Apr 15 06:53 sbin.usr-is-merged
        drwxr-xr-x   2 root root 4096 Sep 17 02:20 srv
        dr-xr-xr-x  12 root root    0 Oct  7 21:20 sys
        drwxrwxrwt   1 root root 4096 Oct  7 22:26 tmp
        drwxr-xr-x   1 root root 4096 Sep 17 02:20 usr
        drwxr-xr-x   1 root root 4096 Sep 17 02:29 var

    3.5
        root@6d356cbcb2c9:/# APP_ENV=staging; sh -c 'echo "child sees: $APP_ENV"'
        child sees:
        
        root@6d356cbcb2c9:/# export APP_ENV=staging

        root@6d356cbcb2c9:/# sh -c 'echo "child sees: $APP_ENV"'
        child sees: staging

    3.6
        root@6d356cbcb2c9:/#   cat /proc/1/cmdline | tr '\0' ' '
        bash root@6d356cbcb2c9:/#
        
        ji@LVAGPLTP5174:/mnt/c/WINDOWS/system32$ time docker stop lab
        lab

        real    0m10.551s
        user    0m0.027s
        sys     0m0.020s
    
    3.7 (outputs and one-line explanations)
        ji@LVAGPLTP5174:/mnt/c/WINDOWS/system32$ docker exec web netstat -ltn
        Active Internet connections (only servers)
        Proto Recv-Q Send-Q Local Address           Foreign Address         State
        tcp        0      0 127.0.0.11:38449        0.0.0.0:*               LISTEN
        tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN
        tcp        0      0 :::80                   :::*                    LISTEN
        ji@LVAGPLTP5174:/mnt/c/WINDOWS/system32$ curl -sI http://localhost:8081/ | head -1
        HTTP/1.1 200 OK


        bash root@6d356cbcb2c9:/# getent hosts web
        172.18.0.2      web
        root@6d356cbcb2c9:/# curl -sI http://web/ | head -1
        HTTP/1.1 200 OK

        nginx listens on port 80, which `lab` reaches directly over the shared `labnet` network as `web:80`, while my laptop, being outside that network, reaches the same port through Docker's port forwarding from `localhost:8081`.


## Part 4 · Broken containers

### lab2-broken:N
- Symptom:
- Cause:
- Fix:

## Answers

1. `cow:bad` contains no `/big.file`, yet it is 50 MB bigger than `cow:good`. Why?
    
    Because each RUN command creates a new layer that does not have an access to the previous one. In cow:good example a single Run command was able to use the big.file and delete it right away. The cow:bad did use a big.file in the first run command, then created a second RUN command to delete that file, but was not able to reach back in to do so. It could only mark it as hidden, not actually remove it.

2. Why does the order of `COPY` and `RUN` lines decide how long a rebuild takes?

    Well. Run installs the dependecies, which is the longest step in the exercise (15s sleep time really makes the difference here). Copy only does a small copy of a file whihch does not take as long. Then there is CACHED status which skips the Layer/command completley, it gets ignored. Layer gets cahced only if all the previous layers (on top of it) stays unchanged. Therefore, the ordering of the layers makes sense when they are sorted in a way where at the bottom sits layers that get changed the most. That way, each build can use as much CACHED layers as possible, reducing the execution time .

3. Why did `docker stop` take 10 seconds before you added the SIGTERM handler?

    docker stop first sends SIGTERM and waits up to 10 seconds for the app to exit. Our Node app ran as PID 1(main) and had no SIGTERM handler. Linux ignores unhandled signals for PID 1, so the app kept running. After 10 seconds Docker sent SIGKILL, which cannot be ignored, so the app was killed, which is why the exit code was 137 (128 + 9). After adding the handler, the app closed itself on SIGTERM and exited with 0 in under 2 seconds.

4. Name three things the naive image contained that `course-api:lab2` does not.
    1. The .env file with the database password (excluded by .dockerignore)
    2. Development dependencies such as test tools (npm ci --omit=dev installs only runtime dependencies)
    3. Compilers and build libraries from the full Debian-based node:24 image (replaced by the minimal node:24-alpine)