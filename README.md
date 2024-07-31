**PHASE 1: BUILD THE CONTAINER FOR THE APP**
In Phase 1 I'm going to focus on building the image and container for the sample app to get started

- Install Docker Desktop or other necessary files for the OS

- Use Git to clone the sample application from Docker to my local machine

~~~	
git clone https://github.com/spring-projects/spring-petclinic.git
~~~

- Since I'm using a sample application from Docker, I'm going to use the 'docker init' command to streamline the default configuration.

~~~	
docker init
~~~

- I answered the utility like this:

~~~
WARNING: The following Docker files already exist in this directory:
  - docker-compose.yml
? Do you want to overwrite them? Yes
	The sample app already has a 'docker-compose.yaml' file so I want to overwrite it instead of creating a new 'compose.yaml' file.

? What application platform does your project use? Java
	The sample application is a Spring Boot application built with Maven.

? What's the relative directory (with a leading .) for your app? ./src

? What version of Java do you want to use? 17
	The sample app used version 17 so I used the same version to avoid any issues.

? What port does your server listen on? 8080
~~~

- Inside the 'spring-petclinic' directory that I cloned I use the 'docker compose up --build' command to build the image, container, detach the container, and start services

~~~	
docker compose up --build -d
~~~

- Due to the port configuration from earlier the app can be viewed in a browser at: **'http://localhost:8080'**

- To stop the container/application use the command **'docker compose down'**


**PHASE 2: DEVELOP THE APPLICATION ENVIRONMENT**
In Phase 2 I'm going to add three improvements: add a database for persistent data, create a development container, and configure Docker Compose to automatically update my containers.

- Modify the **'docker-compose.yaml'** file like so to include a depends on attribute for database creation, health check for the database, and environment variables to establish the name/user/pass.

~~~yaml
services:
  server:
    build:
      context: .
    ports:
      - 8080:8080
    depends_on:
      db:
        condition: service_healthy
    environment:
      - POSTGRES_URL=jdbc:postgresql://db:5432/petclinic
  db:
    image: postgres
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=petclinic
      - POSTGRES_USER=petclinic
      - POSTGRES_PASSWORD=petclinic
    ports:
      - 5432:5432
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "petclinic" ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
~~~


- Modify the **Line 92 in the Dockerfile** to pass in a postgres system property as noted in the included **'petclinic_db_setup_postgres.txt'** file.

~~~	
'ENTRYPOINT [ "java", "-Dspring.profiles.active=postgres", "org.springframework.boot.loader.launch.JarLauncher" ]'
~~~

- Save and close the files, now run the 'docker compose up --build -d' command
- Navigate around and it should operate normally, the volume can be viewed with 'docker volume ls' as well.

- Next I'll modify the Dockerfile to build a 'development' stage, this stage is based on the 'extract' stage and will copy the extracted files to a common directory then run a command to start the application. It also exposes **port 8000** and declares the debug configuration for the JVM so a debugger can be attached.

Add this under **Line 55 in the Dockerfile**:

~~~
FROM extract as development
WORKDIR /build
RUN cp -r /build/target/extracted/dependencies/. ./
RUN cp -r /build/target/extracted/spring-boot-loader/. ./
RUN cp -r /build/target/extracted/snapshot-dependencies/. ./
RUN cp -r /build/target/extracted/application/. ./
ENV JAVA_TOOL_OPTIONS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000
CMD [ "java", "-Dspring.profiles.active=postgres", "org.springframework.boot.loader.launch.JarLauncher" ]
~~~

- The **'docker-compose.yaml'** file needs to be updated to include the new development stage configuration. 

~~~yaml
services:
  server:
    build:
      context: .
      target: development
    ports:
      - 8080:8080
      - 8000:8000
    depends_on:
      db:
        condition: service_healthy
    environment:
      - POSTGRES_URL=jdbc:postgresql://db:5432/petclinic
  db:
    image: postgres
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=petclinic
      - POSTGRES_USER=petclinic
      - POSTGRES_PASSWORD=petclinic
    ports:
      - 5432:5432
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "petclinic" ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
~~~

- Using the IntelliJ IDEA I'll setup a debugger named **'RemoteDebug'** that's included with the IDE.
- Now open **'src/main/java/org/springframework/samples/petclinic/vet/VetController.java'** and set a breakpoint with **ctrl+shift+alt+F8** on the line **'showResourcesVetList'** function.

(The breakpoint will trigger when using the curl command on the container, this is a simple example of debugging code. The debugger has much more functionality that is worth exploring.)

- Compose Watch will be setup to automatically update any Docker Compose services as code is changed. The 'docker-compose.yaml' is changed with the following lines:

~~~yaml
services:
  server:
    build:
      context: .
      target: development
    ports:
      - 8080:8080
      - 8000:8000
    depends_on:
      db:
        condition: service_healthy
    environment:
      - POSTGRES_URL=jdbc:postgresql://db:5432/petclinic
    develop:
      watch:
        - action: rebuild
          path: .
  db:
    image: postgres
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=petclinic
      - POSTGRES_USER=petclinic
      - POSTGRES_PASSWORD=petclinic
    ports:
      - 5432:5432
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "petclinic" ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
~~~

