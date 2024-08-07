# Merkle proof

## Merkle trees

```go
type SimpleProof struct {
    Total    int
    Index    int
    LeafHash []byte
    Aunts    [][]byte
}

func (proof SimpleProof) Verify(rootHash []byte, leaf []byte) bool {
    assert(proof.LeafHash, leafHash(leaf))

    computedHash := computeHashFromAunts(proof.Index, proof.Total, proof.LeafHash, proof.Aunts)
    return computedHash == rootHash
}

func computeHashFromAunts(index, total int, leafHash []byte, innerHashes [][]byte) []byte {
    assert(index < total && index >= 0 && total > 0)

    if total == 1 {
        assert(len(proof.Aunts) == 0)
        return leafHash
    }

    assert(len(innerHashes) > 0)

    numLeft := getSplitPoint(total) // total보다 작은 2의 최대 제곱수
    if index < numLeft {
        leftHash := computeHashFromAunts(index, numLeft, leafHash, innerHashes[:len(innerHashes)-1])
        assert(leftHash != nil)
        return innerHash(leftHash, innerHashes[len(innerHashes)-1])
    }
    rightHash := computeHashFromAunts(index-numLeft, total-numLeft, leafHash, innerHashes[:len(innerHashes)-1])
    assert(rightHash != nil)
    return innerHash(innerHashes[len(innerHashes)-1], rightHash)
}
```

SimpleProof 구조체

- Total: 트리의 총 노드 수
- Index: 리프 노드의 인덱스
- LeafHash: 리프 노드의 해시 값
- Aunts: 증명을 위한 해시 값들(리프 노드를 제외한 해시 값들)

> Aunts 노드란?
> 리프 노드의 해시를 루트 해시로 변환할 때 필요한 해시 값들입니다. Merkle 트리에서 리프 노드가 아닌 노드들의 해시 값을 포함합니다. 이는 리프 노드로부터 상위 노드까지의 경로에서 필요한 형제 노드의 해시 값을 의미

진행 방식
1. 블록의 해시값 확인: 블록의 해시값을 통해 블록 헤더의 Merkle 루트 해시를 가져옵니다.
2. 루트 해시 확인: 블록 해시에서 추출한 루트 해시를 기준으로 합니다.
3. Merkle Proof 검증: SimpleProof의 Verify 함수를 사용해 리프 해시를 검증합니다. computeHashFromAunts 함수는 재귀적으로 Merkle Tree의 해시값을 계산합니다.

Leaf 노드 추가 시기
- Merkle Tree의 리프 노드는 처음부터 정해져 있는 것이 아니라, 데이터가 추가됨에 따라 동적으로 생성됩니다. 블록체인에서는 새로운 트랜잭션이 발생할 때마다 리프 노드가 추가됩니다.
    1. 트랜잭션 발생 시: 새로운 트랜잭션이 블록체인에 추가될 때마다 해당 트랜잭션을 포함하는 리프 노드가 Merkle Tree에 추가됩니다.
	2. 블록 생성 시: 일정 주기마다 새로운 블록이 생성될 때, 해당 블록에 포함된 모든 트랜잭션의 리프 노드가 Merkle Tree에 추가됩니다.

Merkle Tree의 동적 구성
- 동적 추가: 새로운 트랜잭션이 발생하면 이를 포함하는 리프 노드가 추가되고, Merkle Tree는 동적으로 재구성됩니다.
- 최소한의 재계산: 새로운 리프 노드를 추가할 때, 기존 트리 구조를 최대한 유지하며 필요한 부분만 재계산합니다.

```go
// example
package main

import (
	"crypto/sha256"
	"fmt"
)

// tmhash.Sum equivalent function using SHA256
func tmhashSum(data []byte) []byte {
	hash := sha256.Sum256(data)
	return hash[:]
}

// SHA256(0x00 || leaf)
func leafHash(leaf []byte) []byte {
	return tmhashSum(append([]byte{0x00}, leaf...))
}

// SHA256(0x01 || left || right)
func innerHash(left []byte, right []byte) []byte {
	return tmhashSum(append([]byte{0x01}, append(left, right...)...))
}

// largest power of 2 less than k
func getSplitPoint(k int) int {
	if k < 1 {
		panic("k should be greater than 0")
	}
	p := 1
	for p*2 < k {
		p *= 2
	}
	return p
}

func MerkleRoot(items [][]byte) []byte {
	switch len(items) {
	case 0:
		return nil
	case 1:
		return leafHash(items[0])
	default:
		k := getSplitPoint(len(items))
		left := MerkleRoot(items[:k])
		right := MerkleRoot(items[k:])
		return innerHash(left, right)
	}
}

func main() {
	items := [][]byte{
		[]byte("L0"),
		[]byte("L1"),
		[]byte("L2"),
		[]byte("L3"),
		[]byte("L4"),
	}

	root := MerkleRoot(items)
	fmt.Printf("Merkle Root: %x\n", root)
}
```