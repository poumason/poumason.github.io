---
title: 利用 Jinja2 產生 gitlab-ci-yaml 並自動化 trigger pipeline
date: 2024-09-07 14:37:41
categories:
  - DevOps
tags: DevOps
---
介紹如何透過 Jinja2 套件動態產生 gitlab-ci.yml 檔，並透過 gitlab-ci.yml 來自動化 trigger 下一個 pipeline。
<!-- more -->
Jinja2 是一種 python package 可以用來動態產生程式碼，僅需先寫好要產生內容的 template 檔，再透過 Jinia2 的 render() 方法來產生想要的內容。

為什麼我會需要動態產生 gitlab-ci.yml 檔？主因是 CD 過程需要分不同的環境佈署(ex: DEV/STAGING/PROD)，我希望可以一次產生不同的環境的 gitlab-ci.yaml 檔案，讓使用者自行選擇要佈署的環境。

我知道，也許您會想到另一種方式，例如：
```
variables:
  DEPLOY_ENVIRONMENT:
    value: "staging"
    description: "The deployment target. Change this variable to 'canary' or 'production' if needed."

stages:
  - deploy

deploy_${DEPLOY_ENVIRONMENT}:
  stage: deploy
  script:
    - echo ${DEPLOY_ENVIRONMENT}
  rules:
    - if $DEPLOY_ENVIRONMENT == "staging"
```
讓使用者在觸發 pipeline 的時候，自己再給值就好了。

確實上述的方式是我一開始的作法，但考量到未來我的環境愈來愈多或是 gitlab runner 有不同環境時，不斷地用 rules 去判斷並不是一個好的維讀方式。

因此，我需要 Jinja2 來幫助我完成這個需求。

舉例來說：我準備一份 gitlab-ci.yaml 的 template 將基本的任務寫上，並將會動態切換的內容用 jinja2 語法保留起來。

# 定義 template
```
variables:
  - RELEASE_NAME: {{ data.release_name }}

stages:
  - deploy

{% for item in data.deploy_env %}
{{ item.env }}_deploy_package:
    stage: deploy
    script:
        - echo "$RELEASE_NAME"
        - echo "Deploying {{ item.date }} to {{ item.target }}"
    when: manual
    tags:
        - {{ item.tag }}
{% endfor %}
```
template 定義要使用的 yaml 檔，並且在撰寫的時候要記得它是使用 python dict 的結構來進行(也可以相像成 json)並將需要依不同環境的地方保留下來，例如∶
- job name
- job script
- job tag

其中用了 `{% for %}` 迴圈，將 `data.deploy_env` 的每一項目依序渲染出來。


# 如何 redner 內容
有了 template 之後，我們需要定義 data 內容與要如何使用 jijia2 python library 將需要的內容組合出來。

### data 內容
```
{
    "release_name": "APP",
    "deploy_env": [
        {
            "env": "dev",
            "date": "2024/09/12",
            "target": "taipei",
            "tag": "runner-taipei"
        },
        {
            "env": "stage",
            "date": "2024/09/13",
            "target": "taichung",
            "tag": "runner-taichung"
        }
    ]
}
```
嘗試把實際的 data 先寫成一份 json，然後在經過 python 把它讀進來。如果要動態產生這份 data 的結構也沒有問題的。

### python code
```
import jinja2
import json

# Load JSON data
with open('./templates/pipeline.json') as f:
    data = json.load(f)

# Jinja2 template
with open('./templates/deploy-ci.j2', 'r') as template_file:
    template = template_file.read()

# Render the template
jinja_env = jinja2.Environment()
rendered_template = jinja_env.from_string(template).render(data=data)

# Save the rendered template to a file
with open('output.yaml', 'w') as f:
    f.write(rendered_template)
```
程式碼裡最重要的是先準備 `jinja2.Environment()` 再將寫好的 template 與 data 傳給它請它進行渲染。最後將渲染的結果寫到 output.yaml 如下的內容：
```
variables:
  - RELEASE_NAME: APP

stages:
  - deploy

dev_deploy_package:
    stage: deploy
    script:
        - echo "Deploying 2024/09/12 to taipei"
    tags:
        - runner-taipei

stage_deploy_package:
    stage: deploy
    script:
        - echo "Deploying 2024/09/13 to taichung"
    tags:
        - runner-taichung
```

