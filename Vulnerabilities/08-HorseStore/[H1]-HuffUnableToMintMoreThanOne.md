## [H1] - Unable To Mint More Than One in Huff Code

### Severity

High Risk

### Date Modified

2024 Jan 16th

### Summary

The huff version is unable to mint more than one "horse" due to a bad implementation.

### Vulnerability Details (PoC)

This is easy to prove with a simple test:

```
function test_huffSingleMint() external {
        vm.startPrank(peter);
        horseStore.mintHorse();
        horseStore.mintHorse();
        vm.stopPrank();
        assertEq(horseStore.balanceOf(peter), 2);
    }
```

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_huffSingleMint
[⠢] Compiling...
[⠔] Compiling 1 files with 0.8.20Compiler run successful!
[⠒] Compiling 1 files with 0.8.20
[⠑] Solc 0.8.20 finished in 1.39s

Running 1 test for test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: revert: ALREADY_MINTED] test_huffSingleMint() (gas: 59935)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 526.54ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: revert: ALREADY_MINTED] test_huffSingleMint() (gas: 59935)

Encountered a total of 1 failing tests, 0 tests succeeded
➜  2024-01-horse-store git:(main) ✗
```

### Impact

### Tools Used

### Recommendations

The protocol need to add a `nextId` variable that increase wich each mint, so the protocol and `mint` function has a new Id each operation.

Something similar to:

```
#define constant NEXT_ID_SLOT = FREE_STORAGE_POINTER()
#define constant MINT_COUNT_SLOT = FREE_STORAGE_POINTER()

....

// increment balance
    [BALANCES_SLOT] caller STORAGE_HASH()
    dup1 // [balancesHash, balancesHash]
    sload // [currentBalance, balancesHash]
    0x1 add // [newBalance, balancesHash]
    swap1 // [balancesHash, newBalance]
    sstore // []

    // increment nextId
    0x1 add // [newNextId, nextId]
    [NEXT_ID_SLOT] // [nextIdSlot, newNextId, nextId]
    sstore // [nextId]

```
