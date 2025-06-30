# 🚀 Spring Boot EC2 배포 자동화 (with GitHub Actions)

간단한 "Hello, DevOps!" 메시지를 반환하는 Spring Boot 애플리케이션을 EC2에 자동 배포하는 과정을 정리한 실습 프로젝트입니다.


## 📁 프로젝트 구성

📦 devops-ci-practice
├── src/main/java/com/example/demo/HelloController.java
├── src/main/java/com/example/demo/Application.java
├── build.gradle
├── settings.gradle
├── gradlew
├── .github/workflows/deploy.yml

필수 파일 설명
gradlew: Gradle wrapper 실행 스크립트 (리눅스/유닉스용)
build.gradle: 프로젝트 의존성 및 빌드 설정
settings.gradle: 프로젝트 이름 등 기본 설정
.github/workflows/deploy.yml: GitHub Actions 자동화 스크립트


## ✅ 주요 목표

- Spring Boot 애플리케이션 작성 및 GitHub에 업로드
- GitHub Actions를 통해 EC2에 자동 배포
- EC2 인스턴스에서 Spring 애플리케이션 실행 및 확인


## ⚙️ EC2 설정 요약

- Amazon Linux 2023 AMI 
- Java 11 설치 (corretto-11)
- EC2 인스턴스 2개 구성 권장 (소스 push용, 애플리케이션 실행용)

sudo dnf install java-11-amazon-corretto -y
mkdir -p ~/app


## 🧪 에러 이슈 및 해결 과정 정리

### ❌ 문제 1: java -jar 명령어 실행 직후 앱이 종료됨
GitHub Actions의 Workflows 로그에서 종료 확인됨 (EC2 내부의 app.log 파일 자체가 생성되지 않음)
원인: GitHub Actions가 SSH 세션을 종료하면서 nohup으로 실행된 자식 프로세스도 함께 종료됨
해결: tmux 사용하여 세션 독립 실행

### ✅ 해결 방법: tmux로 실행
sudo dnf install -y tmux
tmux kill-session -t springapp || true
tmux new-session -d -s springapp "java -jar ~/app/build/libs/simple-spring-app-0.0.1-SNAPSHOT.jar > ~/app/app.log 2>&1"

※ tmux kill-session은 동일 세션 이름 충돌 방지를 위한 명령으로, 재배포 시 안정성 확보에 중요합니다.


## 🤔 nohup vs tmux 차이

nohup
세션 종료 무시. 이론적으로는 프로세스 유지됨. 하지만 GitHub Actions처럼 aggressive한 SSH 종료에서는 영향 받을 수 있음

tmux
완전히 독립된 세션에서 실행되기 때문에 외부 세션 종료와 무관하게 앱이 계속 실행됨


## 🔐 GitHub Actions Secrets 설정

EC2_HOST: 애플리케이션 실행 EC2의 퍼블릭 IP
EC2_SSH_KEY: 해당 EC2 접속용 키 (.pem 파일 내용 전체)
cat ~/.ssh/your-key.pem
주의: 키 값 앞뒤 공백/줄바꿈 없이 붙여넣기!


## 🧷 deploy.yml 요약 (핵심 부분)
- name: Run jar on EC2
  uses: appleboy/ssh-action@v0.1.7
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ec2-user
    key: ${{ secrets.EC2_SSH_KEY }}
    script: |
      sudo dnf install -y tmux
      tmux kill-session -t springapp || true
      tmux new-session -d -s springapp "java -jar ~/app/build/libs/simple-spring-app-0.0.1-SNAPSHOT.jar > ~/app/app.log 2>&1"


## 📌 참고 명령어들

### EC2 내에서 Java 버전 확인
java -version

### jar 직접 실행해보기
java -jar ~/app/build/libs/simple-spring-app-0.0.1-SNAPSHOT.jar

### tmux 실행 중 확인
ps aux | grep java

### tmux 세션 확인/접속
tmux ls
tmux attach -t springapp

## 📖 정리

이 실습을 통해 CI → EC2 자동 배포 흐름을 경험하고,
GitHub Actions의 SSH 세션 한계와 백그라운드 실행 방식 (tmux vs nohup)의 차이를 직접 확인함.

이 README는 실습 도중 발생한 실제 문제와 해결 과정을 반영하여 기록되었습니다.
