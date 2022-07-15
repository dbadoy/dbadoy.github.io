---
layout: post
title:  "Dixel club community audit"
tags: audit solidity
---

Dixel Club 에서 진행했던 smart contract community audit에 참가하여 활동한 내역을 나중에 보기 쉽게 정리 해보려 한다.

진행 기간 : 6월 2일 ~ 6월 12일 

## 1) PR - scripts/deploy: add verifying contract logic at once.
[URL](https://github.com/Steemhunt/dixel-v2-contract/pull/18)

해당 프로젝트에서는 smart contract 배포를 hardhat으로 진행하고 있었다.
배포 코드 중,
```typescript
  console.log(`
    npx hardhat verify --network ${hre.network.name} ${factory.address}
    npx hardhat verify --network ${hre.network.name} ${nftImplementation}
  `);
```
모든 배포가 끝난 후, verify를 하기 위하여 배포된 contract address가 포함된 커맨드를 로깅 해주고 있다.
이 코드를 보고, 이전에 hardhat을 이용하여 smart contract 테스트 스크립트를 작성하며 알 수 있었던, npx hardhat 커맨드를 코드 레벨에서 실행 시킬 수 있는 방법이 떠올랐다.
```typescript
import hre from "hardhat";

hre.run("COMMAND", {PARAMETER_NAME: VALUE}
```

따라서, verify를 배포와 동시에 진행할 수 있도록 아래와 같이 수정하여 PR을 전송했다.
```typescript
 hre.run('verify', {address: factory.address});
 hre.run('verify', {address: nftImplementation});
```

![](https://velog.velcdn.com/images/dbadoy/post/2cf6d398-0044-4a74-b1e6-1e1be0c7c9dd/image.png)
이렇게 첫번째 PR이 merge 되었다.
그러나...
![](https://velog.velcdn.com/images/dbadoy/post/ae1b5775-2d38-4ad7-ae07-319628afcb0b/image.png)
revert 됨...
hardhat verify는 이더스캔에서 contract address의 바이트코드를 검증하는 과정인데, 배포된 후 이더스캔에 바로 추가가 되는 것이 아닌, 몇 분의 시간이 걸린다고 한다. 그래서 내가 추가한 verify는 무조건 실패가 뜨게 된 것이다.

## 2) Issue - Split updateBeneficiary() in DixelClubV2Factory.sol

[URL](https://github.com/Steemhunt/dixel-v2-contract/issues/20)

smart contract를 보다가,
```typescript
function updateBeneficiary(address newAddress, uint256 newCreationFee, uint256 newMintingFee) external onlyOwner {
  if(newAddress == address(0)) revert DixelClubV2Factory__ZeroAddress();
  if(newMintingFee > FRICTION_BASE) revert DixelClubV2Factory__InvalidFee();
  
  beneficiary = newAddress;
  mintingFee = newMintingFee;
  creationFee = newCreationFee;
}
```
이와 같이 contract logic에 영향을 미치는 변수를 변경하는 메서드가 있었다.
그러나, beneficiary, mintingFee, creationFee 세 값을 한번에 변경할 때는 편하겠지만, 이 중 하나만 바꾸는 경우라면, 그 외의 값을을 contract에서 가져와 입력해주어야 하는 과정이 필요하고, 이는 휴먼 에러가 발생할 가능성과 불필요한 작업을 하게 될 것으로 보였다.
따라서,
```typescript
function updateBeneficiary(address newAddress) external onlyOwner {
  if(newAddress == address(0)) revert DixelClubV2Factory__ZeroAddress();
  beneficiary = newAddress;
}
    
function updateMintingFee(uint256 newMintingFee) external onlyOwner {
  if(newMintingFee > FRICTION_BASE) revert DixelClubV2Factory__InvalidFee();
  mintingFee = newMintingFee;
}
    
function updateCreationFee(uint256 newCreationFee) external onlyOwner {
  creationFee = newCreationFee;
}
```
<br>
만약 beneficiary, mintingFee, creationFee 세가지 변수가 서로 연관이 있어 한 번에 수정되는 경우가 대다수라면 기존 코드가 맞겠지만, 그 문제에 대해서는 확신이 없어 issue를 생성했다.
<br>
그 결과 ...
![](https://velog.velcdn.com/images/dbadoy/post/e5c4bf59-1caa-4a97-918e-c1cf8cb0f768/image.png)
해당 내용이 적용되었다.

## 3) PR - all: typo - remove blank.
[URL](https://github.com/Steemhunt/dixel-v2-contract/pull/22)

단순히 if문 안에 불필요한 공백을 처리한 PR 이다. 

결과 : Merged

## 4) Issue - Optimize gas cost in for loop.
[URL](https://github.com/Steemhunt/dixel-v2-contract/issues/23)

해당 audit에서 가장 재밌고 흥미로웠던 부분은 for문에서의 gas cost optimize 부분이었다.
처음에는 다른 분이 commit 하신 코드를 보며 몰랐던 사실을 알게되어 재밌었다.
```typescript
function getWhitelistAllowanceLeft(address wallet) external view returns (uint256 allowance) {
  uint256 length = _whitelist.length; // gas saving
  for (uint256 i; i != length;) {
      if (_whitelist[i] == wallet) {
          allowance++;
      }
      unchecked {
          ++i;
      }
  }

  return allowance;
}
```
위의 코드에서 배운 두 가지를 적어보려 한다.
### 1 - Unchecked
 Solidity 0.8 버전부터는 산술 연산에서 overflow를 체크해주기 때문에 SafeMath 라이브러리 사용 필요성이 없어졌다. 하지만, 체크가 필요없는 경우에 overflow 체크는 오버헤드를 발생시키기 때문에, 이런 경우에는 unchecked 키워드를 사용하면 gas를 줄일 수 있다.
unchecked는 해당 스코프안 산술 연산에 대한 overflow 체크를 하지 않는 키워드다.
이와 관련해서 직접 테스트를 하여 gas를 비교해보았다.
```typescript
function CheckedSumCase(uint loopCount) external {
	uint temp = 0;
	for (uint i = 0; i < loopCount; i ++) {
		temp += i;
	}
	result = temp;
}

function UncheckedSumCase(uint loopCount) external {
	uint temp = 0;
	for (uint i = 0; i < loopCount;) {
		temp += i;
		unchecked {
			i++;
		}
	}
	result = temp;
}
```
```
loopCount = 10
CheckedSumCase gasUsed: 43689
UncheckedSumCase gasUsed: 42927

loopCount = 100
CheckedSumCase gasUsed: 60969
UncheckedSumCase gasUsed: 53547

loopCount = 1000
CheckedSumCase gasUsed: 233833
UncheckedSumCase gasUsed: 159811
```
꽤 차이가 난다.
위 코드에서도 보면, 이와 같이 for문에 i 증감 연산을 unchecked에 포함시켜 gas optimize를 했다.

### 2 - for문 length
```typescript
uint[] data;

for (uint i; i < data.length; i++) {
 	// do something 
}
```
일반적으로 사용하는 for문은 위와 같이 사용한다.
그런데 내가 본 코드에서는 아래와 같이 사용하고 있다.
```typescript
uint[] data;

uint len = data.length;
for (uint i; i != len;) {
 	// do something 
}
```
왜일까?
테스트를 해보니 놀랍게도 for문을 돌 때마다, data를 SLOAD 하여 length를 구하고 있다.
data의 length가 100이라면, for문이 100번 돌아가며 SLOAD 또한 100번을 하는 것이다...
SLOAD opcode gas cost는 200으로, 100번의 경우 20000 gas, 1000번의 경우 200000 gas ...
크기가 늘어날 수록 gas도 비례하여 늘어날 것이다.
이 말은, 이 부분에 관련해서 optimize를 하면 그만큼 gas를 많이 절약할 수 있다는 것이다.

** + SLOAD 한다는 것 증명했던 방법 **
uint[] data 
data에 100개의 숫자를 넣고 for문을 돌렸을 때와 90개의 숫자를 넣고 돌렸을 때, gasUsed 가 2000 차이 난다. (SLOAD 200 * 10 = 2000)

이와 같은 맥락으로 생각해보았을 때, for문 안에서 어떤 storage 데이터에 계속해서 접근하는 것 보다, 한번 가져오고 for문에서는 가져온 값에 접근을 하는 것은 어떨까? 라는 생각이 들었다.
먼저 테스트를 해보았다.
```typescript
address[20] addrs = [ ... ]

function origin() external {
    uint256 dummy = 0;
    uint256 len = addrs.length;
    for (uint256 i; i < len;) {
        if (addrs[i] == address(0)) {
            dummy += i;
        }
        unchecked {
            i++;
        }
    }
}

function suggest1() external {
    uint256 dummy = 0;
    address[] memory temp = addrs;
    uint256 len = temp.length;
    for (uint256 i; i < len;) {
        if (temp[i] == address(0)) {
            dummy += i;
        }
        unchecked {
            i++;
        }
    }
}

function suggest2() external {
    uint256 dummy = 0;
    address[] storage temp = addrs;
    uint256 len = temp.length;
    for (uint256 i; i < len;) {
        if (temp[i] == address(0)) {
            dummy += i;
        }
        unchecked {
            i++;
        }
    }
}
```
위와 같이, length가 20인 address array를 만들어 준 후, 
origin()에서는 기존과 같이 storage data에 매번 접근하는 경우.
suggest1()에서는 for문 시작전 memory 변수에 값을 가져오는 경우.
suggest2()에서는 storage 변수에 값을 가져오는 경우. (사실상 의미 없다.)
이렇게 세가지로 나누어 테스트를 진행했다.
```
origin() gasUsed : 57873
suggest1() gasUsed : 43636
suggest2() gasUsed : 57925
```
57873 - 43636 = 14237

length가 20인 address array의 경우에 14237의 gas를 절약할 수 있었으며, array의 크기가 커질 수록, 해당 optimize의 효과는 커질 것으로 보인다.
이는 유의미하다고 생각되어 Issue를 생성했다.

![](https://velog.velcdn.com/images/dbadoy/post/eaf012a3-27a1-491a-8af6-3182c841f685/image.png)

작업 후, 코드에 반영되었다.

## 5) PR - contracts: early check that the fee is correct.
[URL](https://github.com/Steemhunt/dixel-v2-contract/pull/26)

```typescript
if(bytes(name).length == 0) revert DixelClubV2Factory__BlankedName();
if(bytes(symbol).length == 0) revert DixelClubV2Factory__BlankedSymbol();
if(bytes(description).length > 1000) revert DixelClubV2Factory__DescriptionTooLong(); // ~900 gas per character
if(metaData.maxSupply == 0 || metaData.maxSupply > MAX_SUPPLY) revert DixelClubV2Factory__InvalidMaxSupply();
if(metaData.royaltyFriction > MAX_ROYALTY_FRACTION) revert DixelClubV2Factory__InvalidRoyalty();

// Validate `symbol`, `name` and `description` to ensure generateJSON() creates a valid JSON
if(!StringUtils.validJSONValue(name)) revert DixelClubV2Factory__NameContainedMalicious();
if(!StringUtils.validJSONValue(symbol)) revert DixelClubV2Factory__SymbolContainedMalicious();
if(!StringUtils.validJSONValue(description)) revert DixelClubV2Factory__DescriptionContainedMalicious();

// Neutralize minting starts date
if (metaData.mintingBeginsFrom < block.timestamp) {
    metaData.mintingBeginsFrom = uint40(block.timestamp);
}

if (creationFee > 0) {
    if(msg.value != creationFee) revert DixelClubV2Factory__InvalidCreationFee();

    // Send fee to the beneficiary
    (bool sent, ) = beneficiary.call{ value: creationFee }("");
    require(sent, "CREATION_FEE_TRANSFER_FAILED");
}
```
위 코드는 사용자가 파라미터들을 체크해준 후, creationFee가 0보다 크게 설정되어 있을 경우 beneficiary address에 transfer 하는, 실제 수행할 로직 전에 체크해주는 코드의 일부이다.
먼저 StringUtils.validJSONValue 메서드를 살펴보자.

```typescript
function validJSONValue(string calldata haystack) internal pure returns (bool) {
    bytes memory haystackBytes = bytes(haystack);
    uint256 length = haystackBytes.length;
    for (uint256 i; i != length;) {
        bytes1 char = haystackBytes[i];
        if (char < 0x20 || char == 0x22 || char == 0x5c || char == 0x7f) {
            return false;
        }

        unchecked {
            ++i;
        }
    }

    return true;
}
```
안에 확인하는 값은 제쳐두고, 해당 메서드는 주어진 파라미터의 크기만큼 순회하고 있다는 것을 볼 수 있다. 위 코드에서 description 같은 경우는 length가 1000보다 작을 경우 에러를 리턴하는 것으로 부터 꽤 큰 길이의 파라미터라는 것을 유추할 수 있고, description이 validJSONValue() 메서드로 넘어가면 가볍지만은 않을 것이다.
필요한 확인 작업이라면 리소스가 전혀 아깝지 않지만, validJSONValue() 메서드 확인이 끝난 후 creationFee를 체크하는 과정에서, fee가 부족하여 revert가 발생하면 위에 수행한 작업들이 헛수고가 된다.
따라서 creationFee를 먼저 체크하고 통과하는 경우에 validJSONValue()를 수행하는 것이 바람직하다고 생각이 들어 PR을 만들었다.

(수정 코드)
```typescript
if (creationFee > 0) {
    if(msg.value != creationFee) revert DixelClubV2Factory__InvalidCreationFee();

    // Send fee to the beneficiary
    (bool sent, ) = beneficiary.call{ value: creationFee }("");
    require(sent, "CREATION_FEE_TRANSFER_FAILED");
}

// Validate `symbol`, `name` and `description` to ensure generateJSON() creates a valid JSON
if(!StringUtils.validJSONValue(name)) revert DixelClubV2Factory__NameContainedMalicious();
if(!StringUtils.validJSONValue(symbol)) revert DixelClubV2Factory__SymbolContainedMalicious();
```
<br>
(재수정 코드)<br>
validJSONValue 또한, 충족되어야 하는 검증과정이기 때문에 성공적으로 검증이 끝난 후 fee를 보내는 것이 코드를 읽을 때 좋을 것 같아 아래와 같이 수정했다.

```typescript
if(msg.value != creationFee) revert DixelClubV2Factory__InvalidCreationFee();
// Validate `symbol`, `name` and `description` to ensure generateJSON() creates a valid JSON
if(!StringUtils.validJSONValue(name)) revert DixelClubV2Factory__NameContainedMalicious();
if(!StringUtils.validJSONValue(symbol)) revert DixelClubV2Factory__SymbolContainedMalicious();

if (creationFee > 0) {
    // Send fee to the beneficiary
    (bool sent, ) = beneficiary.call{ value: creationFee }("");
    require(sent, "CREATION_FEE_TRANSFER_FAILED");
}

```

결과
![](https://velog.velcdn.com/images/dbadoy/post/9f43fade-c394-4f07-a669-19222404eb5b/image.png)

Merge 되었다.

## 마치며)
사실, 어떻게 하면 더 나을까 혼자 고민하며 얻은 것 보다는 다른 사람들이 만든 Issue와 PR을 보며 얻는 것이 더 많았었다. 오픈 소스에 기여도 물론 좋지만, 기본 지식을 더욱 단단히 다지는 것이 중요하다는 것을 깨달을 수 있었던 경험이었다. 즐거웠다.

