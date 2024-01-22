## [High] Huff version doesn´t allow not EOA addresses to mint

### Severity

High Risk

### Date Modified

2024 Jan 16th

### Summary

Contrary to Solidity's version, Huff's version does not correctly implement safeMint() so it does not allow NFTs to be minted when the msg.sender is a smart contract.

### Vulnerability Details (PoC)

1. Let's create a smart contract to call the `mintHorse()` function.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

import "./IHorseStore.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

contract MinterContract {
    IHorseStore public horseStore;

    constructor(address _horseStore) {
        horseStore = IHorseStore(_horseStore);
    }

    function callingFunctions() external {
        horseStore.mintHorse();
    }

    function onERC721Received(address _msgSender, address _from, uint256 _tokenId, bytes calldata _data)
        external
        returns (bytes4 retval)
    {
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

2. We create the same test for both version:
   1. For Solidity version

```
function test_contractFeatures() external {
        minterContract.callingFunctions();
        assertEq(horseStore.totalSupply(), 1);
    }
```

If we run the test:

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_contractFeatures
[⠒] Compiling...No files changed, compilation skipped
[⠢] Compiling...

Running 1 test for test/HorseStoreSolidity.t.sol:HorseStoreSolidity
[PASS] test_contractFeatures() (gas: 94099)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 973.83µs

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

3.  For huff version (`totalSupply()` doesn´t works)

```
function test_huffContractFeatures() external {
        minterContract.callingFunctions();
    }
```

If we run the test:

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_huffContractFeatures
[⠒] Compiling...No files changed, compilation skipped
[⠢] Compiling...

Running 1 test for test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: EvmError: Revert] test_huffContractFeatures() (gas: 2567)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 660.72ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: EvmError: Revert] test_huffContractFeatures() (gas: 2567)

Encountered a total of 1 failing tests, 0 tests succeeded
➜  2024-01-horse-store git:(main) ✗
```

### Impact

This is a HIGH vulnerability because if the Solidity version allows mining to non-EOA addresses, the HUFF version must act in the same way.

### Tools Used

Foundry

### Recommendations

`safeMint` should be implemented correctly.
