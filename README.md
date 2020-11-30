
# 숙소예약 시스템

본 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 시스템입니다.

# Table of contents

- [숙소예약관리 시스템](#---)
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

기능적 요구사항
1. 고객이 숙소를 선택하여 예약한다.
1. 예약이 발생하면 숙소관리시스템에서 관리자가 예약을 확정한다.
1. 예약이 확정되면 결제시스템 결제가 완료된다.
1. 예약이 불가하면 숙소관리시스템에서 예약을 반려한다.
1. 고객이 예약을 취소할 수 있다.
1. 예약이 취소되면 결제가 취소된다.
1. 고객이 숙소 예약상태를 중간중간 조회한다.
1. 관리자가 숙소 요청상태를 중간중간 조회한다.


비기능적 요구사항
1. 트랜잭션
    1. 고객이 예약한 건에 대하여 관리자가 예약가능 여부를 확인한 후에 결제를 진행한다. 
1. 장애격리
    1. 숙소관리 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 관리자가 결제를 잠시후에 하도록 유도한다. 고객에게는 Pending상태로 보여준다. Circuit breaker, fallback
1. 성능
    1. 고객이 숙소에 대한 최종 예약상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다. CQRS
    1. 관리자가 숙소요청상태를 숙소관리시스템(프론트엔드)에서 확인할 수 있어야 한다. CQRS


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://www.msaez.io/#/storming/O2fif4sDU5Qca7gMdXuAxWJXlS42/mine/4b6505b3ec1af64a110d5ec526aa76fc


### 이벤트 도출
![image](https://user-images.githubusercontent.com/63624005/100561536-85a0fe00-32fc-11eb-867a-661a715ab4dd.png)

  
### 액터, 커맨드 부착
![image](https://user-images.githubusercontent.com/63624005/100561589-a8331700-32fc-11eb-9529-fd5e6fbfb6c1.png)


### 어그리게잇으로 묶기, 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/63624005/100561638-c3058b80-32fc-11eb-85e1-9f02767ec27a.png)


    - 도메인 서열 분리
      . Core Domain : 예약 
      . Supporting Domain : 숙소관리
      . General Domain : 결제
    - 팀별 KPI
      . 예약 : 예약 건수
      . 숙소관리 : 예약 성공율
      . 결제 : 결제 성공율
      

### 폴리시 부착
![image](https://user-images.githubusercontent.com/63624005/100561662-d153a780-32fc-11eb-826a-fdaab23e521d.png)


### 컨텍스트 매핑
![image](https://user-images.githubusercontent.com/63624005/100561689-df092d00-32fc-11eb-9b1a-ed67647284e5.png)

    - Correlation Key
     . Reservation ID (reservation - management - payment)
    - View
     . ReservationList : 숙소, 예약 진행상황, 결제 진행상황
     . ManagementList : 예약 요청상황, 예약 가능여부, 결제 진행상황 



### 1차 완성본 (기능적/비기능적 요구사항 검증)

![image](https://user-images.githubusercontent.com/63624005/100561706-ea5c5880-32fc-11eb-84da-5e67a0dab581.png)
    - 고객이 숙소를 선택하여 예약한다. (ok)
    - 예약이 발생하면 숙소관리시스템에서 관리자가 예약을 확정한다. (ok)
    - 예약이 확정되면 결제시스템에서 결제가 완료된다. (ok)
    - 예약이 불가하면 숙소관리시스템에서 예약을 반려한다. (ok)
    
    - 고객이 예약을 취소할 수 있다. (ok)
    - 예약이 취소되면 결제가 취소된다. (ok)
    - 고객이 숙소 예약상태를 중간중간 조회한다. (ok)
    - 관리자가 숙소 요청상태를 중간중간 조회한다. (ok)
    
![image](https://user-images.githubusercontent.com/63624005/100561734-006a1900-32fd-11eb-801c-219aedf91d45.png)


    - 빠른 고객응답 보다는 서비스의 안정성을 더욱 중시하는 비즈니스적인 이유로 수정 (Req/Res --> Pub/Sub)
![image](https://user-images.githubusercontent.com/63624005/100561830-47580e80-32fd-11eb-9277-fe1cae4eecf0.png)

   
### 비기능 요구사항에 대한 검증

    - 트랜잭션 (1)
      . 고객이 예약한 건에 대하여 관리자가 예약가능 여부를 확인한 후에 결제를 진행한다.
    - 장애격리 (2)
      . 숙소관리 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. 
        [Async (event-driven), Eventual Consistency]
      . 결제시스템이 과중되면 관리자가 결제를 잠시후에 하도록 유도한다. 고객에게는 Pending상태로 보여준다. 
    - 성능 (3)
      . 고객이 숙소에 대한 최종 예약상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다. [CQRS]
      . 관리자가 숙소요청상태를 숙소관리시스템(프론트엔드)에서 확인할 수 있어야 한다. [CQRS]
    

## 헥사고날 아키텍처 다이어그램 도출
- 1차
![image](https://user-images.githubusercontent.com/63624005/100561868-5939b180-32fd-11eb-8f9a-debfc8a65cc2.png)

    - 빠른 고객응답 보다는 서비스의 안정성을 중시하는 비즈니스적인 이유로 수정함 (Req/Res --> Pub/Sub)

- 2차
![image](https://user-images.githubusercontent.com/63624005/100561892-69519100-32fd-11eb-99f2-1ec26602fcbd.png)



# 구현


