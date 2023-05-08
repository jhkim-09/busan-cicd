# [0] 가상머신 생성
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "jenkins" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.hostname = "jenkins"
    config.vm.network "private_network", ip: "192.168.56.10"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "jenkins"
      vb.memory = "4096"
      vb.cpus = 2
    end
  end
  config.vm.provision "shell", inline: <<-SHELL
    sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config
    systemctl restart ssh
    sed -i 's/archive.ubuntu.com/ftp.daum.net/g' /etc/apt/sources.list
    sed -i 's/security.ubuntu.com/ftp.daum.net/g' /etc/apt/sources.list
  SHELL
end
```
```powershell
PS C:\Users\SBAUser\cicd> vagrant up
PS C:\Users\SBAUser\cicd> vagrant ssh jenkins
```
AWS 인스턴스로 대체 가능

# [1] built-in 노드 준비

## 1. java 설치
젠킨스가 동작하기 위해서는 자바가 필요
```bash
root@jenkins:~# apt update
root@jenkins:~# apt install openjdk-11-jre -y
root@jenkins:~# java -version
openjdk version "11.0.18" 2023-01-17
OpenJDK Runtime Environment (build 11.0.18+10-post-Ubuntu-0ubuntu120.04.1)
OpenJDK 64-Bit Server VM (build 11.0.18+10-post-Ubuntu-0ubuntu120.04.1, mixed mode, sharing)
```
jre < jdk (좀 더 큰 범위)

## 2. jenkins 설치
```bash
root@jenkins:~# curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key > /usr/share/keyrings/jenkins-keyring.asc
root@jenkins:~# file /usr/share/keyrings/jenkins-keyring.asc
/usr/share/keyrings/jenkins-keyring.asc: PGP public key block Public-Key (old)
root@jenkins:~# echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list
root@jenkins:~# file /etc/apt/sources.list.d/jenkins.list
/etc/apt/sources.list.d/jenkins.list: ASCII text
root@jenkins:~# cat /etc/apt/sources.list.d/jenkins.list
deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/
root@jenkins:~# apt update
root@jenkins:~# apt install jenkins -y
```
만약 노드를 추가로 사용하는 경우에 젠킨스는 컨트롤러에만 설치

## 3. 대시보드 접속 및 초기구성
### 1) 브라우저로 젠킨스 서버 접속 (ex. 192.168.56.10:8080)
### 2) 잠금해제 및 플러그인 설치
```bash
root@jenkins:~# cat /var/lib/jenkins/secrets/initialAdminPassword
b3ff6b9fc5184e239a5da85b0bab715a
```
위와 같이 확인한 초기 암호로 잠금설정 해제 및 기본 플러그인들 설치
### 3) 로그인 계정 생성

# [2] built-in 방식의 프로젝트 생성 실습
## 1. 프리스타일 프로젝트
### 1) 대시보드를 통해 명령어 실행해보기
1. new item 클릭
2. freestyle project 선택 및 이름 입력
3. build steps에서 Execute shell 항목 선택
4. 실행할 명령어 입력  
    `echo "hello world" > hello.txt`

### 2) 빌드 후 콘솔로 확인
1. 지금빌드 클릭으로 빌드
2. 빌드 히스토리에서 해당 빌드넘버 클릭 후 consol output 확인
### 3) 결과물 확인하기
출력 메시지에서 workspace 경로 확인 후 해당 위치에서 파일 확인  
(기본 작업디렉토리 경로 : /var/lib/jenkins/workspace/PROJECTNAME)
## 2. 메이븐 프로젝트
### 1) 로컬 파일로 빌드하기
#### (1) 패키지 설치
```bash
root@jenkins:~# apt install maven
```
#### (2) 글로벌 설정
1. 좌측 탭에서 Jenkins 관리 클릭
2. System Configuration > Global Tool Configuration 선택
3. JDK 항목에서 Add JDK 버튼 클릭 후 값 입력
    - Name : java-11  
    - JAVA_HOME : /usr/lib/jvm/java-11-openjdk-amd64/  
    - Install automatically 체크박스 해제
4. Maven  항목에서 Add Maven 버튼 클릭 후 입력
    - Name : Maven-3  
    - MAVEN_HOME : /usr/share/maven  
    - Install automatically 체크박스 해제
5. save 버튼 클릭
#### (3) 플러그인 설정
1. 좌측 탭에서 Jenkins 관리 클릭
2. System Configuration > 플러그인 관리 클릭
3. Available plugins 선택
4. Maven Integration 체크
5. Install without restart 버튼 클릭
#### (4) 프로젝트 생성
1. new item 클릭
2. Maven project 선택 및 이름 입력
3. Pre Steps 를 Execute shell 선택 후 명령어 입력  
cp -r /tmp/source-maven-java-spring-hello-webapp/* /var/lib/jenkins/workspace/second
4. Build 항목에 값 입력
    - Root POM : pom.xml
    - Goals and options : clean package
5. 빌드 후 확인
* 참고  
프로젝트 빌드 작업 시 작업디렉토리는 자동으로 생성됨  
빌드 시 사용할 파일(pom.xml)은 해당 작업디렉토리 기준으로 위치 지정

### 2) github 으로 빌드하기
#### (1) 깃 회원가입 및 로그인
#### (2) ssh키 설정
```bash
root@jenkins:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:coRM9r1LpQd3+Qs4RtrpKPDAO/Y0O2a7egefqf94WqY root@jenkins
The key's randomart image is:
+---[RSA 3072]----+
|      o          |
|     + o .     . |
|      o o o.o o  |
|    .  .  +*o. . |
|     +. S.+*..  .|
|      =+ .+o. . .|
|     + =o.++   . |
|    . +=+==.     |
|     .=*BE+.     |
+----[SHA256]-----+
root@jenkins:~# ls -l ~/.ssh
total 8
-rw------- 1 root root    0 Apr 11 00:01 authorized_keys
-rw------- 1 root root 2602 Apr 11 03:15 id_rsa
-rw-r--r-- 1 root root  566 Apr 11 03:15 id_rsa.pub
```
#### (3) 깃헙 홈페이지에서 추가 설정
    setting > Access > SSH and GPG keys > New SSH key
    이름 및 공개키의 값 입력
#### (4) 깃 저장소 생성 및 업로드
```bash
root@jenkins:~/test-cicd# ls -a
.  ..  pom.xml  src
root@jenkins:~/test-cicd# git init
Initialized empty Git repository in /root/test-cicd/.git/
root@jenkins:~/test-cicd# ls -a
.  ..  .git  pom.xml  src
root@jenkins:~/test-cicd# git config --global user.name jhkim-09
root@jenkins:~/test-cicd# git config --global user.email juhyokim@nobreak.co.kr
root@jenkins:~/test-cicd# git add .
root@jenkins:~/test-cicd# git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   pom.xml
        new file:   src/main/java/com/example/web/WebInitializer.java
        new file:   src/main/java/com/example/web/config/SpringConfig.java
        new file:   src/main/java/com/example/web/controller/WelcomeController.java
        new file:   src/main/resources/logback.xml
        new file:   src/main/webapp/WEB-INF/views/index.jsp
        new file:   src/test/java/com/example/web/TestWelcome.java

