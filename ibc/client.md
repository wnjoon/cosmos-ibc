# Client

## 개요

- 각 체인은 상대 체인의 상태를 검증하기 위한 light client(이하 client)를 생성
- 체인은 상대 체인과 맺은 connection을 통해 지속적으로 상태를 검증하고 연결을 관리
- 일반적으로 Relayer를 통해 client 생성을 요청하며, relayer가 없는 경우 체인을 통해 직접 생성하기도 함 (확인 필요)

## 인터페이스


## 클라이언트 생성

- [modules/core/02-client/keeper/client.go](https://github.com/cosmos/ibc-go/blob/v7.0.0/modules/core/02-client/keeper/client.go)

```go
// module/core/02-client/keeper/client.go
func (k *Keeper) CreateClient(ctx sdk.Context, clientType string, clientState, consensusState []byte) (string, error) {
	
    // 클라이언트 타입 검증
    if clientType == exported.Localhost {
		return "", errorsmod.Wrapf(types.ErrInvalidClientType, "cannot create client of type: %s", clientType)
	}

    // Client ID(클라이언트 식별자) 생성
	clientID := k.GenerateClientIdentifier(ctx, clientType)

    // clientModule : Client ID에 해당하는 새로운 클라이언트를 관리하기 위한 모듈로, 클라이언트 타입에 따라 결정
    // 예로, Tendermint 기반의 클라이언트인 경우 Tendermint 클라이언트 모듈이 선택됨(내부 GetRoute 함수)
	clientModule, err := k.Route(ctx, clientID)
	if err != nil {
		return "", err
	}

    // 클라이언트 최초 등록
    // 클라이언트 상태(clientState), 합의 상태(consensusState)는 체인 상태 저장소에 기록
	if err := clientModule.Initialize(ctx, clientID, clientState, consensusState); err != nil {
		return "", err
	}

	if status := clientModule.Status(ctx, clientID); status != exported.Active {
		return "", errorsmod.Wrapf(types.ErrClientNotActive, "cannot create client (%s) with status %s", clientID, status)
	}

	initialHeight := clientModule.LatestHeight(ctx, clientID)
	k.Logger(ctx).Info("client created at height", "client-id", clientID, "height", initialHeight.String())

	defer telemetry.ReportCreateClient(clientType)
	emitCreateClientEvent(ctx, clientID, clientType, initialHeight)

	return clientID, nil
}
```

Client ID
- IBC 네트워크 상에서 일종의 IP 주소와 같은 개념으로, IBC 네트워크에서 각 체인의 Chain ID는 DNS와 같은 의미를 가지며 이를 식별자로 사용하지는 않음

ClientState
- 생성되는 클라이언트의 타입마다 다른 값을 가지며, 예시로 CometBFT는 아래와 같은 구조를 갖는다.
```go
interface ClientState {
  chainID: string
  trustLevel: Rational // 검증에 필요한 validator 비율. cometBFT는 2/3으로 정의
  trustingPeriod: uint64 // 제출된 헤더가 해당 시간 내 업데이트 되지 않으면 클라이언트 만료. 복구하려면 별도의 거버넌스 제안 필요
  unbondingPeriod: uint64
  latestHeight: Height
  frozenHeight: Maybe<uint64>
  upgradePath: []string
  maxClockDrift: uint64 // 각 체인별 노드의 위치 및 사용하는 시간대의 불일치로 인하여 발생 가능한 시간 차이를 무시할 수 있도록 허용하는 값
  proofSpecs: []ProofSpec
}
```
- maxClockDrift, trustLevel은 클라이언트가 생성된 이후에는 변경할 수 없음

ConsensusState
- 상대 체인의 실시간 상태를 나타내며, 상대 체인에서 새로운 블록이 생성될때마다 업데이트
```go
interface ConsensusState {
  timestamp: uint64
  nextValidatorsHash: []byte
  commitmentRoot: []byte 
}
```
- timestamp : 상대 체인에서 생성된 블록의 생성시간
- nextValidatorsHash : 다음 블록을 검증한 validator set의 해시값

    > Validator Set에 포함된 검증자들의 ID, 공인키, 그리고 그들의 투표 등의 정보로부터 해시 알고리즘을 통해 고유한 해시값이 생성됨

- commitmentRoot : 상대 체인의 merkle root값(체인의 상태)
- 이전 블록에 대한 신뢰가 기반되기 때문에, 시스템의 효율성 및 네트워크 성능을 위해 genesis block부터 검증하지 않음

## 클라이언트 업데이트

- ConsensusUpdate를 통해서 최근 업데이트 한 블록부터 최신 블록까지 확인
- 업데이트할 체인의 헤더가 포함된 MsgUpdateClient를 Relayer에서 전송
    - IBC에서는 클라이언트 타입, 클라이언트 ID 외 다른 정보만 제공하며, 기타 필요한 정보는 각 클라이언트 구현체 내에 존재

```go
interface TendermintSignedHeader {
  height: uint64
  timestamp: uint64
  commitmentRoot: []byte
  validatorsHash: []byte
  nextValidatorsHash: []byte
  signatures: []Signature 
}
}
```
- TendermintSignedHeader : 확인하고자 하는 상대 체인의 블록 정보
- 헤더에 표현된 validator set 2/3의 서명이 커밋되어 cometBFT 합의가 보장됨

```go
interface Header extends TendermintSignedHeader {
  identifier: string
  validatorSet: List<Pair<Address, uint64>>
  trustedHeight: Height
  trustedValidatorSet: List<Pair<Address, uint64>>
}
```
- validatorSet : 실제 블록을 검증한 validator set 정보
    - 예를 들어 100번의 블록을 업데이트하고자 할 때, 100번 블록을 검증한 validator set
- trustedValidatorSet : consensusState에서 신뢰할 수 있는 가장 최신의 validator set
    - 예를 들어 50번의 블록까지 현재 업데이트 되어있다고 하면, 해당 블록의 validator set
    - consensusState의 nextValidatorsHashd와 동일
- trustedHeight : 클라이언트 내 consensusState의 높이(인덱스)
    - type getConsensusState = (height: Height, proof?: bytes) => ConsensusState

## 패킷 커밋 검증

- Relayer는 패킷을 서로 다른 체인에 전송하기 전 항상 MsgUpdateClient로 클라이언트를 업데이트
```go
function verifyPacketCommitment(
  connection: ConnectionEnd,
  height: Height,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  commitmentBytes: bytes
) {
  clientState = queryClientState(connection.clientIdentifier)
  path = applyPrefix(
    connection.counterpartyPrefix,
    packetCommitmentPath(portIdentifier, channelIdentifier, sequence))
  return verifyMembership(
    clientState,
    height,
    connection.delayPeriodTime,
    connection.delayPeriodBlocks,
    proof,
    path,
    commitmentBytes)
}
```