# 如何串接在 repo 的 gitlab-ci.yaml 讓產生的內容能被 trigger
從上述了解如何使用 jinjia2 動態產生一份 yaml 後，那我們嘗試將它串回到原有的 repo 的 gitlab-ci.yaml 裡。

舉例來說目前的 gitlab-ci.yaml 有以下的內容：
```
stages:
  - generate

generate-deploy-env:
  stage: generate
  image:
    name: python:3.9-alpine
  script:
    - ls
    - pip install -r ./requirements.txt
    - python3 main.py
  artifacts:
    paths:
      - output.yaml
```
內容有負責利用 jinjia2 產生的 output.yaml，接著把 output.yaml 放入 artifacts 裡，提供給下一個 job 使用。

接著加入一個新的 job，並使用上一個 job 所建立的 output.yaml 來觸發 trigger pipeline：
```
deploy-trigger:
  stage: deploy
  trigger:
    include:
      - artifact: output.yaml
        job: generate-deploy-env
```

最後的結果就會產生如下的 pipeline 結果：
![](Screenshot-0.png)

以上是分享如何使用 jinjia2 動態產生 yaml 最後再串接至既有的 gitlab-ci.yaml 之間的流程。

最後備註 Jinja2 的一些常用寫法。

## 基本語法介紹
- `{% %}`: 表示 Jinia2 的開始與結束，內容會被視為程式碼執行
- `{{ }}`: 代表一個變數或字串
- `{% %}`: 代表一段程式碼
- `{% if %}`: 判斷式
- `{% for %}`: 迴圈
- `{% set %}`: 設定變數
- `{% include %}`: 引入檔案
- `{% extends %}`: 繼承
- `{% block %}`: 覆蓋父模板的區塊
- `{% macro %}`: 定義宏
- `{% call %}`: 呼叫
- `{% import %}`: 匯入模組
- `{% from %}`: 從模組中引入變數
- `{% with %}`: 設定變數
- `{% set %}`: 設定變數

## 常用的 filter
- `{{  | safe }}`: 將 HTML 標籤轉換為安全的 HTML 標籤
- `{{ | striptags  }}` : 去除 HTML 標籤
- `{{  | replace('a', 'b') }}` : 替換字串
- `{{   | replace('a', 'b', count=n) }}`  : 替換字串，限制 n 次
- `{{ | int }}`: 轉換為整數
- `{{   | float  }}`: 轉換為浮點數
- `{{ | trim }}`: 去除前後空白


# References
- [Jinja2](https://jinja.palletsprojects.com/en/3.1/)
- [GitLab CI/CD - GitLab Documentation](https://docs.gitlab.com/ee/ci/yaml/#include)
- [一篇文章搞懂Jinja2 Template Engine 模版引擎](https://segmentfault.com/a/1190000018002480)
- [Python — Jinja2 網頁模板](https://jones-coding-room.medium.com/python-%E7%B6%B2%E9%A0%81%E6%A8%A1%E6%9D%BF-jinja2-4081675adc12)
- [Jinja2 Template Designer Documentation](https://jinja.palletsprojects.com/en/3.1/templates/#include)
- [jinja2的基本介绍和使用](https://blog.csdn.net/weixin_48419914/article/details/123579571)
- [和艦長一起 30 天玩轉 GitLab](https://gitlab-book.tw/ithelp/gitlab-ci/chatops/)
- [Jinja2模板引擎的用法](https://www.osgeo.cn/python-tutorial/webpub-jinja2.html)