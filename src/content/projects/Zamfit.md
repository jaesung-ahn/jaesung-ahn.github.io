---
title: zamfit
description: 빅데이터 기반 놀거리 원스톱 플랫폼, 잼핏
date: Oct 18 2024
demoURL: https://zamfit.kr/
repoURL: https://github.com/jaesung-ahn/escape-room-server
---

![](Zamfit.jpeg)

> **2022.03.05 ~ 2022.12.14**

- 안전하게 새로운 친구 사귀고, **간편하게 방탈출 놀거리 찾을 수 있는 서비스**

<aside>
방탈출 컨텐츠의 대한 정보를 사용자에게 공유한다
사용자는 방탈출 정보를 확인 할 수 있으며, 함께할 사람들을 모집할 수 있다
방탈출 모임을 관리해주며 추후에 예약 및 결제 서비스도 연계 예정
</aside>

### 기술 스택

- spring boot, spring security, jpa, postgresql
- aws, linux, jenkins, nginx

### 개발 내용

- Spring Rest Docs와 ascii doc를 사용하여 테스트 코드를 통해 문서 자동화
- 암호화에 Java 기본 라이브러리(javax.crypto.Mac, javax.crypto.spec.SecretKeySpec)를 사용하여 외부 라이브러리 없이 jwt 구현
- domain 패키지 내부의 코드를 외부 라이브러리 의존성 없이 순수한 java 코드로만 작성
- 멀티 모듈을 통한 관리자, API, 공통 코드 분리
- spring AOP와 slack api 사용하여 관리자 승인이 필요할 경우 알림을 전송하는 기능 개발
- 카카오 소셜 회원가입 구현
- spring security를 이용한 인증/인가 처리 로직 구현
- mahout 사용한 간단한 추천시스템 개발