root@jenkins:~/test-cicd# git commit -m "first commit"
On branch master
nothing to commit, working tree clean
root@jenkins:~/test-cicd# git branch -M main
root@jenkins:~/test-cicd# git remote add origin git@github.com:jhkim-09/test.git
root@jenkins:~/test-cicd# git push -u origin main
Enumerating objects: 26, done.
Counting objects: 100% (26/26), done.
Delta compression using up to 2 threads
Compressing objects: 100% (13/13), done.
Writing objects: 100% (26/26), 3.73 KiB | 477.00 KiB/s, done.
Total 26 (delta 0), reused 0 (delta 0)
To github.com:jhkim-09/test.git
 * [new branch]      main -> main
Branch 'main' set up to track remote branch 'main' from 'origin'.
```
#### (5) 깃을 이용한 Maven 프로젝트 생성
    1. Maven project 선택 후 소스 코드 관리를 git 으로 선택
    2. 저장소 URL 확인 후 입력
    3. branch 는 */main 으로 변경
    4. 나머지 동일

# [3] 노드 추가
## 1. vagrant 등을 이용해 가상머신 추가 배포
## 2. 해당 가상머신에 필요한 패키지 설치
```bash
root@node1:~# apt install openjdk-11-jre -y
root@node1:~# apt install maven
```
자바는 필수
나머지 빌드 관련 패키지들은 필요에 따라 추가 설치

## 3. 사용자 설정
```bash
root@node1:~# useradd jenkins -d /var/lib/jenkins -m
root@node1:~# ls -ld /var/lib/jenkins
drwxr-xr-x 2 jenkins jenkins 4096 Apr 13 03:29 /var/lib/jenkins
root@node1:~# grep jenkins /etc/passwd
jenkins:x:1002:1002::/var/lib/jenkins:/bin/sh
root@node1:~# passwd jenkins
New password:
Retype new password:
passwd: password updated successfully
```
사용자 이름과 경로는 중요하지 않음(기존의 사용자도 가능)

## 4. 대시보드에서 노드 추가
1. Jenkins 관리 탭에서 노드 관리 항목 선택
2. +New Node 버튼 클릭
3. 노드 이름 입력 및 타입 선택
4. 세부설정 항목 입력
    - Number of executors : 2 (CPU 개수 기준으로 선택)
    - Remote root directory : /var/lib/jenkins (이전 작업에서 준비한 사용자의 홈디렉토리)
    - Launch method : Launch agents via SSH 로 선택
        - Host : 192.168.56.11 (준비한 노드의 주소)
        - Credentials : Add 버튼 클릭 후 항목 입력
            - kind : Username with password
            - ID : node-ssh (젠킨스에서 사용할 해당 자격증명에 대한 명칭)
            - Username : jenkins (준비한 사용자이름)
            - Password : 해당 사용자의 패스워드 입력
        - Credentials 에서 새로 생성한 항목 선택 (jenkins/***)

## 5. Built-in Node 비활성화
1. Jenkins 관리 > Node관리
2. Built-in Node 설정
3. Number of executors 를 0 으로 변경

# [4] 별도의 노드에서의 프로젝트 빌드
## 1. Maven 프로젝트 생성 실습
상동
## 2. 톰캣 서버에 배포하기
### 1) 패키지설치
```bash
root@tomcat:~# apt update
root@tomcat:~# apt install tomcat9 tomcat9-admin -y
```
### 2) tomcat 설정
```bash
root@tomcat:~# vim /etc/tomcat9/tomcat-users.xml
root@tomcat:~# systemctl restart tomcat9.service
```
/etc/tomcat9/tomcat-users.xml 파일 편집 내용
```xml
<tomcat-users ...>
...
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="manager-jmx"/>
  <role rolename="manager-status"/>
  <user username="admin" password="1" roles="manager-gui, manager-script, manager-jmx, manager-status"/>
