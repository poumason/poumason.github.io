---
title: 在 Azure functions 使用自定義的 Docker Image
date: 2022-01-28 22:16:09
categories: DevOps
tags:
  - DevOps
  - Azure
---
如需在 Azure functions 使用特定的 Lib 版本或第三方套件，例如：ffmpeg，可建立專用的 Docker Image。本篇利用 [Node.js](https://docs.microsoft.com/en-us/azure/developer/javascript/how-to/develop-serverless-apps) 為例分享使用的心得。
<!-- more -->

# 基本需求
- 使用 [Premium plan](https://docs.microsoft.com/en-us/azure/azure-functions/functions-premium-plan) 或是 [Dedicated (App Service) plan](https://docs.microsoft.com/en-us/azure/azure-functions/dedicated-plan)
- 使用的 docker image 必須基於 [Azure Functions Base](https://hub.docker.com/_/microsoft-azure-functions-base)
- 擁有 [docker hub](https://hub.docker.com/) 或 [Azure Container registry](https://azure.microsoft.com/zh-tw/services/container-registry/)
- [開發環境安裝必要的工具](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-function-linux-custom-image?tabs=in-process%2Cbash%2Cazure-cli&pivots=programming-language-javascript#configure-your-local-environment)

下面分別介紹幾個步驟

# 建立 Docker Image 加入需要的內容
> 沒有 Azure functions 專案，參考 建立和測試本機 Functions 專案 準備具有 Http Trigger專案；
在本機專案的根目錄建立 Dockerfile，並填入 ffmpeg 與 curl 的安裝。
```
FROM mcr.microsoft.com/azure-functions/node:3.0
# Install ffmpeg and curl
RUN apt-get update
RUN apt-get -y install curl
RUN apt-get -y install ffmpeg
# Setting environment variables of Azure functions
ENV AzureWebJobsScriptRoot=/home/site/wwwroot \
    AzureFunctionsJobHost__Logging__Console__IsEnabled=true
COPY . /home/site/wwwroot
RUN cd /home/site/wwwroot && \
    npm install
```
上述指令把專案中所有的 functions 複製到 **/home/site/wwwroot**。專案目錄結構需要符合基本規範。

介紹環境變數：
- AzureWebJobsScriptRoot
  代表 Azure functions 專案的執行根目錄，也就是 host.json 的目錄；
- AzureFunctionsJobHost{__path__to__setting}
  設定需要複寫(overwrite) host.json 裡面的變數值，__path__to__settings 可[參考說明](https://docs.microsoft.com/en-us/azure/azure-functions/functions-host-json#override-hostjson-values)換成需要的變數名稱，例如：AzureFunctionsJobHost__Logging_Console__IsEnabled；

設定 Azure functions 的環境變數，可以從兩個來源：
- 固定寫在 Docker image：常放一些通用的資訊
- 利用 Azure portal 上的應用程式設定：建議放有安全疑慮的資訊

兩個來源在 functions 執行時組合在 environment variables，以 node.js 為例，可以使用 `process.env.XXXXX` 的方式來取得；
在測試 docker image 時，建議使用 Http Trigger 的 azure function，因為較能方便測試與除錯。

# 建立 docker container 與本機測試
`docker build --tag <docker ID>/my_az_functions_image:latest .`
如果有修改專案，要記得重新建立 image 並送到 regsitry。

本機測試檢查 http trigger 是否被掛起：
`docker run -it --rm -p 8080:80 <docker ID>/my_az_functions_image:latest`

啟動後，利用 `http://localhost:8080/api/{http trigger name}?name=pou` 檢查 function 是否被掛起；如果預設使用 template 建立的 http trigger 會看到對應的 response，例如下面的 log：
```
info: Function.HttpGreeting[1]
      Executing 'Functions.HttpGreeting' (Reason='This function was programmatically called via the host APIs.', Id=cb489e5e-ef7e-4840-aa08-9bf1bcc32961)
info: Function.HttpGreeting.User[0]
      JavaScript HTTP trigger function processed a request.
info: Function.HttpGreeting[2]
      Executed 'Functions.HttpGreeting' (Succeeded, Id=cb489e5e-ef7e-4840-aa08-9bf1bcc32961, Duration=115ms)
info: Function.HttpGreeting[1]
      Executing 'Functions.HttpGreeting' (Reason='This function was programmatically called via the host APIs.', Id=61e1265a-59c3-4d24-86c8-2c7d55021149)
info: Function.HttpGreeting.User[0]
      JavaScript HTTP trigger function processed a request.
info: Function.HttpGreeting[2]
      Executed 'Functions.HttpGreeting' (Succeeded, Id=61e1265a-59c3-4d24-86c8-2c7d55021149, Duration=7ms)
```

使用 timer trigger 要記得啟動 docker 時帶入 AzureWebJobsStorage 的設定值才能正常執行，例如：
```
// 在根目錄加入 dev.env
AzureWebJobsStorage=DefaultEndpointsProtocol=https;AccountName=myazfunctions;AccountKey=ONDu3VvJpg;EndpointSuffix=core.windows.net"

// 執行 docker 時加入 env file 
docker run -it --rm -p 8080:80 --env-file ./dev.env <docker ID>/my_az_functions_image:latest
```

# 上傳 Docker Image 到 Docker Hub
`docker push <docker ID>/my_az_functions_image:latest `
上傳完畢後，可以從 Docker Hub 看到上傳的內容。

# 建立 Azure Function
- 選擇 Docker-Container
  ![](1_sZr48yiG9sLcMsHJoeNFcw.png)
- 建立 Storage account, 選擇 Linux 與 App service Plan(要 premium 喔)
  ![](1_6GJoxBMqhhuNLP15Sa2WtQ.png)
  也可以[利用 CLI 的方式建立 Azure function 與對應的 resource group](https://docs.microsoft.com/zh-tw/azure/azure-functions/functions-create-function-linux-custom-image?tabs=in-process%2Cbash%2Cazure-cli&pivots=programming-language-javascript#create-supporting-azure-resources-for-your-function)。

# 到 Azure functions 的 portal 把必要環境變數加入
![](1_a2vR3iXjYmn0RS0DZ9HaGQ.png)
詳細設定可參考 [Configure apps in the portal — Azure App Service](https://docs.microsoft.com/en-us/azure/app-service/configure-common) 與 [JavaScript developer reference for Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-node?tabs=v2#environment-variables)。

# 部署到 Azure functions
`az functionapp create --name <APP_NAME> --storage-account <STORAGE_NAME> --resource-group <RESOURCE_GROUP> --plan myPremiumPlan --deployment-container-image-name <DOCKER_ID>/azurefunctionsimage:v1.0.0`

其中的 <App_Name>、<RESOURCE_GROUP>、<STORAGE_NAME>、<DOCKER_ID>請已上面步驟建立的值做更換。

# 設定 Azure functions 使用的 Storage connection
設定 Storage connection 可參考下面的指令，或是使用步驟 4 的介紹從 Azure Portal 上進行設定。
Storage connection 的值必須設定在 AzureWebJobsStorage。
```
// 取得 Stroage connection
az storage account show-connection-string --resource-group <RESOURCE_GROUP> --name <STORAGE_NAME> --query connectionString --output tsv

// 設定 Azure function 使用的 Storage connection
az functionapp config appsettings set --name <APP_NAME> --resource-group <RESOURCE_GROUP> --settings AzureWebJobsStorage=<CONNECTION_STRING>
```

# 確認 Azure Function 是否正確被執行
在步驟 2 介紹使用 HTTP trigger，所以可透過下面的連結來做測試
`https://<APP_NAME>.azurewebsites.net/api/{http trigger name}?name=pou`


以上是紀錄把 Azure Function 用 Docker Image 包裝起來的方法。
如果大家有遇到其他問題歡迎一起討論。

# Reference
- [Quickstart: Create a JavaScript function in Azure using Visual Studio Code](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-node)
- [Create a function on Linux using a custom container](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-function-linux-custom-image?tabs=in-process%2Cbash%2Cazure-cli&pivots=programming-language-javascript)
- [Azure Functions base Images](https://hub.docker.com/_/microsoft-azure-functions-base?tab=description) & [Github](https://github.com/Azure/azure-functions-docker)
- [Deploy a Docker Container to Azure Functions using an Azure DevOps YAML Pipeline](https://www.programmingwithwolfgang.com/deploy-docker-container-azure-functions/)
- [App settings reference for Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-app-settings#azurewebjobsscriptroo)
- [Override host.json values](https://docs.microsoft.com/en-us/azure/azure-functions/functions-host-json#override-hostjson-values)