# Interchain Concepts

## A Blockchain App Architecture

## Accounts

- Cosmos SDK에서 키는 keyring이라는 개체에 저장되고 관리된다.
- secp256k1, secp256r1, tm-ed25519 제공
- 주소(address)는 [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md)을 사용하여 공개키로부터 파생된다. 
    - AccAddress : 사용자
    - ValAddress : 검증자(validator)
    - ConsAddress : 합의에 참여하는 검증자 노드

## Transactions

Transaction Object

```go
Tx interface {
    // Gets the all the transaction's messages.
    GetMsgs() []Msg

    // ValidateBasic does a simple and lightweight validation check that doesn't
    // require access to any other information.
    ValidateBasic() error
}
```
- GetMsgs: tx를 unwrap하고 내부 sdk.Msg 목록 반환. 하나의 트랜잭션엔 다수의 메시지 포함
- ValidateBasic: 트랜잭션의 유효성 확인을 위해 ABCI 메시지(CheckTx, DeliverTx)에서 사용하는 경량의 상태 비저장 검사 포함
- Tx 구성에는 TxBuilder 사용

```go
TxBuilder interface {
    GetTx() signing.Tx

    SetMsgs(msgs ...sdk.Msg) error
    SetSignatures(signatures ...signingtypes.SignatureV2) error
    SetMemo(memo string)
    SetFeeAmount(amount sdk.Coins)
    SetGasLimit(limit uint64)
    SetTimeoutHeight(height uint64)
    SetFeeGranter(feeGranter sdk.AccAddress)
}
```

### Messages

> <u>트랜잭션 메시지와 ABCI 메시지는 다름</u>

- 메시지(sdk.Msg)는 특정 모듈에서 상태를 변경하는 역할을 하는 객체이다.
- 모듈 개발자는 Protobuf에서 Msg라는 서비스로 이 메시지를 정의하고, 그에 맞는 MsgServer를 구현한다.
- 각 메시지는 모듈 내에서 정의된 하나의 Protobuf 서비스와 연결되어 있으며, 해당 메시지가 처리되면 그에 따른 상태 변경이 이루어진다.
- Cosmos SDK 앱은 메시지를 자동으로 적절한 서비스에 연결해, 알맞은 메서드가 호출되도록 한다.
- 이 구조는 모듈 개발자에게 더 많은 책임을 부여하지만, 애플리케이션 개발자는 공통 기능을 재사용할 수 있어 상태 전환 로직을 매번 새로 구현하지 않아도 된다.
- 메시지는 상태 변경에 필요한 정보를 담고 있으며, 거래의 나머지 메타데이터는 TxBuilder와 Context에 저장된다.

### Signing Transactions

- 트랜잭션의 모든 메시지는 GetSigner에서 지정한 주소로 서명된다.
- SIGN_MODE_DIRECT: 모든 서명자가 서명하면 BodyBytes, AuthInfoBytes, 서명값이 TxRaw로 수집되고 직렬화된 바이트가 네트워크를 통해 브로드캐스팅됨

### Generating transactions