</tomcat-users>
```
### 3) 플러그인 추가
    jenkins 관리 > Plugin Manager > Available plugins
    Deploy to container 선택 및 설치
### 4) 자격증명 추가
1. jenkins 관리 > manage credentials 선택
2. Domains 항목 선택 및 Add credentials 버튼 클릭
    - kind : Username with password
    - ID : tomcat-manager (젠킨스에서 사용할 해당 자격증명에 대한 명칭)
    - Username : admin (xml파일 참고)
    - Password : 1 (xml파일 참고)
### 5) 프로젝트 생성
1. 새로운 아이템 항목 선택
2. Maven 프로젝트 선택 및 프로젝트 이름 입력
3. 빌드 후 조치 항목에서 Deploy war/ear to a container 선택
    - WAR/EAR files : target/hello-world.war
    - Containers : tomcat 9.x 선택
        - credentails : 이전 설정 항목 선택
        - tomcat URL : http://192.168.56.12:8080 (http 방식으로 입력)        
4. 나머지 설정 내용은 상동
### 6) 확인
    http://192.168.56.12:8080/hello-world 로 접속 후 확인

## 3. 파이프라인
### 1) 프로젝트 생성
    새로운 item > Pipeline 항목 선택 및 이름 입력
### 2) Jenkinsfile 작성
```Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git', url: 'https://github.com/jhkim-09/test.git'
            }
        }
        stage('Build') {
            steps {
                sh "mvn -Dmaven.test.failure.ignore=true clean package"
            }
        }
        stage('Deploy') {
            steps {
                deploy adapters: [tomcat9(credentialsId: 'tomcat-manager', path: '', url: 'http://192.168.56.12:8080')], contextPath: null, war: 'target/hello-world.war'
            }
        }
    }
}
```
### 3) 빌드 및 확인

## 4. pipeline + git
### 1) Jenkinsfile 작성 및 업로드
```bash
root@jenkins:~/test-cicd# vim Jenkinsfile
root@jenkins:~/test-cicd# git add .
root@jenkins:~/test-cicd# git commit -m "Create jenkinsfile"
root@jenkins:~/test-cicd# git push -u origin main
```
### 2) 프로젝트 생성
- Pipeline 에 Definition 항목을 Pipeline script from SCM 으로 변경
    - SCM : Git
        - URL 입력 및 브랜치 지정

## 5. pipeline 에서 노드 선택하기
### 1) 노드 설정 변경
- Jenkins 관리 > 노드 관리
    - Built-in Node > 설정
        Number of executors : 2
        라벨 : jenkins-controller
    - node1 > 설정
        Labels : jenkins-node
### 2) Jenkinsfile 수정
```Jenkinsfile
pipeline {
    agent {
        label "jenkins-node"
    }
}
```

# [5] 도커에서 사용
## 1. 가상머신 생성
## 2. 도커 설치
```bash
vagrant@jen-docker:~$ sudo apt-get update
vagrant@jen-docker:~$ sudo apt-get install ca-certificates curl gnupg -y
vagrant@jen-docker:~$ sudo install -m 0755 -d /etc/apt/keyrings
vagrant@jen-docker:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
vagrant@jen-docker:~$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
vagrant@jen-docker:~$ echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
vagrant@jen-docker:~$ sudo apt-get update
vagrant@jen-docker:~$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
vagrant@jen-docker:~$ sudo usermod -aG docker $USER
vagrant@jen-docker:~$ exit
```

## 3. 수동으로 tomcat 빌드하기
### 1) Dockerfile 작성
```bash
vagrant@jen-docker:~$ ls
vagrant@jen-docker:~$ mkdir ~/static
vagrant@jen-docker:~$ cd ~/static
vagrant@jen-docker:~/static$ vim Dockerfile
vagrant@jen-docker:~/static$ cat Dockerfile
FROM    maven:3-openjdk-8 AS mbuilder
RUN     mkdir /hello
RUN     git clone https://github.com/jhkim-09/test.git /hello
WORKDIR /hello
RUN     mvn package

