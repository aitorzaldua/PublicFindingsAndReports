## [H1] - TotalSupply() Not Implemented in Huff Code

### Severity

High Risk

### Date Modified

2024 Jan 16th

### Summary

ERC721Enumerable contains the function `tottalSupply()`, so it should be implemented correctly in the Huff version, but is not.

The developer is trying to do it, but it does not work correctly.

### Vulnerability Details (PoC)

1. First we will check the Solidity version, where works fine:

```
function test_solTotalSupplyIstWorking() external {
        vm.prank(peter);
        horseStore.mintHorse();
        assertEq(horseStore.totalSupply(), 1);
    }
```

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_solTotalSupplyIstWorking
[⠑] Compiling...
[⠑] Compiling 1 files with 0.8.20Compiler run successful!
[⠘] Compiling 1 files with 0.8.20
[⠃] Solc 0.8.20 finished in 1.07s

Running 1 test for test/HorseStoreSolidity.t.sol:HorseStoreSolidity
[PASS] test_solTotalSupplyIstWorking() (gas: 90432)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.18ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
➜  2024-01-horse-store git:(main) ✗
```

2. The Huff version is not working:

```
function test_huffTotalSupplyIsNotWorking() external {
        vm.prank(peter);
        horseStore.mintHorse();
        assertEq(horseStore.totalSupply(), 1);
    }
```

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_huffTotalSupplyIsNotWorking -vv
[⠒] Compiling...No files changed, compilation skipped
[⠢] Compiling...

Running 1 test for test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: assertion failed] test_huffTotalSupplyIsNotWorking() (gas: 73932)
Logs:
  Error: a == b not satisfied [uint]
        Left: 0
       Right: 1

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 607.36ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: assertion failed] test_huffTotalSupplyIsNotWorking() (gas: 73932)

Encountered a total of 1 failing tests, 0 tests succeeded
➜  2024-01-horse-store git:(main) ✗
```

### Impact

All functions from the Open Zeppelin libraries should work correctly, but they do not, so this is a HIGH vulnerability. Also, the `mint() ` function uses `totalSupply()`, so it is imperative that it works correctly.

### Tools Used

Foundry

### Recommendations

The function tottalSupply() must be implemented appropriately, for example:

```
#define macro NEXT_ID() = takes (0) returns (0) {
  [NEXT_ID_SLOT] sload POP_RETURN()
}

#define macro TOTAL_SUPPLY() = takes (0) returns (0) {
  NEXT_ID()
}

```
