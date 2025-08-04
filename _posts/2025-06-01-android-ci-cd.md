---
layout: post
title: '[Android] Github workflow 로 CI/CD 구축하기'
categories: [ Android ]
tags: [ Android, CI/CD, Github Workflow ]
comments: true
banner:
  image: "/assets/images/default_android_banner.png"
---

## 1. 개요

`CI/CD(Continuous Integration/Continuous Deployment)`

Github workflow를 사용 하여 테스트 및 배포 자동화 구현

## 2. Github Workflow 설정

### 1. workflow 파일 생성

`.github/workflows` 디렉터리 생성후 `FileName.yml` 파일 생성

### 2. WorkFlow 파일 구성

#### 1. 워크플로우 이름 및 실행 트리거 설정

```yaml
name: Main Android deploy

on:
  push:
    branches: [ main ]

permissions:
  contents: read
```

- name: 워크플로우 이름
- on: 워크플로우 실행 트리거
    - push: 특정 브랜치 푸시될 경우
    - pull-request: 특정 브랜치에 풀 리퀘스트가 생성될 경우
- permissions: 워크플로우 권한 설정

#### 2. job 구성

```yaml
jobs:
  build:
    runs-on: [ self-hosted, linux, x64, docker ]

    steps:
```

- runs-on: 워크플로우가 실행될 환경
    - self-hosted: 자가 호스팅된 runner 사용
    - linux: 리눅스 운영체제
    - x64: x64 아키텍처
    - docker: 도커 컨테이너에서 실행
- steps: 다음 단계에서 후술

#### 3. steps: check out

```yaml
      - name: Checkout repository
        uses: actions/checkout@v4
```

소스코드를 로컬로 가져옴(체크아웃)

#### 4. steps: JDK 설정

```yaml
      - name: JDK 17 set-up
        uses: actions/setup-java@v4
        with:
          java-version: 17
          distribution: 'corretto'
```

[사용가능한 JDK 버전 리스트](https://github.com/actions/setup-java)

#### 5. steps: Android SDK 및 Gradle 설정

```yaml
      # Android SDK set up
      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      # grant gradle script
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
```

#### 6. steps: Gradle 캐싱 (Optional)

```yaml
      - name: Gradle Caching
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${\{ runner.os }}-gradle-${\{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}  # '\' 제거
          restore-keys: |
            ${\{ runner.os }}-gradle-  # '\' 제거
```

CI 빌드 시간 단축을 위한 Gradle 캐싱 설정.

#### 7. steps: Google Services JSON 생성 (Optional)

```yaml
      - name: Create google-services.json
        run: echo '${\{ secrets.GOOGLE_SERVICE_JSON }}' > architecture/app/google-services.json  # '\' 제거
```

Google Play 서비스가 필요한 앱은 빌드 전 google-services.json 파일 생성

**_Secrets 지정 필요_**

#### 8. steps: 테스트, 빌드 및 AAB 생성

```yaml
      # Start java Unit test
      - name: Run unit test
        run: ./gradlew testReleaseUnitTest


      - name: Build Release AAB
        run: ./gradlew app:bundleRelease
```

#### 9. steps: AAB 서명

```yaml
      # Signing AAB
      - name: Sign AAB
        uses: r0adkll/sign-android-release@v1
        with:
          releaseDirectory: architecture/app/build/outputs/bundle/release
          signingKeyBase64: ${\{ secrets.SIGNING_KEY_BASE64 }}   # '\' 제거
          alias: ${\{ secrets.SIGNING_KEY_ALIAS }}   # '\' 제거
          keyStorePassword: ${\{ secrets.SIGNING_KEY_STORE_PASSWORD }}   # '\' 제거
          keyPassword: ${\{ secrets.SIGNING_KEY_PASSWORD }}   # '\' 제거
```

- SIGNING_KEY_BASE64: Base64로 인코딩된 keystore 파일
- SIGNING_KEY_ALIAS: keystore alias
- SIGNING_KEY_STORE_PASSWORD: keystore 비밀번호
- SIGNING_KEY_PASSWORD: 키 비밀번호

##### Base64 변환 방법

openssl 설치 후 아래 명령어로 base64로 변환

```shell
openssl base64 -in [키스토어 파일명] -out [저장될텍스트파일명]
```

#### 10. steps: PlayStore 배포

```yaml
    # Deploy Play Store
    - name: Upload to Google Play Store
      uses: r0adkll/upload-google-play@v1
      with:
        serviceAccountJsonPlainText: ${\{ secrets.SERVICE_ACCOUNT_JSON }} # '\' 제거
        packageName: [ 패키지명 ]
        releaseFiles: app/build/outputs/bundle/release/app-release.aab
        changesNotSentForReview: true
        track: internal # 배포 트랙
        inAppUpdatePriority: 2
        mappingFile: app/build/outputs/mapping/release/mapping.txt
```

[기타 상세 옵션 확인](https://github.com/r0adkll/upload-google-play)

# 3. secrets 설정

1. Github 레포지토리 웹페이지로 이동
2. Settings > Secrets and variables > Actions
3. New repository secret 클릭
4. 위에서 구성한 secrets 입력

- GOOGLE_SERVICE_JSON: google-services.json 파일 내용 입력
- SIGNING_KEY_BASE64: Base64로 인코딩된 keystore 파일
- SIGNING_KEY_ALIAS: keystore alias
- SIGNING_KEY_STORE_PASSWORD: keystore 비밀번호
- SIGNING_KEY_PASSWORD: 키 비밀번호
- SERVICE_ACCOUNT_JSON: Google Play API 서비스 계정 JSON 파일 내용 입력