FROM    tomcat:9-jre8
COPY    --from=mbuilder /hello/target/hello-world.war /usr/local/tomcat/webapps/
```
### 2) 이미지 생성 및 업로드
```bash
vagrant@jen-docker:~/static$ docker image build -t kimjuhyo/myhello:v1 .
vagrant@jen-docker:~/static$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: kimjuhyo
Password:
WARNING! Your password will be stored unencrypted in /home/vagrant/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
vagrant@jen-docker:~/static$ docker image tag myhello:latest kimjuhyo/myhello:v1
vagrant@jen-docker:~/static$ docker image push kimjuhyo/myhello:v1
The push refers to repository [docker.io/kimjuhyo/myhello]
...
```
### 3) 컨테이너 실행 및 확인
```bash
vagrant@jen-docker:~/static$ docker container run -d -p 80:8080 --name myweb kimjuhyo/myhello:v1
fba0ccc11dc82122d5e4c0f2cd8bc034ed5b7c8eaa060e59d34070332168571b
vagrant@jen-docker:~/static$ docker ps
CONTAINER ID   IMAGE                 COMMAND             CREATED         STATUS         PORTS                                   NAMES
fba0ccc11dc8   kimjuhyo/myhello:v1   "catalina.sh run"   6 seconds ago   Up 5 seconds   0.0.0.0:80->8080/tcp, :::80->8080/tcp   myweb

vagrant@jen-docker:~/static$ curl http://localhost/hello-world/

<html>
<head>
<title>Hello World</title>
</head>
<body>
<h1>Hello World</h1>
<h2>Today is 2023-04-19</h2>
<h3>Version: 1.0</h3>
</body>
</html>
```
* 후행 슬러시(/)까지 반드시 입력한다.

## 4. 도커에 젠킨스 설치
### 1) 작업디렉토리 준비
```bash
vagrant@jen-docker:~/static$ mkdir ~/jenkins
vagrant@jen-docker:~/static$ cd ~/jenkins
vagrant@jen-docker:~/jenkins$ pwd
/home/vagrant/jenkins
```
### 2) 도커 네트워크 및 DinD 생성
```bash
vagrant@jen-docker:~/jenkins$ docker network create jenkins
cd208c5d83f4cb76eead08a7bdcab28e83241af8e000effb406821559277abf0
vagrant@jen-docker:~/jenkins$ docker container run --name docker-dind -d --privileged --network jenkins --network-alias docker -e DOCKER_TLS_CERTDIR=/certs -v jenkins-docker-certs:/certs/client -v jenkins-data:/var/jenkins_home -v docker:/var/lib/docker -p 2376:2376 --restart=always docker:dind --storage-dirver overlay2
Unable to find image 'docker:dind' locally
dind: Pulling from library/docker
...
vagrant@jen-docker:~/jenkins$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS                  PORTS                                                 NAMES
2147e77c8e4c   docker:dind           "dockerd-entrypoint.…"   7 seconds ago   Up Less than a second   2375/tcp, 0.0.0.0:2376->2376/tcp, :::2376->2376/tcp   docker-dind
fba0ccc11dc8   kimjuhyo/myhello:v1   "catalina.sh run"        9 minutes ago   Up 9 minutes            0.0.0.0:80->8080/tcp, :::80->8080/tcp                 myweb
vagrant@jen-docker:~/jenkins$ docker container inspect docker-dind | grep -A3 -i cmd
            "Cmd": [
                "--storage-dirver",
                "overlay2"
            ],
