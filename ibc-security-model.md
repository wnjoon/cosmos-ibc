# IBC의 보안 모델 분석

IBC(Inter-Blockchain Communication) 프로토콜은 서로 다른 블록체인 간의 안전한 통신을 가능하게 하는 표준화된 프로토콜이다.  
본 문서에서는 서로 다른 체인 간 신뢰할 수 있는 통신을 보장하기 위해 IBC에서 사용하는 보안 모델과 CometBFT+PoS 환경에서 발생 가능한 다양한 상황을 기술한다.

<br>

## IBC 보안 모델

체인 간의 연결에서 신뢰를 유지하기 위한 모델 방법으로, 클라이언트에 저장된 정보와 상대 체인으로부터 전달된 데이터를 머클 증명을 통해 검증하여 안전한 통신이 가능하도록 한다.

<br>

## 클라이언트

상대 체인의 블록 헤더 정보와 검증자 집합(validator set)을 포함하고 있으며, 이를 지속적으로 확인하여 상대 체인의 신뢰 여부를 검증하기 위해 사용한다. 상대 체인의 모든 상태정보를 포함하지 않기 때문에 자원을 효율적으로 사용한다는 장점이 있다.

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
func (k *Keeper) CreateClient(
    ctx sdk.Context, 
    clientType string, 
    clientState, 
    consensusState []byte
) (string, error) {

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
- 생성된 클라이언트의 고유 식별자로, 이후 해당 클라이언트를 참조하거나 업데이트할 때 사용
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
```go
interface ConsensusState {
  // 상대 체인에서 생성된 블록의 생성시간
  timestamp: uint64

  nextValidatorsHash: []byte
  commitmentRoot: []byte 
}
```
- nextValidatorsHash : 다음 블록을 검증할 검증자 집합(validator set)의 해시값
- commitmentRoot : 검증자 집합을 제외한 트랜잭션, 계정 상태, 스마트 컨트랙트 상태 등 실제 체인에서 발생하는 상태에 대한 해시값

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

## 머클 증명

IBC 내에서 머클증명은 실제 체인에서 전송되는 데이터 패킷 뿐 아니라 클라이언트 내부의 데이터를 검증하기 위해 사용된다.

