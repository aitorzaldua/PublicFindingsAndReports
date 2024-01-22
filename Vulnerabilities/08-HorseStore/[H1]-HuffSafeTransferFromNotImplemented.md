## [H1] - SafeTransferFrom() Not Implemented in Huff Code

### Severity

High Risk

### Date Modified

2024 Jan 16th

### Summary

SafeTransferFrom() works perfectly well in the Solidity code (as it correctly imports the ERC721 Open Zeppelin contract), but in the Huff version the safeTransferFrom() function is not defined. This leads us to a high severity vulnerability:

1. The Huff version was supposed to implement 100% of the Solidity version.
2. The safeTransferFrom function guarantees that the recipient is able to receive the NFT and with this implementation this is not guaranteed if the recipient is a contract.

### Vulnerability Details

As we can see, in the ERC721 definitions:

```
#define function transfer(address,uint256) nonpayable returns ()
#define function transferFrom(address,address,uint256) nonpayable returns ()
```

The definition `transfer(address,uint256)` should be removed, as has no functionality and safeTransferFrom() should be added:

```
#define function safeTransferFrom(address,address,uint256) nonpayable returns ()
```

Check how correctly works in the Solidity version:

```
function test_approveTransferSafeTransfer() external {
        vm.startPrank(peter);
        horseStore.mintHorse();
        horseStore.approve(maryJane, 0);
        vm.stopPrank();
        vm.startPrank(maryJane);
        horseStore.transferFrom(peter, maryJane, 0);
        assertEq(horseStore.ownerOf(0), maryJane);
        horseStore.safeTransferFrom(maryJane, norman, 0, "");
        vm.stopPrank();
        assertEq(horseStore.ownerOf(0), norman);
    }
```

```
➜  2024-01-horse-store git:(main) ✗ forge test --mt test_approveTransferSafeTransfer
[⠒] Compiling...No files changed, compilation skipped
[⠢] Compiling...

Running 1 test for test/HorseStoreSolidity.t.sol:HorseStoreSolidity
[PASS] test_approveTransferSafeTransfer() (gas: 146529)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.72ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
➜  2024-01-horse-store git:(main) ✗
```

But if we move the function to Huff test:

```
function test_huffApproveTransferSafeTransfer() external {
        vm.startPrank(peter);
        horseStore.mintHorse();
        horseStore.approve(maryJane, 0);
        vm.stopPrank();
        vm.startPrank(maryJane);
        horseStore.transferFrom(peter, maryJane, 0);
        assertEq(horseStore.ownerOf(0), maryJane);
        horseStore.safeTransferFrom(maryJane, norman, 0, "");
        vm.stopPrank();
        assertEq(horseStore.ownerOf(0), norman);
    }

```

```
  ├─ [2501] 0x6d2eed85750d316088343D6d5e91ca59eb052768::safeTransferFrom(0x0000000000000000000000000000000000000003, 0x0000000000000000000000000000000000000004, 0, 0x)
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [594] 0x6d2eed85750d316088343D6d5e91ca59eb052768::ownerOf(0) [staticcall]
    │   └─ ← 0x0000000000000000000000000000000000000003
    ├─ emit log(val: "Error: a == b not satisfied [address]")
    ├─ emit log_named_address(key: "      Left", val: 0x0000000000000000000000000000000000000003)
    ├─ emit log_named_address(key: "     Right", val: 0x0000000000000000000000000000000000000004)
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    └─ ← ()

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 645.98ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/HorseStoreHuff.t.sol:HorseStoreHuff
[FAIL. Reason: assertion failed] test_huffApproveTransferSafeTransfer() (gas: 101578)
```

### Impact

This is a HIGH vulnerability because the protocol is unable to guarantee that an NFT is correctly delivered to the receiver.

### Tools Used

Foundry

### Recommendations

The function safeTransferFrom() should be implemented in the Huff version. As an example:

```
#define function safeTransferFrom(address,address,uint256) nonpayable returns ()

(continue ....)

#define macro SAFE_TRANSFER_FROM() = takes (0) returns (0) {
  // check onERC721Received
  0x24 calldataload // [to]
  extcodesize 0x0 eq transferIsSafe jumpi // []
  [ON_ERC721_RECEIVED_SIG] 0x0 mstore
  0x04 calldataload 0x04 mstore
  caller 0x24 mstore
  0x24 calldataload 0x44 mstore
  0x80 0x64 mstore
  0x20 0x00 0xa4 0x0 0x0 0x24 calldataload gas call // [success]
  0x0 eq invalidReceiver jumpi
  0x0 mload // [result]
  [ON_ERC721_RECEIVED_SIG] eq iszero invalidReceiver jumpi

  transferIsSafe:
  TRANSFER_FROM()

  invalidReceiver:
  __ERROR(ERC721InvalidReceiver) 0x0 mstore
  0x24 calldataload 0x04 mstore
  0x24 0x0 revert
}


```