```
### 3) 도커파일 작성
```bash
vagrant@jen-docker:~/jenkins$ vim Dockerfile
vagrant@jen-docker:~/jenkins$ cat Dockerfile
FROM    jenkins/jenkins:lts-jdk11
USER    root
RUN     apt-get update && apt-get install -y lsb-release
RUN     curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg
RUN     echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
        https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN     apt-get update && apt-get install -y docker-ce-cli
USER    jenkins
RUN     jenkins-plugin-cli --plugins "docker-plugin docker-workflow"
```
### 4) 이미지 빌드 및 컨테이너 실행
```bash
vagrant@jen-docker:~/jenkins$ docker image build -t jenkins-docker:lts-jkd11 .
vagrant@jen-docker:~/jenkins$ docker container run --name jenkins-docker -d --network jenkins -e DOCKER_HOST=tcp://docker:2376 -e DOCKER_CERT_PATH=/certs/client -e DOCKER_TLS_VERIFY=1 -v jenkins-data:/var/jenkins_home -v jenkins-docker-certs:/certs/client:ro -p 8080:8080 -p 50000:50000 --restart=always jenkins-docker:lts-jkd11
3d09e4ce85cc952d6373e79e7f043a3e957ad3c338a165d55abd5598d2971f04
vagrant@jen-docker:~/jenkins$ docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS                          PORTS
                                                                         NAMES
3d09e4ce85cc   jenkins-docker:lts-jkd11   "/usr/bin/tini -- /u…"   4 seconds ago    Up 3 seconds                    0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp   jenkins-docker
2147e77c8e4c   docker:dind                "dockerd-entrypoint.…"   16 minutes ago   Restarting (1) 19 seconds ago
                                                                         docker-dind
fba0ccc11dc8   kimjuhyo/myhello:v1        "catalina.sh run"        25 minutes ago   Up 25 minutes                   0.0.0.0:80->8080/tcp, :::80->8080/tcp                                                      myweb
```
### 5) 대시보드 접속
#### (1) 초기 설정
```bash
vagrant@jen-docker:~/jenkins$ docker container exec jenkins-docker cat /var/jenkins_home/secrets/initialAdminPassword
```
대시보드 접속은 해당 시스템의 8080포트로 접속
접속 후 초기화를 위한 인증은 위와 같이 파일 내용을 확인

#### (2) 에이전트 구성(노드설정)
    Jenkins 관리 > 노드 관리 > Configure Clouds
    1. Add a new cloud 항목 docker 로 선택
    2. Name : docker
        Docker Cloude details 항목에서
        1) Docker Host URI : tcp://docker:2376
        2) Server credentials 에서 add 선택 후
            (1) Kind 항목 X.509 로 선택 후 Client Key / Client Certificate / Server CA Certificate 항목들 입력
            항목들은 다음 명령어로 확인
            $ docker container exec jenkins-docker cat /certs/client/key.pem
            $ docker container exec jenkins-docker cat /certs/client/cert.pem
            $ docker container exec jenkins-docker cat /certs/client/ca.pem
            (2) ID : docker-client-certs
            (3) ADD 버튼 클릭
        3) 생성한 항목으로 선택 후 Test Connection 버튼으로 확인
        
## 5. 도커로 파이프라인 테스트
### 1) 예제 직접 입력 or Git 에서 가져오기
#### (1) 에이전트 지정 및 실행(기본)
```Jenkinsfile
pipeline {
  agent {
    docker { image 'node:16-alpine' }
  }
  stages {
    stage('Test') {
      steps {
        sh 'node --version'
      }
    }
  }
}
```
#### (2) 작업 별로 여러 컨테이너 사용
```Jenkinsfile
pipeline {
  agent none
  stages {
    stage('Back-end') {
      agent {
        docker { image 'maven:3.8.1-adoptopenjdk-11' }
      }
      steps {
        sh 'mvn --version'
      }
    }
    stage('Front-end') {
      agent {
        docker { image 'node:16-alpine' }
      }
      steps {
        sh 'node --version'
      }
    }
  }
}
```
#### (3) 이미지 대신 Dockerfile 로 빌드 후 작업하기
```Jenkinsfile
pipeline {
  agent { dockerfile true }
  stages {
    stage('Test') {
      steps {
        sh '''
          node --version
          git --version
          curl --version
        '''
      }
    }
  }
}
```

### 2) tomcat 빌드
#### (1) 도커파일 작성
```Dockerfile
FROM    tomcat:9-jre8
COPY    target/hello-world.war /usr/local/tomcat/webapps/
```
#### (2) 도커허브 토큰 생성
    1. hub.docker.com 접속
    2. 우측상단 계정이름 클릭 후 Account Settings 선택
    3. Security 탭에서 New Access Token 버튼 클릭
    4. 이름 입력 후 생성
    5. 토큰 값 복사해두기
    dckr_pat_7BsO06g1-W6jS9abGMa6KAnCcMM
#### (3) 자격증명 추가
    1. Jenkins 관리 > Manage Credentials > global 클릭
    2. Add Credentails 버튼 클릭
    3. Kind : Username with password 선택 (기본값)
        Username : kimjuhyo (내 도커 계정)
        Password : dckr_pat_7BsO06g1-W6jS9abGMa6KAnCcMM (좀전에 만든 토큰값)
        ID : docker-hub-token (내가 사용할 이름)
#### (4) 도커 데몬 원격 설정
```bash
vagrant@jenkins:~$ sudo systemctl edit docker.service

