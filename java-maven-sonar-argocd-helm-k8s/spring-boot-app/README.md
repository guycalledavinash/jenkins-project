# Spring Boot web application

This application can be built with Maven from this directory:

```sh
mvn clean verify
```

The resulting JAR is written under `target/`. Build and run a local container with:

```sh
docker build -t spring-boot-app:local .
docker run --rm -p 8010:8080 spring-boot-app:local
```

## Jenkins pipeline

The `JenkinsFile` in this directory runs checkout, Maven build and tests, SonarQube analysis, Docker build/push, and an optional deployment manifest update. It updates a Git repository only; it does not deploy to a Kubernetes cluster.

### Jenkins prerequisites and credentials

Configure these credentials in Jenkins Credentials Manager:

- `sonarqube`: Secret text containing a SonarQube analysis token.
- `docker-cred`: Username with password for the configured container registry. Used only when `PUBLISH_IMAGE=true`.
- `deployment-repository-ssh`: SSH Username with private key, with write access to the chosen deployment repository branch. Used only when `UPDATE_DEPLOYMENT_MANIFEST=true`.

The Jenkins agent image must include Java/Maven, Git, Docker CLI, and mikefarah `yq` v4. The Docker Pipeline, Git, and SSH Agent plugins must be installed. The agent must run as a non-root user and have read/write access to `/var/run/docker.sock`; the pipeline checks this before building. Docker socket access grants control of the host Docker daemon and is effectively host-level access, so use an isolated, dedicated Jenkins agent and never expose this socket to untrusted jobs.

### Parameters

Set the following Jenkins build parameters for the services you want to run:

- `SONAR_HOST_URL`: your configured SonarQube URL. HTTPS is recommended when supported.
- `DOCKER_IMAGE_REPOSITORY`: full image repository without a tag, such as `registry.example/team/app`.
- `DOCKER_REGISTRY_URL`: HTTPS registry URL, required only when publishing.
- `RUN_SONAR_ANALYSIS`: defaults to true. Set false for a local pipeline smoke test without SonarQube access.
- `PUBLISH_IMAGE`: defaults to false. Enables registry login and image push.
- `UPDATE_DEPLOYMENT_MANIFEST`: defaults to false. Requires `PUBLISH_IMAGE=true`.
- `DEPLOY_REPOSITORY_URL`: SSH Git URL of the deployment repository.
- `DEPLOY_BRANCH`: branch to update.
- `DEPLOY_MANIFEST_PATH`: relative path to the Kubernetes deployment YAML; defaults to this repository's sample manifest path.
- `DEPLOY_CONTAINER_NAME`: container to update; defaults to `spring-boot-app`.
- `GIT_COMMIT_AUTHOR_NAME` and `GIT_COMMIT_AUTHOR_EMAIL`: identity recorded in the manifest commit.

The image tag is generated from the Jenkins build number and the checked out commit's first 12 characters, so each build gets a traceable tag. The image repository is supplied separately; the pipeline does not assume a registry account or repository.

### Manifest update behavior

When enabled, the pipeline checks out the configured repository and branch using the SSH credential, verifies that exactly one container with the selected name exists, and confirms its image is either the `replaceImageTag` placeholder or already belongs to the configured image repository. It changes only that container's `image` field using `yq`, verifies the new value, stages only the specified manifest, and pushes the commit. Missing files, containers, image fields, invalid placeholders, or Git failures stop the build. If the manifest already has the desired image, it skips the commit and push and reports that no change was made.

### Safe pipeline testing

Run `mvn clean verify` locally to validate the application without contacting SonarQube, a registry, or a deployment repository.

For a Jenkins smoke test, set `RUN_SONAR_ANALYSIS=false` and leave both `PUBLISH_IMAGE` and `UPDATE_DEPLOYMENT_MANIFEST` false. The pipeline will run Maven and build the image locally, but will not log into or push to a registry and will not check out or change a deployment repository. This still requires an isolated Jenkins agent with Docker socket access.
