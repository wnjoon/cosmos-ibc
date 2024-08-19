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

체인 간 서로 데이터를 주고 받기 위해 서로가 상대 체인을 신뢰하는 환경을 구성하는 단계이다. 오프체인으로 별도 구성된 릴레이어(Relayer)는 각 체인에 상대 체인에 대한 클라이언트 생성을 요청한다. 클라이언트가 정상 생성되면 클라이언트를 기반으로 체인끼리 서로 통신할 수 있는 연결 통로인 커넥션(Connection)을 생성하고, 커넥션 위에 기능 프로토콜 단위로 데이터 패킷을 주고 받을 수 있는 채널(Channel)을 생성한다. 

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

### 채널(Channel)

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

설명에 앞서, 토큰을 전송하거나 받는 역할에 따라 2가지 종류의 체인으로 구분할 수 있다.

소스 체인
- 자산이나 데이터를 전송하는 체인
- 전송하고자 하는 자산의 양만큼 소스 체인에서 잠금(escrow)되며, 해당 자산은 해제(unescrow)되기 전까지 소스 체인 내에서 사용되지 않는다.

싱크 체인
- 자산이나 데이터를 전송받는 체인
- 소스 체인으로부터 전송된 토큰의 가치에 해당하는 바우처 토큰을 발행(mint)한다.
- 발행된 자산이 다시 소스 체인으로 돌아가는 경우, 해당 자산을 소각(burn)한다.

### Token Denomination

IBC 내에서 토큰은 `{Port}/{Channel}/{denom}`으로 표현된다. 해당 의미는 자산(denom)이 어떤 채널(channel)과 포트(port)에서 전송되었는지를 나타낸다.  

