---
title: CI/CD Pipeline for Versioned Docker Releases Using GitLab
date: 2025-03-27 12:00:00 +0800
categories: [DevOps, GitLab, Docker]
tags: [gitlab, docker, git]
image: /images/post-1/docker-build-push-gitlab-ci.png
show_top_image: false
---

To make deployments easier and more reliable, I have created a GitLab CI/CD pipeline.

Whenever I push code to a `release-X.X.X` branch, GitLab automatically builds a Docker image with the updated `index.html`, tags the repository, and pushes the image to DockerHub. A manual deployment job is also available, giving me full control over when to release the image.

<div style="display: flex; justify-content: center; margin: 20px 0;">
  <div style="flex: 1;"></div>
  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-1/flow-chart.png" class="popup img-link shimmer">
      <img src="/images/post-1/flow-chart.png" alt="GitLab CI/CD flow for Docker releases"
           style="width: 100%; max-width: 500px; border-radius: 8px;" loading="lazy" />
    </a>
  </div>
  <div style="flex: 1;"></div>
</div>

<div style="text-align: center; font-size: 0.9em; color: gray; margin-top: -20px;">
  <em>Figure: GitLab CI/CD flow for versioned Docker releases</em>
</div>

---

### Prerequisites

To set this up, I already had:

- An EC2 instance with Docker installed and accessible via SSH  
- A GitLab repository for storing the `.gitlab-ci.yml` file  
- A DockerHub account with credentials ready  

---

### Pipeline Configuration

<p style="margin-bottom: 0.5em; font-weight: bold;">.gitlab-ci.yml</p>

<div style="max-height: 400px; overflow-y: auto; background: #000000; color: #00B200; padding: 1em; border-radius: 6px; font-size: 14px;">
<pre><code class="language-yaml">
stages:
  - build
  - tag
  - deploy

variables:
  DOCKER_IMAGE: vamsi193/release-cicd

workflow:
  rules:
    - if: '$CI_COMMIT_REF_NAME =~ /^release-\d+\.\d+\.\d+$/'
      when: always
    - when: never

build_image:
  stage: build
  tags:
    - my-runner
  script:
    - echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
    - TAG_NAME="${CI_COMMIT_REF_NAME#release-}"
    - echo "TAG_NAME=$TAG_NAME" > tag.env
    - docker build -t $DOCKER_IMAGE:$TAG_NAME .
    - docker push $DOCKER_IMAGE:$TAG_NAME
  artifacts:
    reports:
      dotenv: tag.env

tag_repo:
  stage: tag
  tags:
    - my-runner
  dependencies:
    - build_image
  script:
    - git tag v$TAG_NAME
    - git push https://oauth2:${CI_GITLAB_PAT}@gitlab.com/vamsi193/release-cicd.git v$TAG_NAME

manual_deploy:
  stage: deploy
  tags:
    - my-runner
  script:
    - echo "Stopping existing container if it exists..."
    - docker stop release-app || true
    - docker rm release-app || true
    - docker run -d --name release-app -p 80:80 $DOCKER_IMAGE:$TAG_NAME
  when: manual
</code></pre>
</div>

---

### Workflow Logic

The `workflow` block ensures the pipeline runs **only** when the branch name follows the `release-X.X.X` format:

```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_REF_NAME =~ /^release-\d+\.\d+\.\d+$/'
      when: always
    - when: never
```

---

### Stage Breakdown

- **`build_image`**: Builds the Docker image, tags it with the version, and pushes it to DockerHub.  
- **`tag_repo`**: Automatically tags the Git repository with `vX.X.X`.  
- **`manual_deploy`**: Lets me deploy the container manually when I’m ready — no automatic production deploys.  

---

<div style="display: flex; gap: 20px; justify-content: center; flex-wrap: wrap; margin-bottom: 2em;">
  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-1/gitlab-image.png" class="popup img-link shimmer">
      <img src="/images/post-1/gitlab-image.png" alt="GitLab Pipeline"
           style="width: 100%; max-width: 400px; border-radius: 8px;" loading="lazy" />
    </a>
    <p style="margin-top: 0.5em; font-size: 14px;">GitLab Pipeline Status</p>
  </div>

  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-1/dockerhub.png" class="popup img-link shimmer">
      <img src="/images/post-1/dockerhub.png" alt="DockerHub Tag"
           style="width: 100%; max-width: 400px; border-radius: 8px;" loading="lazy" />
    </a>
    <p style="margin-top: 0.5em; font-size: 14px;">DockerHub images with tag</p>
  </div>
</div>

---

### Insights from This Pipeline

- Built and pushed Docker images when a release tag is created  [DockerHub](https://hub.docker.com/r/vamsi193/release-cicd/tags)

- Tagged the Git repository with the version [GitLab Repository](https://gitlab.com/vamsi193/release-cicd)

- Deployed the image manually in a safe and controlled way [Pipeline View](https://gitlab.com/vamsi193/release-cicd/-/pipelines/1895260863)

---
