# 🛠️ 개인 시스템 기반 CI/CD 자동화 구축

## 🎯 프로젝트 목표
### 1. 로컬 Jenkins 기반 CI 파이프라인 구축
- GitHub 저장소에 push가 발생하면 코드를 자동으로 pull 받아 빌드

- 빌드된 `.jar` 파일을 생성하여 관리하는 자동화된 빌드 시스템 구성

### 2. 운영 서버(myserver02)로의 자동 배포 및 실행 (CD)
- 빌드된 .jar 파일을 원격 서버로 전송
  
- 기존 프로세스를 종료한 후 nohup으로 새로운 앱을 자동 실행하여 완전한 개인 CI/CD 흐름 구현

---

## 🛠 사용 기술 스택

<div>
  <img src="https://img.shields.io/badge/jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white">
  <img src="https://img.shields.io/badge/gradle-02303A?style=for-the-badge&logo=gradle&logoColor=white">
  <img src="https://img.shields.io/badge/springboot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white">
  <img src="https://img.shields.io/badge/ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white">
  <img src="https://img.shields.io/badge/shell_script-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white">
  <img src="https://img.shields.io/badge/github-181717?style=for-the-badge&logo=github&logoColor=white">
</div>

<br>

| 분류       | 기술 / 도구 |
|------------|--------------|
| CI 도구    | Jenkins (Docker 기반) |
| VCS        | Git + GitHub (WebHook 연동) |
| 빌드 도구  | Gradle |
| 백엔드     | Spring Boot (JAR 빌드) |
| 배포 방식  | SCP + SSH + nohup 실행 |
| 네트워크   | NAT Network / (2단계: ngrok) |
| 운영체제   | Ubuntu 24.04.2 LTS (VirtualBox 기반) |
| 스크립트   | Shell Script (`deploy.sh`) |

---

## ✅ 기본 구조

```
GitHub → [myserver01]
              |
              ↓
        [Jenkins 컨테이너]
        - 빌드 완료된 .jar 생성
        - /jar_output 에 복사
              |
              ↓
  scp로 myserver02 전송 (10.0.2.20)
              |
              ↓
  myserver02: 기존 jar 파일 kill → nohup 으로 새로운 jar 실행
```

---

## 📁 1. 배포 스크립트 (`deploy.sh`) 작성
![image (10)](https://github.com/user-attachments/assets/b360301e-a474-4854-b167-6382ee8a5724)

- 위치: `/home/ubuntu/jarappdir` (호스트 기준)
  
- 역할: 최신 `.jar` 파일을 myserver02로 전송 후 실행

```bash
#!/bin/bash

JAR_NAME=$(ls /var/jenkins_home/jar_output/*.jar | grep -v 'plain' | tail -n 1)

DEST_USER=ubuntu
DEST_HOST=10.0.2.20
DEST_DIR=/home/ubuntu/app
KEY=/var/jenkins_home/keys/jenkins_key

echo "📦 Deploying $JAR_NAME to $DEST_HOST..."
chmod 600 $KEY
scp -o StrictHostKeyChecking=no -i $KEY "$JAR_NAME" "$DEST_USER@$DEST_HOST:$DEST_DIR/"

ssh -o StrictHostKeyChecking=no -i $KEY $DEST_USER@$DEST_HOST <<EOF
    echo "🛑 Killing previous Java process..."
    pkill -f 'java -jar' || true
    echo "🚀 Running new app with nohup..."
    cd $DEST_DIR
    nohup java -jar $(basename $JAR_NAME) > app.log 2>&1 &
EOF

echo "✅ Deployment complete! Check $DEST_HOST:$DEST_DIR/app.log for logs."
```

---

## ⚙️ 2. Jenkins Pipeline 설정
- git url은 본인의 Github Repository 주소로 작성

```groovy
pipeline {
    agent any

    environment {
        PROJECT_DIR = 'step07_cicd'
        JAR_OUTPUT = '/var/jenkins_home/jar_output'
        DEPLOY_SCRIPT = '/var/jenkins_home/scripts/deploy.sh'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Pulling from Git repository...'
                git url: 'https://github.com/username/repo.git', branch: 'main'
            }
        }

        stage('Build') {
            steps {
                echo "⚙️ Running Gradle Build in ${env.PROJECT_DIR}..."
                dir("${env.PROJECT_DIR}") {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew clean build'
                }
            }
        }

        stage('Copy to Bind Folder') {
            steps {
                echo '📁 Copying JAR to bind-mounted host directory...'
                sh "cp ${env.PROJECT_DIR}/build/libs/*.jar ${env.JAR_OUTPUT}/"
            }
        }

        stage('Run Deploy Script') {
            steps {
                echo '🚀 Running deploy.sh to deliver and run JAR on remote system...'
                sh "chmod +x ${env.DEPLOY_SCRIPT}"
                sh "${env.DEPLOY_SCRIPT}"
            }
        }
    }

    post {
        success {
            echo '✅ CI/CD pipeline executed successfully!'
        }
        failure {
            echo '❌ CI/CD pipeline failed.'
        }
    }
}
```

---

## 📦 3. Jenkins 컨테이너에 `deploy.sh` 복사

```bash
docker exec -it jenkins mkdir -p /var/jenkins_home/scripts

docker cp /home/ubuntu/jarappdir/deploy.sh jenkins:/var/jenkins_home/scripts/deploy.sh
```

---

## ✅ 4. 배포 확인
운영 서버(myserver02)에서 `.jar`가 실행 중인지 확인:

```bash
ps aux | grep java
```
![image (11)](https://github.com/user-attachments/assets/2ae1c0e1-d82b-4255-92dc-674330e5314f)

또는 로그 확인:

```bash
tail -f /home/ubuntu/app/app.log
```
![image](https://github.com/user-attachments/assets/e5651be7-8b6b-401a-8093-cf2fa1f43f07)

---

## 🚀 5. GitHub → Jenkins 자동 빌드 연동 (ngrok 사용)

### 📡 1) ngrok 실행

```bash
ngrok http http://localhost:8080
```

→ 생성된 URL을 복사 (예: `https://1234-abc.ngrok.app`)

![image (12)](https://github.com/user-attachments/assets/9265b93f-d888-4fe3-b602-50645cb9c358)


### 🔗 2) GitHub 웹훅 설정

- GitHub → Settings → Webhooks
  
- Payload URL에 `https://xxxx.ngrok.app/github-webhook/` 입력
   - /github-webhook/ 을 꼭 적어줘야 함
- Content type: `application/x-www-form-urlencoded`
- 이벤트: `Just the push event.`

![image (13)](https://github.com/user-attachments/assets/03855339-f474-40d4-9607-90665372297d)

### 🛠 3) Spring Boot 포트 포워딩
- 브라우저 접근을 위해 호스트 포트와 게스트 포트를 매핑하는 포트 포워딩 설정이 필요
- VirtualBox 포트 포워딩 설정 예:
  
  | 이름 | 호스트 IP | 호스트 포트 | 게스트 IP | 게스트 포트 |
  |------|-----------|--------------|-----------|--------------|
  | app  | 127.0.0.1 | 8081         | 10.0.2.20 | 8080         |

### ✅ 4) 자동화 테스트

1. GitHub에 push
   
2. Jenkins에서 자동 빌드  
3. myserver02에서 앱 자동 실행  
4. 브라우저로 `localhost:8081` 접근하여 결과 확인

---
