---
layout: post
title:  "Jib java version 에러 해결 과정"
date:   2024-04-23 19:41:06 +0900
categories: error
tag: jib
---

jib을 통해서 손쉽게 개발한 java application 쉽게 컨테이너 이미지로 빌드하기 위한 과정 중에 만난 문제를 공유합니다.

## 초기 설정
jib을 이용해서 Spring boot 이미지를 배포하기 위해서 다음 설정을 사용했습니다. gradle언어로는 kotlin을 사용하였습니다.

```gradle
plugins {
    ...
    // jib
    id("com.google.cloud.tools.jib") version "3.1.4"
}

jib {
    from {
        image = "eclipse-temurin:21-jre"
    }
    to {
        image = "localhost:5000/authApplication"
        setAllowInsecureRegistries(true)
        tags = setOf("latest")
    }
    container {
        jvmFlags = listOf("-Xms512m", "-Xmx512m")
    }
}
```

## 문제 1 : docker image name with UPPER CASE
하지만 다음과 같은 에러가 발생했습니다.
![image](https://github.com/kSideProject/kSideProject.github.io/assets/67232422/d25aa8cd-d469-4e01-8cf8-1050feb4712d)
https://docs.docker.com/reference/cli/docker/image/tag/#extended-description

위의 문서를 참고해서 말하자면 원인은 이미지 이름으로 대문자를 사용했기 때문이었습니다.
> The path consists of slash-separated components. Each component may contain lowercase letters, digits and separators

에러 메시지에서 역시 대문자를 사용해서 실패했음을 알려주었기 때문에 `authApplication`을 `auth-application`으로 수정하게 되었습니다.

## 문제 2 : Unsupported java error
그 다음으로 발생한 에러는 자바의 버전이 매칭되지 않은 에러가 발생했습니다.
![image](https://github.com/kSideProject/kSideProject.github.io/assets/67232422/20cc47de-b498-44f0-b30f-3d1c8a230417)

로컬상에서 분명 빌드까지는 확인했기 때문에 소스 코드와 관련된 문제라고 생각하지 않았습니다.
해당 프로젝트에서는 jdk21를 최신 LTS 버전을 사용하고 있기 때문에 낮은 버전의 jib에서는 jdk 21을 지원하지 않을 수도 있겠다는 생각에
jib의 버전을 `3.1.4`에서 `3.4.0`으로 업그레이드를 하여 실행을 해본 결과 성공하게 되었습니다.

하지만 공식적으로 jib의 버전에 따른 jdk지원 버전에 대한 정보는 찾지 못했습니다.

## 해결된 설정
그래서 사용하게된 최종 설정은 다음과 같습니다.
```gradle
plugins {
    ...
    // jib
    id("com.google.cloud.tools.jib") version "3.4.0"
}

jib {
    from {
        image = "eclipse-temurin:21-jre"
    }
    to {
        image = "localhost:5000/auth-application"
        setAllowInsecureRegistries(true)
        tags = setOf("latest")
    }
    container {
        jvmFlags = listOf("-Xms512m", "-Xmx512m")
    }
}
```

## 참고자료
jib를 통한 java 이미지 생성 : https://cloud.google.com/java/getting-started/jib
