# Light client

## 개요

- 블록체인의 전체 데이터를 다운로드하지 않고도 블록 헤더의 유효성을 검증할 수 있는 경량 클라이언트
- 블록체인의 상태를 효율적으로 검증하며, 주로 Merkle Proof와 validator set의 서명을 이용하여 신뢰성을 확보

## 작동 방식

1. 블록 헤더 수신
    - 블록체인 네트워크에서 블록 헤더를 수신
    - 헤더에는 블록의 해시, 트랜잭션 루트, 상태 루트, 그리고 validator set의 서명이 있음
2. Merkle Proof을 통한 검증
    - Merkle Proof을 통해 특정 데이터가 Merkle Tree에 포함되어 있음을 증명
    - 블록체인의 상태와 트랜잭션은 Merkle Tree로 구성되며, 각 노드는 해시 값으로 이루어져 있음
3. Validator Set 서명 검증
    - validator set은 블록체인의 합의를 책임지는 노드들의 집합을 의미하며, 각 블록에 대한 서명을 통해 그 블록의 유효성을 보장
    - 수신한 블록 헤더의 상태를 검증하기 위해 validator set의 서명을 확인
    

## 참고

- [Tendermint Core - Light Client](https://docs.tendermint.com/v0.34/tendermint-core/light-client.html)
- [블록체인 라이트 클라이언트 - Medium](https://medium.com/codechain-kr/블록체인-라이트-클라이언트-505f01a40221)