![Token denomination example](https://tutorials.cosmos.network/resized-images/600/academy/3-ibc/images/sourcetosink.png)

위의 그림과 같은 구조에서 체인 A가 체인 B로 100개의 ATOM을 전송할 경우, 체인 A에서 초기 전송된 토큰은 uatom이고, 체인 B에서 새롭게 발행되는 토큰은 `ibc/<hash of transfer/channel-40/uatom>`으로 표현된다. 

그렇다면 체인 A에서 체인 C로 자산이 이동되는데, 중간에 체인 B가 존재하는 다중 홉(multi-hop) 구조에서는 어떻게 토큰이 표현될까? 지나가는 통로를 `port/channel-id/...` 식으로 앞에 연결하면 된다. 예를 들어 체인 B와 연결된 체인 C의 채널이 channel-50이라고 가정한다면, 해당 토큰은 최종적으로 `ibc/<hash of transfer/channel-50/transfer/channel-40/uatom>`으로 표현된다.

반대로 전송되었던 토큰을 반대로 전송(반환)하는 구조라면, `port/channel-id/...`로 표현된 토큰의 경로가 하나씩 제거되면 된다. 위의 예시에서 체인 C로 전달되었던 토큰이 다시 체인 A로 전송된 최종 결과는 denom인 uatom이 된다.

### 전송 절차

ICS-20 토큰의 전송은 크게 5단계로 진행된다.
1. 소스 체인에서 토큰 전송 트랜잭션 생성
2. 트랜잭션에 대한 패킷 생성
3. 싱크체인의 패킷 수신 및 검증
4. 트랜잭션 처리
5. 소스 체인의 트랜잭션 완료 처리

#### 1. 소스 체인에서 토큰 전송 트랜잭션 생성

토큰 전송 트랜잭션 생성에 앞서 아래와 같은 전제상황이 포함되어야 한다.
- 어플리케이션이 동작하는 체인은 cometBFT 기반의 합의 알고리즘을 사용해야 하며, 2/3 이상의 voting power를 갖는 정상적인 검증자 집합이 유지되어야 한다.
- 어플리케이션은 토큰 전송(ICS-20)에 해당하는 프로토콜을 기반으로 동작해야 한다.
- 소스 체인과 싱크 체인 모두 상대 체인을 검증하기 위한 클라이언트를 구성하고, 해당 클라이언트로부터 상대 체인의 최신 상태가 지속적으로 업데이트 되어야 한다.
- 체인 간 커넥션 연결이 완료되어야 한다.
- 소스 체인과 싱크 체인은 ICS-20 기반 토큰 전송에 해당하는 포트 ID인 transfer로 채널이 연결되어 있어야 한다.
- 

트랜잭션 생성이 완료되면 소스 체인은 해당 트랜잭션의 상태를 머클 트리에 저장하고 트리의 루트 해시 값인 commitmentRoot를 갱신한다. commitmentRoot는 소스 체인에서 패킷이 정상적으로 전송되었음을 나타내는 증거로 사용된다.

#### 2. 트랜잭션에 대한 패킷 생성

소스 체인이 생성한 [패킷](https://github.com/cosmos/ibc-go/blob/main/modules/core/04-channel/types/channel.pb.go)은 릴레이어에게 전달된다. 패킷의 내부 구조는 아래와 같다.

```go
// Packet defines a type that carries data across different chains through IBC
type Packet struct {
	// number corresponds to the order of sends and receives, where a Packet
	// with an earlier sequence number must be sent and received before a Packet
	// with a later sequence number.
	Sequence uint64 `protobuf:"varint,1,opt,name=sequence,proto3" json:"sequence,omitempty"`
	// identifies the port on the sending chain.
	SourcePort string `protobuf:"bytes,2,opt,name=source_port,json=sourcePort,proto3" json:"source_port,omitempty"`
	// identifies the channel end on the sending chain.
	SourceChannel string `protobuf:"bytes,3,opt,name=source_channel,json=sourceChannel,proto3" json:"source_channel,omitempty"`
	// identifies the port on the receiving chain.
	DestinationPort string `protobuf:"bytes,4,opt,name=destination_port,json=destinationPort,proto3" json:"destination_port,omitempty"`
	// identifies the channel end on the receiving chain.
	DestinationChannel string `protobuf:"bytes,5,opt,name=destination_channel,json=destinationChannel,proto3" json:"destination_channel,omitempty"`
	// actual opaque bytes transferred directly to the application module
	Data []byte `protobuf:"bytes,6,opt,name=data,proto3" json:"data,omitempty"`
	// block height after which the packet times out
	TimeoutHeight types.Height `protobuf:"bytes,7,opt,name=timeout_height,json=timeoutHeight,proto3" json:"timeout_height"`
	// block timestamp (in nanoseconds) after which the packet times out
	TimeoutTimestamp uint64 `protobuf:"varint,8,opt,name=timeout_timestamp,json=timeoutTimestamp,proto3" json:"timeout_timestamp,omitempty"`
}
```
- SourcePort : 토큰을 전송하는 애플리케이션의 포트 (transfer)
- SourceChannel :  패킷 전송에 사용할 채널 (위의 그림에서 channel-2)
- DestinationPort : 싱크 체인에서 패킷을 수신할 포트 (transfer)
- DestinationChannel :  싱크 체인에서 패킷을 수신할 채널 (위의 그림에서 channel-40)
- Data : 실제로 전송되는 토큰과 관련된 정보(수량, 수신자 주소 등)

#### 3. 싱크체인의 패킷 수신 및 검증

릴레이어는 소스 체인에서 생성된 패킷과 해당 트랜잭션의 Merkle Proof를 함께 싱크 체인으로 전달한다. 싱크 체인에 있는 소스 체인에 대한 클라이언트는 Merkle Proof를 통해 해당 패킷이 소스 체인에서 유효하게 생성되었는지, 조작되지 않았는지 등을 검증한다. 검증이 완료된 패킷은 채널을 통해 수신된다. 

#### 4. 트랜잭션 처리

채널을 통해 수신된 패킷을 받은 싱크 체인은, 해당 패킷에 포함된 정보를 기반으로 토큰을 처리한다. 위의 예시처럼 체인 A에서 체인 B로 토큰을 이동하는 경우, 소스 체인(체인 A)에서 전송한 토큰에 해당하는 가치를 지닌 바우처 토큰을 싱크 체인(체인 B)에서 발행(mint)한다. 발행된 토큰은 수신자에게 전달된다.

#### 5. 소스 체인의 트랜잭션 완료 처리

싱크 체인에서 패킷 처리가 완료되면, 해당하는 성공 내역에 대한 commitmentRoot가 업데이트 된다. 싱크 체인은 갱신된 commitmentRoot 값을 소스 체인에 알리기 위한 응답 패킷(Acknowledgement)을 생성한다.

릴레이어는 해당 응답 패킷을 소스 체인으로 전달하고, 소스 체인은 전달 받은 패킷과 같이 전송된 Merkle Proof를 사용하여 싱크 체인에서 전달한 내용이 정상적인 내용인지 확인한다. 정상임이 확인되면 전송 요청에 대한 트랜잭션을 최종 완료 상태로 표시한다.

<br><br>

## 참고
- [Inter-Blockchain Communication 구조와 Relayer](https://consensusmymem.tistory.com/47)