[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock 
```

```bash
vagrant@jenkins:~$ sudo systemctl daemon-reload
vagrant@jenkins:~$ sudo systemctl restart docker
vagrant@jenkins:~$ sudo ss -nltp | grep docker
LISTEN    0         4096               0.0.0.0:8080             0.0.0.0:*        users:(("docker-proxy",pid=11071,fd=4))                                     
LISTEN    0         4096               0.0.0.0:50000            0.0.0.0:*        users:(("docker-proxy",pid=11051,fd=4))                                     
LISTEN    0         4096                     *:2375                   *:*        users:(("dockerd",pid=10907,fd=3))                                          
LISTEN    0         4096                  [::]:8080                [::]:*        users:(("docker-proxy",pid=11079,fd=4))                                     
LISTEN    0         4096                  [::]:50000               [::]:*        users:(("docker-proxy",pid=11057,fd=4))         
```
#### (5) 사용할 Jenkinsfile 작성
```Jenkinsfile
pipeline {
  agent none

  environment {
    registry = "index.docker.io/v1/"
    imageName = "kimjuhyo/docker-cicd-tomcat-test"
  }

  stages {
    stage('Checkout') {
      agent any
      steps {
        git branch: 'main', url: 'https://github.com/jhkim-09/test.git'
      }
    }
    stage('Build') {
      agent {
        docker { image 'maven:3-openjdk-8' }
      }
      steps {
        sh 'mvn -DskipTests=true clean package'
      }
    }
    stage('Test') {
      agent {
        docker { image 'maven:3-openjdk-8' }
      }
      steps {
        sh 'mvn test'
      }
    }
    stage('Build Docker Image') {
        agent any
        steps {
            sh 'docker image build -t $imageName:$BUILD_NUMBER .'
        }
    }
    stage('Tag Docker Image') {
        agent any
        steps {
            sh 'docker image tag $imageName:$BUILD_NUMBER $imageName:latest'
        }
    }
    stage('Publish Docker Image') {
        agent any
        steps {
            withDockerRegistry(credentialsId: 'docker-hub-token', url: $registry) {
              sh 'docker image push $imageName:$BUILD_NUMBER'
              sh 'docker image push $imageName:latest'
            }
        }
    }
    stage('Run Docker Container') {
        agent any
        steps {
            sh 'docker -H tcp://192.168.56.10:2375 container run -d --name myweb -p 80:8080 $imageName:$BUILD_NUMBER'
        }
    }
  }
}
```
#### (6) github 에 업로드
```bash
vagrant@jenkins:~/test$ ls
Dockerfile  Jenkinsfile  Jenkinsfile-docker  pom.xml  src
vagrant@jenkins:~/test$ git add .
vagrant@jenkins:~/test$ git commit -m "Jenkinsfile-docker add"          
[main 93f7ade] Jenkinsfile-docker add
 2 files changed, 56 insertions(+)
 create mode 100644 Dockerfile
 create mode 100644 Jenkinsfile-docker
vagrant@jenkins:~/test$ git push -u origin main
...
Branch 'main' set up to track remote branch 'main' from 'origin'.
```
#### (7) 프로젝트 생성 및 빌드

# [6] 쿠버네티스에서 사용
## 1. 설치
### 1) 쿠버네티스 설치
kubespray 로 클러스터 설치
```bash
root@control-plane:~# kubectl version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.6", GitCommit:"ff2c119726cc1f8926fb0585c74b25921e866a28", GitTreeState:"clean", BuildDate:"2023-01-18T19:22:09Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.6", GitCommit:"ff2c119726cc1f8926fb0585c74b25921e866a28", GitTreeState:"clean", BuildDate:"2023-01-18T19:15:26Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"linux/amd64"}
```
설치 시 addons.yaml 파일 설정해서 helm , metallb 설치

### 2) nfs 동적볼륨 구성
```bash
root@control-plane:~# apt install nfs-kernel-server -y
root@control-plane:~# ansible -b -u vagrant -m apt -a name=nfs-common -i kubespray/inventory/mycluster/inventory.ini all
root@control-plane:~# vim /etc/exports
root@control-plane:~# tail -n1 /etc/exports
/nfs  *(rw,sync,no_subtree_check,no_root_squash)
root@control-plane:~# mkdir /nfs
root@control-plane:~# exportfs -r
root@control-plane:~# helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.56.11 --set nfs.path=/nfs
root@control-plane:~# helm list
NAME                            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                   APP VERSION
nfs-subdir-external-provisioner default         1               2023-05-01 03:29:08.45493436 +0000 UTC  deployed        nfs-subdir-external-provisioner-4.0.18  4.0.2
root@control-plane:~# kubectl get sc,deploy,po
NAME                                     PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   105m

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-subdir-external-provisioner   1/1     1            1           105m

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/nfs-subdir-external-provisioner-6bdc77f8fc-kgnwq   1/1     Running   0          105m
```

### 3) 젠킨스 설치
#### (1) chart 추가
```bash
root@control-plane:~# helm repo add jenkinsci https://charts.jenkins.io
"jenkinsci" has been added to your repositories
root@control-plane:~# helm repo list
NAME            URL
jenkinsci       https://charts.jenkins.io
root@control-plane:~# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkinsci" chart repository
Update Complete. ⎈Happy Helming!⎈

```
### 2) 사용자화 파일 수정
```yaml jenkins-values.yaml
controller:
  tag: "lts-jdk11"
  serviceType: LoadBalancer
  installPlugins:
  - kubernetes
  - workflow-aggregator
  - git
  - configuration-as-code
  - pipeline-stage-view
  adminPassword: "젠킨스패스워드"

persistence:
  storageClass: "스토리지클래스이름"
```

### 3) 젠킨스 실행
```bash
root@control-plane:~# vim jenkins-values.yaml
root@control-plane:~# helm install jenkins jenkinsci/jenkins -f jenkins-values.yaml
NAME: jenkins
LAST DEPLOYED: Wed Apr 26 02:29:43 2023
NAMESPACE: jenkins
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc --namespace jenkins -w jenkins'
  export SERVICE_IP=$(kubectl get svc --namespace jenkins jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
  echo http://$SERVICE_IP:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://$SERVICE_IP:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins

root@control-plane:~# kubectl get sts,po,svc,pv,pvc
NAME                       READY   AGE
statefulset.apps/jenkins   1/1     41m

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/jenkins-0                                          2/2     Running   0          41m
pod/nfs-subdir-external-provisioner-6bdc77f8fc-kgnwq   1/1     Running   0          109m

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/jenkins         LoadBalancer   10.233.4.47     192.168.56.200   8080:31269/TCP   41m
service/jenkins-agent   ClusterIP      10.233.50.127   <none>           50000/TCP        41m
service/kubernetes      ClusterIP      10.233.0.1      <none>           443/TCP          3h14m

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
persistentvolume/pvc-241f8af5-592a-412a-853a-364c41055398   8Gi        RWO            Delete           Bound    default/jenkins   nfs-client              41m

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/jenkins   Bound    pvc-241f8af5-592a-412a-853a-364c41055398   8Gi        RWO            nfs-client     41m
```

## 2. 접속 및 사용
### 1) 대시보드 접속
EXTERNAL-IP 로 접속 가능
접속 시 초기구성 없음 (완료상태)
### 2) 기본예제 테스트
  1. 새로운 item 클릭
  2. 이름 입력 및 Pipeline 선택
  3. 구성에서 Pipeline 을 스크립트 방식 중 Declarative 로 선택 및 빌드

### 3) kaniko 로 구성
#### (1) 프로젝트 생성
새 item > pipeline > pipeline script from SCM 선택
#### (2) jenkinsfile 생성 및 깃허브에 업로드
```jenkinsfile
pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.6-openjdk-8
    command:
    - sleep
    args:
    - infinity
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - name: registry-credentials
      mountPath: /kaniko/.docker
  volumes:
  - name: registry-credentials
    secret:
      secretName: regcred
      items:
      - key: .dockerconfigjson
        path: config.json
'''
    }
  }
  environment {
    imageName = "kimjuhyo/cicd-kaniko"
    projectName = "kube_pipeline"
  }
  stages {
    stage('Checkout') {
      steps {
        container('maven') {
          git branch: 'main', url: 'https://github.com/jhkim-09/test.git'
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn -DskipTests=true clean package'
        }
      }
    }
    stage('Test') {
      steps {
        container('maven') {
          sh 'mvn test'
        }
      }
    }
    stage('Tag Docker Image') {
      steps {
        container('kaniko') {
          sh 'executor --context=dir:///home/jenkins/agent/workspace/$projectName/ --destination=$imageName:$BUILD_NUMBER --destination=$imageName:latest'
        }
      }
    }
  }
}
```
```bash
root@control-plane:~# vim Jenkinsfile-k8s
root@control-plane:~# git add .
root@control-plane:~# git commit -m "jenkins and k8s"
root@control-plane:~# git push
```

#### (3) 도커허브에서 액세스 토큰 생성 및 쿠버네티스 시크릿 생성
```bash
root@control-plane:~# kubectl create secret docker-registry regcred --docker-server https://index.docker.io/v1/ --docker-username kimjuhyo --docker-password dckr_pat_7I05cPL1w-c7qiyC9XJz0jc-Dfg 
```
#### (4) 빌드

## 3. Argo CD
### 1) 설치
```bash
root@control-plane:~# kubectl create namespace argocd
namespace/argocd created
root@control-plane:~# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
...

root@control-plane:~# kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.233.61.42    <none>        7000/TCP,8080/TCP            2m
argocd-dex-server                         ClusterIP   10.233.53.40    <none>        5556/TCP,5557/TCP,5558/TCP   2m
argocd-metrics                            ClusterIP   10.233.61.98    <none>        8082/TCP                     2m
argocd-notifications-controller-metrics   ClusterIP   10.233.4.42     <none>        9001/TCP                     2m
argocd-redis                              ClusterIP   10.233.55.121   <none>        6379/TCP                     2m
argocd-repo-server                        ClusterIP   10.233.60.195   <none>        8081/TCP,8084/TCP            2m
argocd-server                             ClusterIP   10.233.3.181    <none>        80/TCP,443/TCP               2m
argocd-server-metrics                     ClusterIP   10.233.46.143   <none>        8083/TCP                     2m
root@control-plane:~# kubectl patch svc -n argocd argocd-server -p '{"spec": {"type": "LoadBalancer"}}'
service/argocd-server patched

root@control-plane:~# kubectl -n argocd get secrets argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
ilWAEw3ESkgGBp5H
root@control-pget svc -n argocd argocd-server
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
argocd-server   LoadBalancer   10.233.3.181   192.168.56.201   80:32123/TCP,443:30994/TCP   34m
```

### 2) 샘플 테스트
  1. EXTERNAL-IP 로 접속
  2. admin 사용자로 로그인
  3. 좌측 탭에서 Applications 항목 선택 및 상단 메뉴에서 +New APP 버튼 클릭
  4. 항목 별 입력 후 생성
    GENERAL
      Application NAME : guestbook
      Project : default
      SYNC POLICY : Automatic
    SOURCE 
      Repository URL : https://github.com/argoproj/argocd-example-apps.git
      Path : guestbook
    DESTINATION
      Cluster URL : https://kubernetes.default.svc
      Namespace : default

### 3) 직접 작성
```bash
root@control-plane:~# git clone https://github.com/jhkim-09/test.git
Cloning into 'test'...
remote: Enumerating objects: 88, done.
remote: Counting objects: 100% (88/88), done.
remote: Compressing objects: 100% (59/59), done.
remote: Total 88 (delta 30), reused 67 (delta 15), pack-reused 0
Unpacking objects: 100% (88/88), 11.88 KiB | 225.00 KiB/s, done.
root@control-plane:~# cd test/
```

```yaml deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deploy
spec:
  replicas: 2
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - image: IMAGE
        name: hello-world
        ports:
        - containerPort: 8080
```
```yaml service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world-svc
spec:
  type: LoadBalancer
  selector:
    app: hello-world
  ports:
    - targetPort: 8080
      port: 80
```

### 