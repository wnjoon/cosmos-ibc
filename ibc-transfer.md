# IBC 토큰 전송 절차 분석

본 문서에서는 IBC 프로토콜을 사용하여 ICS-20(Fungible Token Transfer)가 일어나는 전체 단계에 대해서 알아본다.   

![Structure of several layers in IBC](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/lightclient.png)

위 그림을 간략하게 표현하면 아래와 같다.
1. 릴레이어는 각 체인별로 상대 체인에 대한 클라이언트([ICS-2](https://github.com/cosmos/ibc/tree/main/spec/core/ics-002-client-semantics), [ICS-7](https://github.com/cosmos/ibc/tree/main/spec/client/ics-007-tendermint-client))를 생성한다. 
2. 클라이언트는 상대 체인의 최신 상태를 검증([ICS-23](https://github.com/cosmos/ibc/tree/main/spec/core/ics-023-vector-commitments))하고, 이를 통해 양 체인의 상태가 동기화된다.
3. 커넥션([ICS-3](https://github.com/cosmos/ibc/tree/main/spec/core/ics-003-connection-semantics))을 통해 체인을 서로 연결한다. 
4. 토큰 전송을 위한 채널([ICS-4](https://github.com/cosmos/ibc/tree/main/spec/core/ics-004-channel-and-packet-semantics))을 구성한다. 
5. . 채널을 통해 전달된 데이터 패킷이 클라이언트를 통해 검증완료되면, 토큰 전송([ICS-20](https://github.com/cosmos/ibc/tree/main/spec/app/ics-020-fungible-token-transfer)) 트랜잭션을 완료한다.

<br>

## IBC Handshaking

체인 간 서로 데이터를 주고 받기 위해 서로가 상대 체인을 신뢰하는 환경을 구성하는 단계이다. 오프체인으로 별도 구성된 릴레이어(Relayer)는 각 체인에 상대 체인에 대한 클라이언트 생성을 요청한다. 클라이언트가 정상 생성되면 클라이언트를 기반으로 체인끼리 서로 통신할 수 있는 연결 통로인 커넥션(Connection)을 생성하고, 커넥션 위에 기능 프로토콜 단위로 데이터 패킷을 주고 받을 수 있는 채널(Channel)을 생성한다. 

커넥션과 채널은 TCP 소켓 연결과 같은 handshaking 과정을 통해 만들어지는데, 각각 4번의 확인 과정을 거쳐 4-way handshaking이라고 한다.

### 커넥션(Connection)

커넥션은 상대 체인에 대한 명확한 식별을 통해, 악의적인 주체가 상대 체인이라 속이고 잘못된 정보를 위조하는 것을 방지하기 위해 생성된다. 커넥션에 대한 정보는 온체인 원장코드에 기록되기 때문에, 별도의 오프체인 시스템에서 이를 관리하기 위한 상호작용이 필요하지 않다.

커넥션은 클라이언트를 기반으로 생성되는데, 체인마다 클라이언트를 가질 경우 하나의 클라이언트는 연결할 체인의 수에 따라서 여러개의 커넥션이 생길 수 있다. 이는 연결되는 체인에 의해 클라이언트가 여러가지 버전의 클라이언트와 연결될 수 있다는 의미로, IBC 프로토콜에서는 프로토콜 버전의 허용 범위를 정하여 클라이언트가 잘못된 검증을 하지 않도록 방지한다.

![4-way handshaking in connection](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/connectionstate.png)

위의 그림과 같이 커넥션은 4-way handshaking 방식으로 연결되며, 체인 A에서 커넥션 요청을 시작하는 경우 아래와 같이 단계별로 진행된다. 여기서 모든 요청 메시지는 릴레이어에 의해 트리거되어 상대방 체인으로 전달된다.

#### 1. ConnOpenInit

1. Handshaking에 앞서, 릴레이어는 MsgUpdateClient를 통해 체인 A의 클라이언트(체인 B) 상태를 최신으로 업데이트한다. MsgUpdateClient에는 해당 체인의 블록 정보와 최신 합의 상태가 포함되어 있다.
2. 체인 A가 OpenInit 메시지를 생성하고, 연결상태를 INIT으로 변경한다.
3. 체인 A는 새로운 커넥션에 대한 식별자(ID)를 만들고, 클라이언트에 추가한다.

> 릴레이어는 체인 A가 생성한 OpenInit을 확인하고, 체인 A의 프로토콜과 상이한 버전이 등록된 경우 해당 요청을 실패로 처리한다. 

#### 2. ConnOpenTry

1. 체인 B는 클라이언트를 통해 체인 A의 신원을 확인한다. 
2. 릴레이어는 MsgUpdateClient를 연결하고자 하는 각 체인에 전송하여 클라이언트를 최신상태로 업데이트하고, 체인 B는 체인 A에 대한 확인을 검증한다.
4. 체인 B의 연결 상태가 TRYOPEN으로 변경된다.

> 체인 A에서 사용 불가능한 프로토콜 버전을 제안할 경우, 연결은 실패한다. 사용 가능한 프로토콜 버전을 제안했을 경우, 체인 B는 제안한 버전을 그대로 수락하거나 사용 가능한 다른 버전을 제안할 수 있다.

#### 3. ConnOpenAck

1. 체인 A는 클라이언트를 통해 체인 B의 신원을 확인한다.
2. 릴레이어는 MsgUpdateClient를 연결하고자 하는 각 체인에 전송하여 클라이언트를 최신상태로 업데이트하고, 체인 A는 체인 B에 대한 확인을 검증한다.
3. 체인 A의 상태가 OPEN으로 변경된다. 이때 체인 B의 연결상태는 TRYOPEN 이어야 한다.
#### 4. ConnOpenConfirm

1. 체인 A와 체인 B에 대한 모든 식별이 마무리 되면, 체인 B의 연결상태를 OPEN으로 변경하고 커넥션 생성이 마무리된다.

### 채널

채널은 두 체인 간 한쌍의 연결 통로를 나타내며, 이를 통해 서로 다른 체인간 데이터 패킷이 이동할 수 있다. 채널은 커넥션을 기반으로 생성되며, 하나의 커넥션 위에 다수의 채널이 생성될 수 있다.

채널은 체인과 체인을 전체로 연결하지 않고, 포트(Port)라고 하는 체인 내 어플리케이션 모듈을 나타내는 ID를 기반으로 연결된다. 중요한 점은 채널은 항상 두 체인 간의 특정 **포트**를 연결하는데, 예를 들어 하나의 채널은 체인 A와 체인 B에서 각각 하나의 포트에 연결되며, 이를 통해 특정 애플리케이션 간의 데이터 교환을 가능하게 합니다.

아래 그림과 같이, 체인 A의 채널 2와 체인 B의 채널 40을 transfer 포트로 연결하거나 체인 A의 채널 5와 체인 B의 채널 80을 transfer 포트로 연결하여 토큰 전송 어플리케이션 데이터를 주고 받을 수 있다. 중요한 점은 하나의 채널이 2개 이상의 포트로 구성될 수는 없다는 것인데, 체인 A의 채널 2가 체인 B에 있는 여러 채널들과 모두 연결하는 것은 불가능하다.

![ICS-20 cross-chain transfers](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/transferoverview.png)

아래 그림과 같이, 채널도 커넥션처럼 4-way handshaking을 통해 구성된다. 

![4-way handshaking in channel](https://tutorials.cosmos.network/resized-images/1200/academy/3-ibc/images/channelhandshake.png)

#### 1. ChanOpenInit

1. 릴레이어가 체인 A의 상태를 INIT으로 설정한다. 
2. 어플리케이션마다 정의된 콜백을 통해 포트의 유효성을 검증하고, 채널에서의 패킷 처리 옵션을 확인한다. 

> 채널이 전달되는 패킷을 처리하는 방식에는 order/unorder가 존재한다.
> - order : 전송된 패킷이 순서대로 정확하게 전달되는 채널
> - unorder : 패킷이 전송된 순서와 다르게 전달될 수 있는 채널

#### 2. ChanOpenTry

1. 체인 B의 상태를 TRY로 변경한다.
2. 어플리케이션마다 정의된 콜백을 통해 포트의 유효성을 검증하고, 채널에서의 패킷 처리 옵션을 확인한다. 

#### 3. ChanOpenAck

1. 체인 A의 상태를 OPEN으로 변경한다.
2. 채널 상에서 주고받는 데이터 구조의 일치를 위해, 사용할 어플리케이션 버전을 이 시기에 결정한다.
#### 4. ChanOpenConfirm

1. 체인 B의 상태를 OPEN으로 변경한다.

<br>

## Token Transfer

IBC 내에서 토큰을 보내는 체인은 소스(source) 체인, 토큰을 받는 체인은 싱크(sink) 체인이라고 한다.

- 소스 체인 : 
- 토큰을 보내는, 자산이나 데이터의 전송을 시작하는 체인
- 전송하려는 자산(예: ATOM)을 잠금(escrow)한다. 잠금 상태인 자산은 소스 체인 내에서 더이상 사용되지 않는다.

싱크 체인
- 토큰을 받는, 소스 체인으로부터 자산이나 데이터를 수신하는 체인
- 소스 체인으로부터 전송된 토큰을 받아 해당 자산을 발행(mint)하거나 사용자가 접근할 수 있도록 한다.
- 소스 체인으로부터 전달받은 자산을 검증한 후, 수신자의 계정에 할당한다.

### 전송 절차

소스 체인과 싱크 체인이 서로 토큰을 주고받으려면, 양 체인이 IBC Handshaking을 통해 커넥션과 채널 생성을 완료해야 한다. 기본적으로 채널마다 사용하는 포트는 ICS-20 표준인 transfer를 사용한다.




### 경로

IBC에서 전송되는 토큰은 {Port}/{Channel}/{denom} 형태로, 소스 체인의 어떤 포트와 채널을 통해 토큰이 전송되었는지를 나타낸다. 해당 토큰이 이동하는 경로는 ibc/<hash of {Port}/{Channel}/{denom}> 형태로 표현된다. 

예로 체인 A에서 체인 B로 토큰이 전송되었을 때, ibc/<hash of transfer/channel-10/uatom> 형태로 기록된 경로는 아래와 같은 의미를 갖는다.
- transfer : 체인 A에서 토큰 전송에 사용한 포트(ICS-20 표준)
- channel-10 : 체인 A에서 토큰 전송에 사용된 채널
- uatom : 토큰의 denom

만약 토큰이 여러 경로를 거쳐서 전송되는 다중 홉(multi-hop) 구조를 갖는 경우, 해당 경로가 앞에 연결되는 구조로 작성된다.

예로 체인 A에서 체인 B로, 그리고 체인 B에서 체인 C로 토큰이 전송되었을 때, ibc/<hash of transfer/channel-20/ibc/<hash of transfer/channel-10/uatom>> 형태와 같이 각 초기 경로 앞에 중간에 거쳐간 경로들이 앞에 붙어서 연결되는 구조로 작성된다. 

전송된 토큰이 다시 반대로 돌아가는(반환)경우, 해당 전송 경로는 삭제되며 최종적으로 전송을 시작한 체인까지 돌아오면 토큰의 denom인 'uatom'만이 남는다.
<!-- 
### 토큰 전송 및 반환 시나리오

소스 체인에서 싱크 체인으로 자산을 이동하는 것을 전송, 전송된 자산이 반대로 싱크체인에서 소스 체인으로 이동하는 것을 반환이라고 할 때 아래와 같은 순서로 진행될 수 있다. 이 때 체인에서 사용하는 포트는 ICS-20 표준 'transfer'로 하고, 소스 체인은 channel-10, 싱크 체인은 channel-20으로 통신한다고 가정한다. 자산은 uatom을 주고받는다.

#### 1. Escrow
- 소스 체인에서 전송하고자 하는 자산의 양만큼 잠금(escrow) 처리된다.
- 경로 정보 : 변경되지 않음 

#### 2. Mint
- 싱크 체인에서 전송된 자산에 해당하는 양만큼 발행(mint)한다.
- 경로 정보: ibc/<hash of transfer/channel-10/uatom>

#### 3. Unescrow
- 자산이 싱크 체인에서 소스 체인으로 돌아오고, 소스 체인은 잠겨있던 자산 중 돌아온 양 만큼을 해제(unescrow)된다.
- 경로 정보 : 토큰이 원래 전송된 경로로 돌아오기 때문에, 경로 정보는 역순으로 제거된다. 다중 홉의 경우 escrow 시점에 작성된 경로가 삭제되며, 아닌 경우 원래의 denom인 'uatom'이 복원된다.

#### 4. Burn
- 싱크 체인이 발행했던 토큰을 소각(burn)한다.
- 경로 정보 : 싱크 체인에 등록되었던 경로 정보가 삭제되고, 해당 토큰은 더이상 존재하지 않는다. -->

### 패킷 전송 흐름도

양 체인간 채널 설정이 완료되면 서로 토큰을 주고받을 수 있다. 








