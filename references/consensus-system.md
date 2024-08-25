# Consensus Systems with Ethan Buchman

2018년 Ethan Buchman이 Tendermint와 Cosmos 프로젝트에 관해 팟캐스트로 진행한 내용을 ChatGPT를 이용하여 간략하게 요약한다.

> 원문 : [Consensus Systems with Ethan Buchman](https://softwareengineeringdaily.com/2018/03/26/consensus-systems-with-ethan-buchman/)

## 1. 합의 프로토콜의 개념

- 합의 프로토콜(Consensus Protocol)은 분산된 시스템에서 여러 노드가 시스템 상태에 대해 합의할 수 있도록 하는 메커니즘입니다.
- Tendermint는 이러한 합의 프로토콜 중에서도 비잔틴 장애 허용(Byzantine Fault Tolerance, BFT)을 지원하는 알고리즘으로, 네트워크 상의 일부 노드가 악의적으로 행동하거나 잘못된 데이터를 전달해도 시스템의 안전성과 일관성을 유지할 수 있습니다.
- Tendermint의 BFT 합의 알고리즘은 최대 1/3의 노드가 비정상적이더라도 나머지 노드가 합의에 도달할 수 있도록 설계되었습니다.

<br>

## 2. Tendermint 기술적 구조

- 상태 머신 복제(State Machine Replication): Tendermint는 상태 머신 복제 엔진입니다. 즉, 어떤 애플리케이션이든 결정론적이고 유한 상태 머신(Deterministic Finite State Machine)으로 구성되면, 이를 분산된 여러 노드에서 일관성 있게 실행할 수 있습니다.
- ABCI (Application BlockChain Interface): Tendermint는 애플리케이션과의 상호작용을 위해 ABCI라는 인터페이스를 제공합니다. 이를 통해 개발자는 어떤 프로그래밍 언어로도 애플리케이션을 개발할 수 있으며, Tendermint가 합의 프로토콜을 처리하고 애플리케이션에 합의된 트랜잭션을 전달합니다. 예를 들어, Python, Go, Haskell 등 다양한 언어로 애플리케이션을 개발하고 Tendermint에 연결할 수 있습니다.
- Tendermint는 애플리케이션과 독립적으로 동작하며, 트랜잭션을 전달하고 상태를 업데이트하는 책임을 애플리케이션에 맡깁니다.

<br>

## 3. Cosmos와 Tendermint의 연계

- Cosmos Hub는 여러 블록체인 간 상호 운용성(Interoperability)을 지원하는 구조로, 각각의 블록체인(‘Zone’)이 Cosmos Hub를 통해 통신합니다.
- 각 블록체인이 서로를 직접 신뢰하는 대신, 중앙 허브(예: Cosmos Hub)를 통해 안전하게 상호작용할 수 있습니다. 예를 들어, 한 블록체인에서 다른 블록체인으로 자산을 전송할 때 IBC를 사용해 안전한 자산 이동이 가능합니다.
- IBC(Inter-Blockchain Communication): Cosmos에서 블록체인 간 상호작용은 IBC 프로토콜을 통해 이루어지며, 이는 Tendermint의 라이트 클라이언트(light client) 기능을 기반으로 구현되었습니다.

<br>

## 4. 기존 블록체인과의 차별점

- Ethereum과의 차별점: Tendermint는 EVM(Ethereum Virtual Machine)과 달리, 애플리케이션을 특정 가상 머신에 종속시키지 않습니다. 이더리움에서는 Solidity 등의 언어로 작성된 코드가 EVM 바이트코드로 변환되어 실행되는데, Tendermint는 이러한 제한 없이 다양한 프로그래밍 언어로 애플리케이션을 개발할 수 있게 합니다.
- 이더리움의 확장성 문제(예: Cryptokitties와 같은 애플리케이션이 네트워크에 큰 부담을 주는 상황)를 해결하기 위해, Ethermint라는 프로젝트를 통해 이더리움의 상태 머신을 Tendermint 합의 엔진 위에서 실행할 수 있도록 하고 있습니다. 이를 통해 이더리움의 확장성을 크게 개선할 수 있습니다.

<br>

## 5. 기술적 트레이드오프
- 비잔틴 장애 허용(BFT) vs 비잔틴 비허용(Non-BFT): Tendermint는 BFT 합의를 지원하는 반면, Raft나 Paxos와 같은 기존의 합의 알고리즘은 비잔틴 환경에서는 취약합니다. 즉, Paxos나 Raft는 악의적인 노드가 있는 경우 시스템이 무너질 수 있지만, Tendermint는 악의적인 노드가 일부 존재해도 안전하게 동작할 수 있습니다.
- 동기 네트워크(Synchronous Network)와 비동기 네트워크(Asynchronous Network): Tendermint는 비동기 네트워크에서도 안전성을 보장할 수 있지만, 네트워크가 완전히 분리되면 진행이 멈출 수 있습니다. 반면, 비트코인은 완전한 동기 네트워크를 가정하며, 네트워크가 분리될 경우 나중에 재조정될 가능성이 있습니다. 이는 각 시스템의 안전성(safety)과 생존성(liveness) 간의 트레이드오프에 해당합니다.

<br>

## 6. 사용 사례
- 외환 거래소(Forex Exchange): 유럽의 한 외환 거래소는 매일 약 5억 달러의 외환 거래를 Tendermint 위에서 처리하고 있습니다. 이 시스템은 분산된 환경에서도 일관된 거래 기록을 유지하며, 비잔틴 장애 허용 덕분에 악의적인 공격에 대비할 수 있습니다.
- 다양한 애플리케이션: 에너지 거래 시스템, 탈중앙화 거래소(DEX), 예측 시장 등 다양한 애플리케이션이 Tendermint와 Cosmos를 사용해 개발되고 있습니다.

<br>

## 7. 향후 발전 방향
- Cosmos 네트워크는 다양한 블록체인 간 상호 운용성을 강화하기 위해 지속적으로 발전 중이며, Ethereum과의 연계나 Plasma와 같은 Layer 2 솔루션과의 결합도 고려하고 있습니다.
