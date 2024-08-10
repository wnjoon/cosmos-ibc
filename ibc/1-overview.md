# Overview

## IBC (Inter-Blockchain Communication)

- 서로 다른 블록체인간 인증 및 데이터 전송을 처리하는 프로토콜
- [ICS(Interchain Standards)](https://github.com/cosmos/ibc/tree/main/spec/ics-001-ics-standard)이라고 부르는 표준을 준수해야한다.
- ICS는 크게 2가지 카테고리로 구분 가능하다.
    - IBC/TAO : 패킷의 Transport, Authentication, Ordering을 정의하는 표준
    - IBC/APP : 전송 계층(IBC/TAO)을 통과한 데이터 패킷을 어플리케이션 레벨에서 처리하는 방식을 정의하는 표준

<br>

## 간략한 동작 방식

![high level overview](https://tutorials.cosmos.network/resized-images/600/academy/3-ibc/images/ibcoverview.png)

- 체인간 통신은 Relayer라고 부르는 오프체인에서 이루어지며, Relayer는 각 체인의 상태를 확인하고 적절한 데이터그램을 구성하고 프로토콜에서 허용하는 동작을 반대편 체인에서 실행한다.
- Relayer는 체인간 메시지 전송을 위해 채널을 구성한다.
- 각 체인은 상대 체인의 Light Client를 통해 Relayer로 전달되는 메시지를 빠르게 검증한다.

<br>

4-