**PHASE 3: TESTING THE APPLICATION BUILD**
This phase will integrate a unit test into the application while Docker is creating the image.

- First, added a new base stage. In the base stage, I added common instructions that both the test and deps stage will need.

- Next, I added a new test stage labeled test based on the base stage. In this stage I copied in the necessary source files and then specified RUN to run './mvnw' test. Instead of using CMD, I used RUN to run the tests. The reason is that the CMD instruction runs when the container runs, and the RUN instruction runs when the image is being built. When using RUN, the build will fail if the tests fail.

- Finally, I updated the deps stage to be based on the base stage and removed the instructions that are now in the base stage.

- Run the following command to build a new image using the test stage as the target and view the test results. Include --progress=plain to view the build output, --no-cache to ensure the tests always run, and --target-test to target the test stage.

~~~
'docker build -t java-docker-image-test --progress=plain --no-cache --target=test .'
~~~

**PHASE 4: CONFIGURE CI/CD FOR THE JAVA APPLICATION**
Setup a GitHub Repo to build and push a Docker image to Docker Hub.

**Create a new Repository**
- Create a new repository on GitHub.

- Open the repository Settings, and go to **Secrets and Variables > Actions**.

- Create a new Repository variable named **DOCKER_USERNAME** and your Docker ID as value.

- Create a new **Personal Access Token (PAT)** for Docker Hub. You can name this token **docker-tutorial**. Make sure access permissions include **Read and Write.**

- Add the PAT as a Repository secret in your GitHub repository, with the name **DOCKERHUB_TOKEN**.

- In your local repository on your machine, run the following command to change the origin to the repository you just created. Make sure you change your-username to your GitHub username and your-repository to the name of the repository you created.

~~~
'git remote set-url origin https://github.com/your-username/your-repository.git'
~~~

- Run the following commands to stage, commit, and push your local repository to GitHub.

~~~
git add -A
git commit -m "my commit"
git push -u origin main
~~~

**Set up a Git workflow**
- Select New workflow.

- Select set up a workflow yourself.

- This takes you to a page for creating a new GitHub actions workflow file in your repository, under .github/workflows/main.yml by default.

- In the editor window, copy and paste the following YAML configuration.

~~~yaml
name: ci

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      -
        name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      -
        name: Build and test
        uses: docker/build-push-action@v6
        with:
          target: test
          load: true
      -
        name: Build and push
        uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          target: final
          tags: ${{ vars.DOCKER_USERNAME }}/${{ github.event.repository.name }}:latest
~~~

**Run the workflow**
- Select Commit changes... and push the changes to the main branch.

- After pushing the commit, the workflow starts automatically.

- Go to the Actions tab. It displays the workflow.

- Selecting the workflow shows you the breakdown of all the steps.

- When the workflow is complete, go to your repositories on Docker Hub.
If you see the new repository in that list, it means the GitHub Actions successfully pushed the image to Docker Hub.


**PHASE 5: TEST THE JAVA APP DEPLOYMENT**
Docker Desktop will be used to deploy the sample application to a fully-featured Kubernetes environment.

- Create a new YAML file to configure Kubernetes with the following code:

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-java-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      service: server
  template:
    metadata:
      labels:
        service: server
    spec:
      containers:
       - name: server-service
         image: #DOCKER_USERNAME/REPO_NAME#
         imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: service-entrypoint
  namespace: default
spec:
  type: NodePort
  selector:
    service: server
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30001
~~~

This code will define the pod with the container and the NodePort service which routes traffic from port 30001 to port 8080 so the app can communicate

- **Deploy and check the application**

In a terminal, navigate to spring-petclinic and deploy the application to Kubernetes.

~~~
kubectl apply -f docker-java-kubernetes.yaml
~~~

The output should look like this:

~~~
deployment.apps/docker-java-demo created
service/service-entrypoint created
~~~

Make sure everything worked by listing the deployments.

~~~
kubectl get deployments
~~~

The deployment should be listed as: **docker-java-demo**

This indicates all of the pods you asked for in your YAML are up and running. Do the same check for your services.

~~~
kubectl get services
~~~

I should get output like the following.

~~~
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP          23h
service-entrypoint   NodePort    10.99.128.230   <none>        8080:30001/TCP   75s
~~~

In addition to the default kubernetes service, see the service-entrypoint service, accepting traffic on port 30001/TCP.

In a terminal, curl the service.

~~~
 curl --request GET \
  --url http://localhost:30001/actuator/health \
  --header 'content-type: application/json'
~~~

I should get output like this.

~~~
{"status":"UP","groups":["liveness","readiness"]}
~~~

Run the following command to tear down the application.

~~~
kubectl delete -f docker-java-kubernetes.yaml
~~~
