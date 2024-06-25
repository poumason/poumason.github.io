---
title: 如何利用 variables 切換不同的 CI template
tags:
  - DevOps
categories: DevOps
date: 2024-06-13 23:10:34
---

.gitlab-ci.yml 是一份事先定義好的執行腳本，讓 Gitlab 根據裡面的設定產生對應的 pipeline。
那如何需要按不同的執行參數來切換不同的腳本內容，又要怎麼定義呢？
<!-- more -->

# 定義變數來切換不同的 yml

repo 檔案結構如下:
- .gitlab-ci.yml
- HOME/deploy.yml
- SCHOOL/deploy.yml

在 .gitlab-ci.yml 利用特定變數(ex: $ENV)與 rules 來判斷該讀取哪一份 yml 進來，如下:
```
include:
  - local: HOME/deploy.yml
    rules:
      - if: $ENV == "home"
  - local: SCHOOL/deploy.yml
    rules:
      - if: $ENV == "school"
stages:
  - test

test:
  stage: test
  script:
    - echo $ENV
    # 來自 deploy.yml 內有的變數
    - echo $LOCATION
  rules:
    - if: $ENV != null
```
那 deploy.yml 要放什麼呢？
```
variables:
  LOCATION: "HOME"
```

透過 $ENV 的變數來控制 .gitlab-ci.yml 應該 include 哪一份 yaml 進來。

# 如何讓 downstream 取得 upstream 的 artificat
## upstream 的 ci yaml
```
stages:
  - build
  - trigger

build-job:
  stage: build
  script:
    - echo "Building the project..."
    - mkdir -p output
    - echo "artifact content" > output/artifact.txt
  artifacts:
    paths:
      - output/artifact.txt
    expire_in: 1 day

trigger_job:
  stage: trigger
  trigger:
    project: poulin/cd-template
    branch: main
    #wait for pipeline to run
    strategy: depend 
  variables:
    UPSTREAM_PROJECT_ID: $CI_PROJECT_ID
    UPSTREAM_PIPELINE_ID: $CI_PIPELINE_ID
  when: manual

```
## downstream 的 ci yaml
```
stages:
  - download

download-upstream-artifacts:
  stage: download
  variables:
    DEPLOY_TOKEN_PASSWORD: $TOKEN
  image:
    name: amazon/aws-cli:latest
    entrypoint:
      - ""
  script:
    - yum install jq unzip -y
    - echo "Downloading artifacts from upstream..."
    - |
      job_id=$(curl --silent --header "PRIVATE-TOKEN: ${DEPLOY_TOKEN_PASSWORD}" \
        "https://gitlab.com/api/v4/projects/${UPSTREAM_PROJECT_ID}/pipelines/${UPSTREAM_PIPELINE_ID}/jobs" \
        | jq -r '.[] | select(.name=="build-job") | .id')
      curl --location --header "PRIVATE-TOKEN: ${DEPLOY_TOKEN_PASSWORD}" \
        "https://gitlab.com/api/v4/projects/${UPSTREAM_PROJECT_ID}/jobs/${job_id}/artifacts" --output artifacts.zip
    - echo "Unzipping the artifacts"
    - unzip artifacts.zip
    - cat output/artifact.txt
  needs:
    - pipeline: $UPSTREAM_PIPELINE_ID
      job: build-job
      artifacts: false
  dependencies: []
  only:
    - pipelines
```
- 需要設定一組 access token 放在 downstream 的 repo 中


# References
- [Accessing Artifacts Created in a GitLab Upstream Pipeline From A Downstream Pipeline](https://blog.devops.dev/accessing-artifacts-created-in-a-gitlab-upstream-pipeline-from-a-downstream-pipeline-e50c16ab6f87)