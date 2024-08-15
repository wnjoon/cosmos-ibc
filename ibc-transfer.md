# IBC 토큰 전송 절차 분석

본 문서에서는 IBC 프로토콜을 사용하여 ICS-20(Fungible Token Transfer)가 일어나는 전체 단계에 대해서 알아본다. 

![Structure of several layers in IBC](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/lightclient.png)

<br>

## IBC Handshaking

토큰 전송을 위해서는 IBC Hanadshaking 과정을 통해 각 체인이 상대 체인을 신뢰하는 환경을 먼저 구성해야 한다. 이러한 환경 구성에는 오프체인으로 별도로 구성된 릴레이어(Relayer)가 필요하며, 구성 결과로 클라이언트(Client), 커넥션(Connection), 채널(Channel)이 생성된다. 릴레이어와 클라이언트에 대한 설명은 이전 과제에서 진행하였으므로 생략한다.

### 커넥션(Connection)

서로 다른 두 체인을 연결하기 위해 사용하며, 상대 체인에 대한 명확한 식별을 통해 상대 체인인 것 처럼 가장하여 잘못된 정보를 전달하는 것을 방지한다. 체인의 원장 코드에 의해 설정되기 때문에, 오프체인 기반의 제3자 프로세스와의 상호작용을 요구하지 않는다. 클라이언트 위에 구축되며, 클라이언트가 상호작용하고자 하는 IBC 프로토콜 개수에 따라 기술적으로 하나의 클라이언트는 여러 커넥션을 가질 수 있다. 

#### 4-way Handshaking

커넥션은 4-way handshaking 방식으로 연결되며, 단계별로 OpenInit, OpenTry, OpenAck, OpenConfirm으로 표현된다. 아래 그림은 두개의 체인(A, B)이 커넥션 과정에 주고받은 패킷과 이로 인하여 변경되는 상태에 대해 보여준다. 체인A는 소스 체인(source chain), 체인B는 대상 체인(destination chain)이라고 가정한다.

![4-way handshaking in connection](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/connectionstate.png)

##### ConnOpenInit

1. 릴레이어는 MsgUpdateClient라는 메시지를 소스 체인으로 보내서, 대상 체인(체인 B)의 최신 상태를 소스 체인의 클라이언트에 업데이트한다. MsgUpdateClient에는 대상 체인에 대한 블록 정보와 최신 합의 상태가 포함되어 있다.
2. 소스 체인은 OpenInit 메시지를 전송하고, 릴레이어가 이 메시지를 중계한다.
3. 소스 체인의 연결 상태를 INIT으로 변경한다. 
4. 소스 체인의 클라이언트 내 커넥션 리스트에 새로 생성된 식별자(Connection ID)를 추가한다.

> 릴레이어는 커넥션 과정에서 주고받는 메시지를 중계할 뿐, 메시지의 내용이나 프로토콜 확인과 같은 검증은 하지 않는다.

##### ConnOpenTry

1. 릴레이어는 소스 체인과 대상 체인 각각에 MsgUpdateClient 메시지를 전송해서 각 체인의 클라이언트를 최신 상태로 업데이트 한다. 
2. 대상 체인의 연결 상태를 TRYOPEN으로 변경한다.
3. 대상 체인은 클라이언트에 있는 소스 체인의 정보(마지막 블록의 루트 해시를 포함하는 합의 상태 알고리즘 및 마지막 스냅샷, 다음 검증자 집합)를 통해 소스 체인의 신원을 확인하고 소스 체인이 자신(대상 체인)의 정보를 정확하게 가지고 있는지 이중 검증한다.
4. 대상 체인은 소스 체인에서 OpenInit을 통해 제안한 프로토콜 버전을 확인하고, 지원되지 않는 버전일 경우 커넥션이 실패한다.

> 사용 가능한 프로토콜 버전일 경우, 제안한 버전을 그대로 수락하거나 사용 가능한 다른 버전을 제안할 수 있다.

##### ConnOpenAck

1. 릴레이어는 소스 체인과 대상 체인 각각에 MsgUpdateClient 메시지를 전송해서 각 체인의 클라이언트를 최신 상태로 업데이트 한다. 
2. 대상 체인이 소스 체인으로 연결을 요청하면, 소스 체인은 상태를 OPEN으로 변경한다. 이 때 대상 체인의 상태가 TRYOPEN이 아닐 경우 커넥션은 실패한다.
3. 소스 체인은 대상 체인에서 OpenTry를 통해 제안한 프로토콜 버전을 확인하고, 지원되지 않는 버전일 경우 커넥션이 실패한다.

