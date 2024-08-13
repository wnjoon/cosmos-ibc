# IBC 프로토콜 보안 모델 분석

IBC(Inter-Blockchain Communication) 프로토콜은 서로 다른 블록체인 간 안전한 통신을 가능하게 하는 표준화된 프로토콜이다. 본 문서에서는 서로 다른 체인 간 신뢰할 수 있는 통신을 보장하기 위해 IBC에서 사용하는 보안 모델과 CometBFT+PoS 환경에서 발생 가능한 다양한 상황에 대해 기술한다.

<br>

## IBC 보안 모델

IBC 프로토콜은 서로 다른 두 체인간 안전한 통신을 유지하기 위해 서로가 상대 체인을 신뢰할 수 있는 검증 환경을 제공하기 위해 존재한다. 이를 위해 IBC 프로토콜의 구성 요소 중 하나인 클라이언트를 기반으로 서로 다른 두 체인의 신뢰성을 보장한다. 클라이언트에서 네트워크의 검증 완료를 확인하면, IBC 프로토콜은 연결된 양 쪽 체인을 직접 확인할 필요가 없다. 

<br>

## 클라이언트

통신하고자 하는 상대 체인의 블록 헤더 정보와 검증자 집합(validator set)을 포함한다. 상대 체인에서 새로운 블록이 생성되었음을 [릴레이어](#릴레이어)가 확인하면, 새로운 블록에 대한 검증자 집합 정보를 통해 클라이언트가 아직도 유효한 상태인지를 검증한다. 모든 클라이언트는 고유의 식별자(Client ID)를 갖는다.

> ### 릴레이어
> 오프체인 형태로 존재하며, 연결된 양쪽 체인이 서로를 신뢰할 수 있는 다양한 방법을 제공한다. 릴레이어에서 각 체인과 연결된 IBC 모듈로 클라이언트 생성을 요청하고, 정상적으로 생성이 완료되면 클라이언트를 서로 연결하는 Connection을 구성한다. 그리고 Connection 위에 각 체인의 어플리케이션 모듈에서 상대 체인으로 데이터 패킷을 전송할 수 있는 Channel을 생성한다.  

IBC 보안 모델은 클라이언트를 기반으로 동작하기 때문에, Connection과 Channel 모두 Client ID를 기반으로 생성된다. 그리고 클라이언트를 기반으로 각 체인이 연결되기 때문에, Client ID는 IP주소와 같은 역할을 하며 각 체인의 ID는 DNS에 비유되기도 한다.

<br>

## ConsensusState

ConsensusState는 클라이언트의 구성요소 중 하나로, IBC 프로토콜에서 클라이언트가 연결된 상대 체인의 상태를 추적하고 검증하기 위해 사용된다. 

```go
interface ConsensusState {
  timestamp: uint64 
  nextValidatorsHash: []byte
  commitmentRoot: []byte 
}
```

### timestamp

블록이 생성된 시점을 나타내며, UNIX 시간으로 표현된다. 체인간 데이터를 전송할 때, 타임스탬프 값을 기준으로 데이터의 유효성을 판단한다. 

### nextValidatorsHash

다음 블록을 검증할 검증자 집합(validator set)의 해시값을 나타낸다. 클라이언트는 해시값을 통해 다음 블록에서 어떤 검증자들이 참여할지 확인할 수 있기 때문에, 블록체인의 무결성을 유지하고 악의적인 행위가 발생하지 않도록 막을 수 있다.

```go
interface Header extends TendermintSignedHeader {
  identifier: string
  validatorSet: List<Pair<Address, uint64>>
  trustedHeight: Height
  trustedValidatorSet: List<Pair<Address, uint64>>
}
```































-------

### IBC의 목적과 클라이언트의 중요성

IBC 프로토콜은 특정 체인에서 발생한 트랜잭션을 다른 체인으로 안전하게 전송하기 위해 존재한다. 

블록체인이 IBC를 통해 다른 체인으로 데이터를 전송하려고 하는 목적으로 생성된 트랜잭션을 패킷으로 변환하고, 이를 상대 블록체인으로 전달한다는 내용만 관리하기 때문에 체인의 종류 뿐만 아니라 체인 별로 독립적으로 관리되는 트랜잭션은 다루지 않는다.

그러므로 클라이언트가 항상 최신 상태로 유지되고 지속적으로 유효한 정보를 업데이트하고 있다면, 연결된 체인은 신뢰할 수 있다고 판단할 수 있다. 즉 체인간 직접적인 신뢰 관계가 아닌, 클라이언트 간 신뢰에 기반을 둔 구조를 가질 수 있게 된다.

### 클라이언트 생성 과정

서로 다른 두 체인(A, B)과 릴레이어가 존재한다고 가정한다.

> **릴레이어(Relayer)**
> <br> <br>
> 별도의 오프체인으로 구성되며, 각 체인은 네트워크에 직접 패킷을 전달하지 않고 릴레이어를 통해 전달한다.
릴레이어는 클라이언트의 생성을 요청하고, 생성된 클라이언트의 최신 블록 헤더 정보와 검증자 집합을 지속적으로 전달하여 클라이언트의 상태를 항상 최신으로 유지한다.

1. 릴레이어가 A 체인에 B 체인에 대한 클라이언트 생성을 요청한다.
2. A 체인의 IBC 모듈이 CreateClient 함수를 호출하고 클라이언트를 생성하고 클라이언트 ID를 반환한다.
3. 생성된 클라이언트(B)는 A 체인의 상태 저장소에 저장된다.

<br>

CreateClient 함수의 실행 과정은 아래와 같다.

```go


  // 클라이언트 타입 검증 (ex: Tendermint)
  if clientType == exported.Localhost {...}

  // 클라이언트 ID 생성
	clientID := k.GenerateClientIdentifier(ctx, clientType)

  // Client ID에 해당하는 새로운 클라이언트를 관리하기 위한 모듈로, 클라이언트 타입에 따라 결정
	clientModule, err := k.Route(ctx, clientID)

  // clientState, consensusState 체인 상태 저장소에 기록
	if err := clientModule.Initialize(ctx, clientID, clientState, consensusState); err != nil {...}
    ...
}
```

> **상태 저장소(State Storage)**
> <br> <br>
> 블록체인에서 각 노드가 블록체인의 상태를 저장하고 관리하는 데이터베이스를 의미하며, IBC 프로토콜이 블록체인 간의 통신을 안전하고 일관되게 유지할 수 있도록 한다. 상태 저장소에는 클라이언트 외에도 connection, channel, packet, consensus state, event, logging 등 다양한 정보들이 저장된다.

<br>

Client ID
- 생성된 클라이언트의 고유 식별자로, 이후 해당 클라이언트를 참조하거나 업데이트할 때 사용한다.
- 체인 ID는 특정 체인을 식별할 때 사용되며, 클라이언트 ID는 IP 주소, 체인 ID는 DNS로 비유된다.

ClientState

```go
// 클라이언트 타입마다 다르며, 아래는 cometBFT 기반의 ClientState
interface ClientState {
  chainID: string

  // 검증에 필요한 validator 비율. cometBFT는 2/3으로 정의
  trustLevel: Rational 

  // 제출된 헤더의 업데이트 가능 시간
  // 시간 초과시 클라이언트가 만료되며, 복구하려면 별도의 거버넌스 제안 필요
  trustingPeriod: uint64 

  unbondingPeriod: uint64
  latestHeight: Height
  frozenHeight: Maybe<uint64>
  upgradePath: []string

  // 각 체인별 노드의 위치 및 사용하는 시간대의 불일치로 인하여 발생 가능한 시간 차이 허용 범위
  maxClockDrift: uint64 
  proofSpecs: []ProofSpec
}
```

ConsensusState
- 클라이언트에 해당하는 체인의 상태를 나타내며, 머클 증명을 통해 체인의 검증자 집합과 트랜잭션의 유효성을 검증할 때 사용한다.

### 클라이언트 업데이트

두 체인(A, B)가 모두 클라이언트를 생성한 상태로 가정한다.

1. 릴레이어가 A 체인에 클라이언트(B) 업데이트 요청(MsgUpdateClient)을 보낸다.
2. A 체인의 IBC 모듈에서 UpdateClient 함수를 호출하고 클라이언트(B)를 업데이트 한다.
3. B 체인의 최신 블록 헤더와 검증자 집합을 통해 클라이언트(B)를 검증한다.
4. 검증이 성공하면 클라이언트(B)의 변경된 상태가 A 체인의 상태 저장소에 기록된다.
5. 릴레이어는 클라이언트 업데이트가 정상적으로 완료된 것을 확인하고, 이후 데이터 중계를 계속 진행한다.

MsgUpdateClient는 업데이트 할 체인의 정보를 포함하는 Header가 존재하며, 해당 정보를 가지고 클라이언트를 업데이트 한다. 

```go
interface Header extends TendermintSignedHeader {
  identifier: string
  validatorSet: List<Pair<Address, uint64>>
  trustedHeight: Height
  trustedValidatorSet: List<Pair<Address, uint64>>
}
```
- identifier : 클라이언트 식별자
- validatorSet : 최신 블록을 검증한 검증자 집합
- trustedHeight : 신뢰할 수 있는 이전 블록의 높이
- trustedValidatorSet : 신뢰할 수 있는 이전 블록을 검증한 검증자 집합

<br>

## 머클 증명(Merkle Proof)

머클 증명은 블록체인 시스템에서 데이터의 무결성을 확인하기 위한 핵심적인 암호학적 기법이다. IBC 프로토콜에서는 체인 간의 신뢰할 수 있는 데이터 전송을 보장하기 위해 머클 증명을 사용한다. 

### IAVL 트리

IBC에서 머클 증명에 사용되는 머클 트리를 구성하기 위해 사용하는 자료구조로, 내부 노드의 높이차를 1로 보장하며 2 이상 발생할 경우 리밸런싱을 진행하여 최대 O(logn)의 삽입, 탐색, 삭제 연산속도를 보장하는 방식이다. 

- 리프(leaf) 노드 : 머클 트리의 가장 하위에 위치하며, 검증하려는 데이터의 해시값을 저장한다.
- 형제(aunt) 노드 : 리프 노드와 같은 레벨에 있는 노드로, 좌우 연관된 리프 노드와 결합하여 상위 노드로 해시값을 전파한다.
- 루트(root) 노드 : 머클 루트라고도 부르며, 머클 트리에 존재하는 모든 트랜잭션이 포함된 상태를 나타내는 최상위 노드를 의미한다.

### 클라이언트와 머클 증명

클라이언트는 ConsensusState를 통해 상대 체인의 상태와 검증자 집합의 유효성을 검증한다.

```go
interface ConsensusState {
  timestamp: uint64
  nextValidatorsHash: []byte
  commitmentRoot: []byte 
}
```

#### commitmentRoot

IBC 프로토콜에서 클라이언트는 상대 체인의 상태를 추적하고 검증하기 위해 CommitmentRoot(머클 루트)를 포함한 상태 정보를 저장하고, 이를 통해 상대 체인에서 전송된 패킷의 무결성을 보장한다.

#### nextValidatorsHash






<br><br>

## 참고

- [AVL Tree 시뮬레이터](https://cmps-people.ok.ubc.ca/ylucet/DS/AVLtree.html)