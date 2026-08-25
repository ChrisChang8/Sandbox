# Docker + Jenkins + Java Practice Project

A small Spring Boot app used to practice the classic CI/CD loop: build the app,
containerize it with Docker, and automate build → test → image → deploy with
Jenkins.

## Architecture

```mermaid
flowchart LR
    subgraph VM["Practice VM"]
        subgraph JenkinsC["Jenkins container"]
            J[Jenkins + Maven + Docker CLI]
        end
        subgraph Registry["registry:2 container"]
            R[(Local image registry :5000)]
        end
        subgraph AppC["demo-app container"]
            A[Spring Boot app :8080]
        end
        Sock[/var/run/docker.sock/]
    end

    Repo[(Git repo\n~/sandbox)] -- Jenkinsfile --> J
    J -- "1. mvn clean verify" --> J
    J -- "2. docker build" --> Sock
    J -- "3. docker push" --> R
    Sock -- "4. docker run" --> A
    J -- "5. curl health check" --> A
```

Jenkins runs the whole pipeline itself: it checks out the code, compiles/tests
it with Maven, then talks to the **host's** Docker daemon (via the mounted
`docker.sock`) to build the image, push it to a local registry, and run it as
a sibling container — all without needing a separate build agent.

## What each file is for

| File | Why it exists |
|---|---|
| `pom.xml` | Maven's project file. Tells Maven which dependencies to pull in (Spring Web, Actuator, Test) and how to package the app into a runnable jar. |
| `src/main/java/.../DemoApplication.java` | The entry point that boots the Spring Boot application. |
| `src/main/java/.../HelloController.java` | A minimal REST endpoint (`GET /hello`) so the pipeline has something real to build, test, and verify. |
| `src/test/java/.../HelloControllerTests.java` | A basic automated test that calls `/hello` and checks the response — gives the Jenkins pipeline something meaningful to run in the "Build & Test" stage. |
| `Dockerfile` | Recipe for packaging the **app itself** into a Docker image. Uses a multi-stage build: one stage compiles the jar with Maven, the second copies just the jar into a slim Java runtime image (keeps the final image small). |
| `Dockerfile.jenkins` | Recipe for building a **custom Jenkins image**. The stock `jenkins/jenkins:lts` image doesn't include Maven or the Docker CLI, both of which the pipeline needs to run on the Jenkins agent itself. |
| `Jenkinsfile` | The actual CI/CD pipeline definition (Jenkins Pipeline-as-Code). Defines the stages: checkout → build & test → docker build → docker push → deploy & verify. |
| `jenkins-job-config.xml` | Jenkins' internal job configuration XML, describing the Pipeline job (`demo-app-pipeline`) and pointing it at the `Jenkinsfile` in this repo. Used to create the job without clicking through the UI. |
| `.dockerignore` / `.gitignore` | Keep build output (`target/`) and other noise out of Docker build contexts and git history. |

## Why Docker-outside-of-Docker (DooD)

Jenkins runs in its own container but needs to build/run Docker images. Instead
of running a separate Docker daemon inside Jenkins (Docker-in-Docker), this
setup mounts the **host's** `/var/run/docker.sock` into the Jenkins container.
Jenkins then talks to the same Docker engine the host uses, and any containers
it starts (like `demo-app`) are **siblings** of the Jenkins container, not
children of it — which is why the pipeline has to reach the app container by
its own container IP rather than `localhost`.

## Practice flow

1. **Local build** — `mvn test` / `mvn package` to make sure the app works
   before involving Docker at all.
2. **Manual Docker practice** — build the image by hand, run it, curl it, push
   it to a local `registry:2` container, pull it back down. This builds the
   core Docker muscle memory.
3. **Jenkins in Docker** — run Jenkins itself as a container, with Maven and
   the Docker CLI baked into a custom image.
4. **Pipeline as code** — the `Jenkinsfile` automates the same steps done
   manually in step 2.
5. **Wire it up** — a Jenkins Pipeline job runs the `Jenkinsfile` against this
   repo, end to end.


## Multi-Stage Build

1. Build the image and tag it baseline: `docker build -t demo-app:baseline`
2. List hte image and note its size (8/25 - 317MB): `docker images demo-app`
3. Read Dockerfile `COPY --from=build /app/target/demo-app-*.jar app.jar`, this tells Docker "reach into the build stage's filesystem and copy this one file out." Everything else from stage 1 (Maven cache, JDK, .java sources, pom.xml) is discarded — it never becomes part of the final image layers.
4. Verify discarding: `docker history demo-app:baseline` You should see layers corresponding only to the eclipse-temurin:17-jre base + the COPY app.jar + EXPOSE/ENTRYPOINT metadata — nothing about mvn clean package.
5. Confirm OS Base image: `docker run --rm eclipse-temurin:17-jre cat /etc/os-release`, ensure it's Debian based, and use Debian syntax
6. Udpate Docker Runtime stage: 
    * Ensure changes are applied: `nano ~/sandbox/Dockerfile`
    ```
    FROM eclipse-temurin:17-jre
    RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
    WORKDIR /app
    COPY --from=build /app/target/demo-app-*.jar app.jar
    RUN chown appuser:appgroup app.jar
    USER appuser
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "app.jar"]
    ```
    Why each line is there:

    - `addgroup --system / adduser --system` — creates a system account with no login      shell/home dir/password, standard for Ubuntu/Debian.
    - `chown appuser:appgroup app.jar` — the jar is copied in as root; without this, appuser can't read it.
    - `USER appuser` comes after COPY/chown (which need root) so only the running java process drops privileges.
7. Rebuild and Verify the non-root user works 
    - Rebuild: `docker build -t demo-app:secure .`
        - `docker build --no-cache -t demo-app:secure .`
        - `docker build --no-cache --pull -t demo-app:secure .`
    - Verify: `docker run --rm demo-app:secure whoami`
        -  `docker run --rm --entrypoint whoami demo-app:secure`
    - Compare both tags side by side: `docker images demo-app`
        - More detail comparison: 
            - `docker inspect -f "{{.Size}}" demo-app:baseline`
            - `docker inspect -f "{{.Size}}" demo-app:secure`
    - Sanity Check: `docker run --rm -p 8080:8080 demo-app:secure`
        - If Port Occupied, run
            - list all running containers: `docker ps`
            - stop containers: `docker stop a1f0bdfacf29`
            - run server: `docker run --rm -p 8081:8080 demo-app:secure`, jenkins default 8080, also the -rm deletes afterwards

8. Add sql dependy pom.xml:
    ```
    <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    ```
9. Created Person Entity, locaated in src/main/java/com/example/demo/Person.java
10. Create the controller — new file (PersonController.java)
11. Add datasource config — append to ~/main/java/resources/application.properties
12. Create init.sql — new file at repo root (init.sql):

