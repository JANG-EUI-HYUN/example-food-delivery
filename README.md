![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 예제 - 음식배달

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 음식배달](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

배달의 민족 커버하기 - https://1sung.tistory.com/106

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다
1. 고객이 결제한다
1. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
1. 상점주인이 확인하여 요리해서 배달 출발한다
1. 고객이 주문을 취소할 수 있다
1. 주문이 취소되면 배달이 취소된다
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven

#Model
https://www.msaez.io/#/storming/m1rDZxH8GYRdTt5O9mPJGwCfT4h2/d6d03daa47220fdbbddd6e25d629ecdb
![image](https://user-images.githubusercontent.com/81133592/204477418-a442a638-c85e-469e-a01d-6845161527ef.png)

# 체크포인트

1.Saga (Pub / Sub)
2.CQRS
3.Compensation / Correlation
4.Request / Response
5.Circuit Breaker
6.Gateway / Ingress

# Saga (Pub / Sub)
![image](https://user-images.githubusercontent.com/81133592/204477939-4e508a34-2788-4ea1-bfc9-f573001ae418.png)
![image](https://user-images.githubusercontent.com/81133592/204477968-07aeda02-8360-428a-b0f3-95c1e47a7689.png)
![image](https://user-images.githubusercontent.com/81133592/204477989-0b6dc9bb-3361-4c9a-aea2-8affb20efd00.png)

# Request / Response
![image](https://user-images.githubusercontent.com/81133592/204478054-00433938-7fce-4472-a260-34af76d30921.png)

# Circuit Breaker
![image](https://user-images.githubusercontent.com/81133592/204478104-c9a3d8da-746d-4d86-8009-81b2cf515ce9.png)

# Gateway / Ingress
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: front
          uri: http://front:8080
          predicates:
            - Path=/주문/**, /orders/**, /payments/**, /메뉴판/**, /통합주문상태/**
        - id: store
          uri: http://store:8080
          predicates:
            - Path=/주문관리/**, /orderManages/**, /주문상세보기/**
        - id: customer
          uri: http://customer:8080
          predicates:
            - Path=/logs/**, /orderStatuses/**
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/**, 
        - id: frontend
          uri: http://frontend:8080
          predicates:
            - Path=/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080

# 추가사항
![image](https://user-images.githubusercontent.com/81133592/204478254-f336a322-710f-4c44-826e-e710a0238880.png)
![image](https://user-images.githubusercontent.com/81133592/204478278-0547956f-fb2e-4962-9a45-f7382d4890be.png)
![image](https://user-images.githubusercontent.com/81133592/204478293-592b8d45-1ec2-45ee-b798-ecbb88a91099.png)

- 배송 완료/픽업 상태 확인 가능
![image](https://user-images.githubusercontent.com/81133592/204478502-4da3c65e-886a-4f2b-8ef5-35ff8b303a52.png)
![image](https://user-images.githubusercontent.com/81133592/204478524-2f7d86d7-025e-4ab4-9b59-4008ad5874de.png)
![image](https://user-images.githubusercontent.com/81133592/204478535-54ca83a3-ddb4-4f05-a652-ce9d97092f5c.png)
![image](https://user-images.githubusercontent.com/81133592/204478557-ee755592-d005-4d3e-b06a-516678bc2fae.png)
![image](https://user-images.githubusercontent.com/81133592/204478582-2b9ee966-7990-422d-8c3b-5b94d30c07d1.png)

- 주문 수락/거절, 요리 시작/완료 상태 확인 가능
- 사용자는 요리 시작 전 주문 취소 가능

# Before Running Services
Make sure there is a Kafka server running
cd kafka
docker-compose up

- Check the Kafka messages:
cd kafka
docker-compose exec -it kafka /bin/bash
cd /bin
./kafka-console-consumer --bootstrap-server localhost:9092 --topic 

- Run the backend micro-services
See the README.md files inside the each microservices directory:

- front
- store
- customer
- delivery

# Run API Gateway (Spring Gateway)
cd gateway
mvn spring-boot:run

# Test by API
- front
 http :8088/주문 id="id" 품목="품목" 수량="수량" 
 http :8088/orders id="id" foodId="foodId" amount="amount" customerId="customerId" options="options" address="address" status="status" 
 http :8088/payments id="id" orderId="orderId" amount="amount" 
 
- store
 http :8088/주문관리 id="id" 
 http :8088/orderManages id="id" foodId="foodId" orderId="orderId" status="status" test="test" couponNumber="couponNumber" 
 
- customer
 http :8088/logs id="id" customerId="customerId" message="message" 
 
- delivery
 http :8088/deliveries id="id" address="address" orderId="orderId" riderId="riderId" status="status"  






