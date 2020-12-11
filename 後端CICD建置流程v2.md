---
title: 後端CICD建置流程v2
tags: 技術學習
---

# 後端CICD建置流程v2



## 情境說明:
1. 本流程是透過Docker and Jenkins實現，原因後續會說明
2. 為什麼需要Docker?

    A. 一般將程式碼佈署在server都會建立個別虛擬環境，此時若有N個以相同程式碼為基底的project，可能要建立N個虛擬環境，較耗費硬體資源

    B. 相對於虛擬環境，若使用Docker佈署程式碼，只要使用一個image(映像檔)即可佈署在N個containers，這些containers共用同一個Linux kernel, OS system，較節省硬體資源
    ![docker vs venv](./images/img1.jpg)
    ![](https://i.imgur.com/4Ssg3ZT.jpg)

    [reference](https://www.serverpronto.com/spu/2016/05/containers-vs-virtual-machines-vms-is-there-a-clear-winner/)
2. 為什麼需要Jenkins?
    A. 1234
    B. 5678
4. 假設我們已安裝好docker
5. 以Solar dashboard專案為例，示範以下流程(已在Linux環境測試成功)

    A. 建立Dockerfile
    
    B. 建立Jenkinsfile

    C. 根據Jenkinsfile紀錄的指令，將code佈署到指定的server

------------
## 建立Dockerfile

請先建立一個檔案，命名為Dockerfile(沒有副檔名)
```dockerfile=
# Use an local Python with taipei time as a parent image
FROM roykuo/python:taipei_time

# Set the container working directory to /opt/SolarPlatForm_Backend
# if no such dir, create new one automatically
WORKDIR /opt/Traffic_lights

# Copy the directory where Dockerfile at, into the container at /opt/SolarPlatForm_Backend
COPY  . /opt/Traffic_lights

# setting crontab
ADD solar_cron /etc/cron.d/crontab
RUN chmod 0644 /etc/cron.d/crontab
RUN touch /var/log/cron.log
RUN mkdir /run/Traffic_lights/

# Install any needed packages specified in requirements.txt
RUN python -m pip install --trusted-host pypi.python.org --upgrade pip
RUN python -m pip install --trusted-host pypi.python.org --upgrade setuptools
RUN pip install --trusted-host pypi.python.org -r requirements.txt
RUN apt-get update
RUN apt-get -y install iputils-ping
RUN apt-get -y install vim
RUN apt-get -y install procps
RUN apt-get -y install cron

# Run crontab and app.py when the container launches
CMD /usr/sbin/cron -f
```
--------------------
依序說明Dockerfile內每個步驟的效果，以下將Dockerfile所在的位置稱為"外面"，要建立的image目錄位置稱為"裡面"。

```dockerfile
FROM roykuo/python:taipei_time
```
>以修改後的python3.7.4輕量版為image基底(時區設為台北)

```dockerfile
WORKDIR /opt/Traffic_lights
```
>將/opt/SolarPlatForm_Backend設為image裡面的目錄，若沒有則會新增之

```dockerfile
COPY  . /opt/Traffic_lights
```
>將外面的所有檔案複製到裡面的/opt/SolarPlatForm_Backend路徑

```dockerfile=
ADD solar_cron /etc/cron.d/crontab
RUN chmod 0644 /etc/cron.d/crontab
RUN touch /var/log/cron.log
```
>此步驟是將外面的solar_cron檔案新增到裡面的/etc/cron.d/並命名為: crontab，先不贅述

```dockerfile=
RUN mkdir /run/SolarPlatForm/
```
>此為dashboard執行時，紀錄pid以免程式重複執行的資料夾 先不贅述

```dockerfile=
RUN python -m pip install --trusted-host pypi.python.org --upgrade pip
RUN python -m pip install --trusted-host pypi.python.org --upgrade setuptools
RUN pip install --trusted-host pypi.python.org -r requirements.txt
RUN apt-get update
RUN apt-get -y install iputils-ping
RUN apt-get -y install vim
RUN apt-get -y install procps
RUN apt-get -y install cron
```
>安裝image裡面的必要套件，包括:
>>1. python專案的所有套件(紀錄於requirements.txt)
>>2. linux的ping, vim and cron程式

```dockerfile
CMD /usr/sbin/cron -f
```

>image打包成功並運行在container時，要執行的指令

------------

## 建立Jenkinsfile

請先建立一個檔案，命名為Jenkinsfile(沒有副檔名)
```Jenkinsfile=
def VERSION_NUMBER
def BUILD_DATE = new Date().format("yyyyMMdd")
def BUILD_NUMBER = env.BUILD_NUMBER

pipeline {
    agent any
    environment {
        DEV_SERVER_IP = credentials("Dev_Server_IP")
        STAGE_SERVER_IP  = credentials("Stage_Server_IP")
	HARBOR_IP = credentials("harbor_ip")
        HARBOR_PROJECT_NAME = "ems"
        SERVER_CONTAINER_NAME = "solar_server"
        DASHBOARD_CONTAINER_NAME = "solar_latest_dashboard"
        IMAGE_NAME = "solar_dashboard"
	GPG_PASSPHRASE = credentials("gpg-passphrase")
    }
    stages {
        stage("Test") {
            agent { docker { image "roykuo/python:taipei_time" } }
            steps {
                sh "python -m venv venv"
                sh ". venv/bin/activate"
                sh "python -m pip install --upgrade pip"
                sh "python -m pip install --upgrade setuptools"
                sh "pip install -r requirements.txt"
                sh "python -m unittest discover ./test"
            }
        }
        stage("Generate Dev Version Number") {
            when {
                expression {
                    env.GIT_BRANCH == "origin/develop"
                }
            }
            steps {
                script {
                   VERSION_NUMBER = "v." + BUILD_DATE + "_dev_" + BUILD_NUMBER
                }
                echo "$VERSION_NUMBER"
            }
        }
        stage("Generate Stage Version Number") {
            when {
                expression {
                    env.GIT_BRANCH == "origin/master"
                }
            }
            steps {
                script {
                   VERSION_NUMBER = "v." + BUILD_DATE + "_stage_" + BUILD_NUMBER
                }
                echo "$VERSION_NUMBER"
            }
        }

        stage("Setting .env For Dev Server and build image") {
            when {
                expression {
                    env.GIT_BRANCH == "origin/develop"
                }
            }
            steps {
                sh "ls -al"
                sh "git secret reveal -p $GPG_PASSPHRASE"
                sh "ls -al"

                // remove all .env files except .env.Dev
                sh "rm `ls .env* | grep -v 'env.Dev'`"

                // rename file so that we don't need to edit the .env name
                sh "mv .env.Dev .env"
                sh "echo 'BACKEND_VERSION_NUMBER=$VERSION_NUMBER' >> .env"
                sh "cat .env"
                sh "docker build -t $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER} ."
            }
        }
        stage('Setting .env For Stage Server and build image') {
            when {
                expression {
                    env.GIT_BRANCH == "origin/master"
                }
            }
            steps {
                sh "ls -al"
                sh "git secret reveal -p $GPG_PASSPHRASE"
                sh "ls -al"

                // remove all .env files except .env.Stage
                sh "rm `ls .env* | grep -v '.env.Stage'`"

                // rename file so that we don't need to edit the .env name
                sh "mv .env.Stage .env"
                sh "echo 'BACKEND_VERSION_NUMBER=$VERSION_NUMBER' >> .env"
                sh "cat .env"
                sh "docker build -t $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER} ."
            }
        }

        stage("Login Registry & Push Image") {
	    when {
                expression {
                    env.GIT_BRANCH == "origin/develop" || env.GIT_BRANCH == "origin/master"
                }
            }
            steps {
                withCredentials([usernamePassword(
                                    credentialsId: "jenkins_in_harbor",
                                    usernameVariable: "USERNAME",
                                    passwordVariable: "PASSWORD")]) {
                    sh "docker login $HARBOR_IP -u $USERNAME -p $PASSWORD"
                    sh "docker push $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER}"
                }
            }
        }

        stage("Deploy on Dev Server") {
	    when {
                expression {
                    env.GIT_BRANCH == "origin/develop"
                }
            }
            steps {
                withCredentials([usernamePassword(
                                    credentialsId: "dev_server_in_harbor",
                                    usernameVariable: "USERNAME",
                                    passwordVariable: "PASSWORD")]) {
                    sh """
                        ssh -tt solarplatform@$DEV_SERVER_IP  -p 52100 << EOF
                        docker login $HARBOR_IP -u $USERNAME -p $PASSWORD
                        ls
                        pwd
                        docker rm -f solar_latest_dashboard
                        docker rm -f solar_server
                        docker run -d -p 4000:80 --rm -v /var/docker/solar_latest_dashboard_log:/opt/SolarPlatForm_Backend/log --name $DASHBOARD_CONTAINER_NAME --network solar_bridge $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER}
                        docker run -d -p 53001:80 --rm -v /var/docker/solar_server_log:/opt/SolarPlatForm_Backend/log --name $SERVER_CONTAINER_NAME --network solar_bridge $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER}
                        docker exec -itd $DASHBOARD_CONTAINER_NAME crontab /etc/cron.d/crontab
                        docker exec -itd $SERVER_CONTAINER_NAME python app.py
                        exit
                    EOF"""
                }
            }
        }
        stage("Deploy on Stage Server") {
	    when {
                expression {
                    env.GIT_BRANCH == "origin/master"
                }
            }
            steps {
                withCredentials([usernamePassword(
                                    credentialsId: "stage_server_in_harbor",
                                    usernameVariable: "USERNAME",
                                    passwordVariable: "PASSWORD")]) {
                    sh """
                        ssh -tt solarplatform@$DEV_SERVER_IP  -p 52100 << EOF
                        docker login $HARBOR_IP -u $USERNAME -p $PASSWORD
                        ls
                        pwd
                        docker rm -f solar_latest_dashboard
                        docker rm -f solar_server

                        docker run -d -p 4000:80 --rm -v /var/docker/solar_latest_dashboard_log:/opt/SolarPlatForm_Backend/log --name $DASHBOARD_CONTAINER_NAME --network solar_bridge $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER}
                        docker run -d -p 53001:80 --rm -v /var/docker/solar_server_log:/opt/SolarPlatForm_Backend/log --name $SERVER_CONTAINER_NAME --network solar_bridge $HARBOR_IP/$HARBOR_PROJECT_NAME/solar_dashboard:${VERSION_NUMBER}
                        docker exec -itd $DASHBOARD_CONTAINER_NAME crontab /etc/cron.d/crontab
                        docker exec -itd $SERVER_CONTAINER_NAME python app.py
                        exit
                    EOF"""
                }
            }
        }
    }
    post {
        always {
            cleanWs()
            dir("${env.WORKSPACE}@tmp") {
                deleteDir()
            }
            dir("${env.WORKSPACE}@script") {
                deleteDir()
            }
            dir("${env.WORKSPACE}@script@tmp") {
                deleteDir()
            }
            dir("${env.WORKSPACE}@2") {
                deleteDir()
            }
	    dir("${env.WORKSPACE}@2@tmp") {
                deleteDir()
            }
        }
    }
}
```

由於內容較多，以下僅針對程式段落做說明

```Jenkinsfile
def VERSION_NUMBER
def BUILD_DATE = new Date().format("yyyyMMdd")
def BUILD_NUMBER = env.BUILD_NUMBER
```
宣告三個變數，用於儲存版本資訊: date + build_number

```Jenkinsfile
pipeline {
    ...
}
```
pipeline記錄所有要讓Jenkins代勞的所有流程

```Jenkinsfile
environment {
    ...
}
```
environment紀錄整個流程會用到的變數
可自行定義，若為敏感資訊，須預先設定好後透過credentials(...)引用
[詳情參閱](前端如何撰寫Jenkins執行腳本)

```Jenkinsfile
stages {
        stage("Test") {
        ...
            }
        stage("Generate Dev Version Number") {
        ...
            }
            
```
stages and stage是一個巢狀結構，stages記錄各個stage要執行的動作，stage("test")、stage("Generate Dev Version Number")除了可以凸顯各stage的目的以外，也會顯示在Jenkins執行中的dashboard畫面
[詳情參閱](前端如何撰寫Jenkins執行腳本)

此外stage之間均為獨立container個體(need confirm)

```Jenkinsfile
steps {
    sh "git secret reveal -p $GPG_PASSPHRASE"
    sh "rm `ls .env* | grep -v 'env.Dev'`"
    
    sh """
        ssh -tt solarplatform@$DEV_SERVER_IP  -p 52100 << EOF
        docker login $HARBOR_IP -u $USERNAME -p $PASSWORD
                        
      EOF"""
}
```
steps可以記錄該stage要執行哪些流程，支援shell script(sh)，可依需求自行選擇格式

```Jenkinsfile
when {
        expression {
                    env.GIT_BRANCH == "origin/develop"
                }
     }
```
條件表達式，若條件成立才執行該stage後續動作


### 事前準備
1. 請先clone專案並切換到demo branch:
    ```=
    git clone https://lighting.acbel.com:8020/gitea/SIT/SolarPlatForm_Backend.git
    git checkout 4655_demo_backend_CICD
    ```
2. 根據路徑建立以下資料夾:
    * /var/docker/redis_data
    * /var/docker/solar_latest_dashboard_log
    * /var/docker/solar_server_log

3. 建立containers之間的溝通管道
    `sudo docker network create solar_bridge --ip-range 192.168.144.0/24 --subnet=192.168.144.0/24 --gateway=192.168.144.254`
5. 啟動Redis服務
    ```=
    sudo docker pull redis:alpine3.12
    sudo docker run -d -v /var/docker/redis_data:/data --rm --name AcBel_redis --network solar_bridge redis:alpine3.12
    ```

### 佈署:

code有修改
- git push / merge 到develop

code未修改
- 前往[Jenkins dashboard](10.10.111.242:53101/)
- 登入 (帳號密碼跟系統管理員索取)
- 前往指定的repository
    - 這裡以SolarPlatForm_Backend為例
    ![](https://i.imgur.com/GTQwL9q.png)
- 指定要建置的branch(佈署的server目的地定義在Jenkinsfile)
    - ![](https://i.imgur.com/2FvIiNL.png)
    - ![](https://i.imgur.com/5IImnF6.png)

監控執行過程與結果
- ![](https://i.imgur.com/dlssrfX.png)
- 若出現Success，表示成功，反之則失敗
    - ![](https://i.imgur.com/RDlakQ8.png)