트랜잭션 생성에 사용되는 값
- Msgs : 트랜잭션에 포함된 메시지 배열
- GasLimit : 사용자가 지출할 가스 금액을 계산하는 방법에 대한 옵션
- Memo : 거래와 함께 보낼 메모(설명)
- FeeAmount : 사용자가 수수료로 지불할 의사가 있는 최대 금액
- TimeOutHeight : 트랜잭션이 유효할 때 까지의 블록 높이
- Signature : 거래에 참여한 모든 서명자의 서명 배열

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder
```

<br>

## [Messages](https://tutorials.cosmos.network/academy/2-cosmos-concepts/4-messages.html)

메시지는 Cosmos SDK의 모듈에서 처리하는 두 가지 기본 개체 중 하나로(다른 하나는 쿼리로, 메시지는 상태를 알리고 이를 변경할 가능성이 있지만 쿼리는 모듈 상태를 검사하며 항상 읽기 전용), Cosmos SDK에서 트랜잭션에는 하나 이상의 메시지가 포함된다. 모듈은 합의 계층에 의해 트랜잭션이 블록에 포함된 후 메시지를 처리한다.

### Message and Transaction Lifecycle

블록에 포함된(확인된) 트랜잭션은 해석을 위해 Cosmos SDK 어플리케이션으로 전달된다. 각 메시지는 MsgServiceRouter를 사용하여 BaseApp을 통해 적절한 모듈로 라우팅된다. BaseApp은 트랜잭션에 포함된 각 메시지를 디코딩한다. 각 모듈에는 수신된 각 메시지를 처리하기 위한 MsgService가 있다.

- MsgService : Protobuf Msg 서비스를 정의하는 방식을 추천하며, 각 모듈에는 tx.proto에 정의된 정확히 하나의 Protobuf Msg 서비스가 존재하고 모듈의 각 메시지 유형에 대한 rpc 서비스 메서드가 존재한다.

```go
// Msg defines the bank Msg service.
service Msg {
  // Send defines a method for sending coins from one account to another account.
  rpc Send(MsgSend) returns (MsgSendResponse);

  // MultiSend defines a method for sending coins from some accounts to other accounts.
  rpc MultiSend(MsgMultiSend) returns (MsgMultiSendResponse);
}
```
- 각 Msg 서비스 메서드에는 sdk.Msg 인터페이스와 protobuf 응답을 구현하기 위한 MsgSend가 존재한다.

<br>

## [Modules](https://tutorials.cosmos.network/academy/2-cosmos-concepts/5-modules.html#)

- 트랜잭션이 기본 CometBFT 합의 엔진에서 중계되면 BaseApp은 트랜잭션 내에 포함된 메시지를 분해하고 처리를 위해 메시지를 적절한 모듈로 라우팅한다.
- 적절한 모듈 메시지 핸들러가 메시지를 수신하면 해석 및 실행이 발생하고, 개발자는 Cosmos SDK를 사용하여 모듈을 구성하여 맞춤형 애플리케이션별 블록체인을 구축한다.

### Module scope

모듈에는 모든 블록체인 노드에 필요한 핵심 기능이 포함되어 있다.
- cometBFT와 통신하는 ABCI의 상용구 구현
- 모듈 상태를 유지하기 위한 범용 데이터 저장소, multistore
- 노드와 상호작용하기 위한 서버 및 인터페이스

모듈은 대부분의 어플리케이션 로직을 구현하며, KVStore(키-밸류 저장소), 어플리케이션에 필요하지만 아직 구현되지 않은 메시지 집합 등을 포함한다. 또한 모듈은 다른 모듈과의 상호작용도 가능하다.

모듈을 구성하는 요소
- Interface : 인터페이스는 모듈 간의 통신을 촉진하고 여러 모듈을 일관된 응용프로그램으로 구성
- Protobuf : 메시지를 처리하는 하나의 msg 서비스와 쿼리를 처리하는 하나의 grpc 쿼리 서비스를 제공
- Keeper : 상태를 정의, 업데이트, 검사하는 방법을 제공하는 컨트롤러

### Interfaces

모듈은 필수적으로 다른 어플리케이션들과 통합될 3가지 어플리케이션 모듈 인터페이스를 구현해야 한다.
- AppModuleBasic : 모듈의 비종속 요소들을 구현
- AppModule : 어플리케이션에 고유한 모듈의 상호 의존적이고 특수화된 요소
- AppModuleGenesis : 블록체인의 초기 상태를 설정하는 모듈의 상호 의존적인 생성/초기화 요소

### Protobuf

Msg
- 메시지 처리를 위한 RPC 메서드 집합
- 

query
- 쿼리를 위한 gRPC 쿼리 서비스