##### ConnOpenConfirm

1. 대상 체인에서 자신과 소스 체인에 대한 식별이 모두 성공하면, 대상 체인은 상태를 OPEN으로 변경하고 커넥션 생성은 완료된다.

### 채널

IBC는 채널과 포트(Port)를 통해 서로 다른 블록체인 간 데이터와 메시지를 안전하고 효율적으로 전송한다. 포트는 IBC 프로토콜을 사용하는 애플리케이션을 위한 논리적 통로이고, 채널은 이 포트를 통해 다른 체인과 실제로 연결되는 경로를 말한다. 

포트(Port)

- 특정 애플리케이션이 IBC 프로토콜을 통해 데이터를 전송할 수 있는 통신 경로로, IBC 모듈을 사용하는 블록체인의 애플리케이션은 포트에 바인딩되어야 다른 체인과 통신할 수 있다.
- 포트마다 애플리케이션을 구별하기 위한 식별자(Port ID)를 갖는다. 예를 들어, ICS-20 프로토콜(대체 가능한 토큰 전송)의 경우 Port ID는 transfer로 설정된다.
- 여러 포트가 같은 체인에서 사용할 수 있으며, 포트는 애플리케이션의 타입과 목적에 따라 고유하게 설정됩니다.

채널(Channel)
- 포트가 실제로 데이터를 전송하기 위해 다른 체인과 연결되는 경로로, 채널은 특정 포트를 통해 소스 체인과 대상 체인 간 데이터를 주고받는다.
- 채널마다 고유한 식별자(Channel ID)를 갖는다. 채널마다 소스 체인의 포트와 대상 체인의 포트를 연결하고 있다.
- 하나의 포트는 여러 채널을 통해 여러 대상 체인과 연결할 수 있다. 예를 들어, 한 체인의 transfer 포트는 여러 채널을 통해 여러 다른 체인으로 데이터를 전송할 수 있다.

![ICS-20 cross-chain transfers](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/transferoverview.png)

위의 그림처럼 IBC 프로토콜은 각 체인 간 여러 경로(채널)를 설정할 수 있다. 이를 통해 복잡한 네트워크 환경에서 동일한 체인 내 복수의 연결 경로를 제공함으로써,한 채널이 고장나거나 사용 불가능할 때 다른 채널을 통해 통신할 수 있다.

#### 4-way Handshaking

채널도 커넥션과 비슷하게 4-way handshaking을 통해 설정된다. 동일하게 체인 A를 소스 체인, 체인 B를 대상 체인으로 표현한다.

![4-way handshaking in channel](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/channelhandshake.png)

##### ChanOpenInit

1. 소스 체인의 상태를 INIT으로 설정한다.
2. 올바른 포트가 설정되어 있는지, 채널이 order/unorder 상태인지 확인을 위한 콜백을 호출한다.

> order : 전송된 패킷이 순서대로 정확하게 전달되는 채널
> unorder : 패킷이 전송된 순서와 다르게 전달될 수 있는 채널

##### ChanOpenTry

1. 대상 체인의 상태를 TRY로 설정한다.
2. ChanOpenInit처럼 포트 설정 및 채널의 상태에 대한 콜백을 호출한다.

##### ChanOpenAck

1. 소스 체인의 상태를 OPEN으로 설정한다.
2. 두 체인이 서로 데이터를 주고받을 때 사용하는 어플리케이션 프로토콜 버전에 대한 협상을 마무리한다.

> 어플리케이션 버전 협상은 IBC 프로토콜을 사용하는 서로 다른 두 체인의 어플리케이션이 서로 호환되도록 하기 위한 단계이다. 얘로 토큰 전송(ICS-20)을 사용할 경우, 각 체인에서 지원하는 토큰 전송에 대한 프로토콜을 확인하고 이를 일치시켜야 한다.

##### ChanOpenConfirm

1. 대상 체인의 상태를 OPEN으로 설정한다.

<br>

## Token Transfer

소스체인과 싱크체인





