## [H1] Part of the balance could be locked on the contract VotingBooth.sol

### Severity

High Risk

### Date Modified

Dec 20th, 2023

### Summary

The basic operation of the contract is to distribute the balance if the proposal passes or return the balance to the creator if it does not pass.

There is no problem if the vote fails, the entire balance is sent to the creator, but in the case of a positive proposal, the balance should be distributed until it results in a balance = 0, but this is not the case.

The problem arises because the function use different variables to calculate and distribute the rewards:

1. `totalVotes` is used to calculate the reward.
2. `totalVotesFor` is used to distribute the reward.

In that way, for each approved proposal, the amount of `rewardPerVoter x totalVotesAgainst` will be locked in the contract.

### Vulnerability Details (PoC)

We have created a contract with 9 authorized addresses, so that it is clearly visible. The test consists of executing 5 votes, 3 true + 2 false, the contract will become inactive and the balance will be distributed.

```
    function test_dos() public {
        // Voters initial balance = 0
        console2.log("initial balance creator: ", address(this).balance);
        console2.log("initial balance booth:   ", address(booth).balance);
        console2.log("initial Is active =>     ", booth.isActive());

        // 5 votes: 2 x false + 3 x true => Approved!!
        vm.prank(address(0x1));
        booth.vote(false);
        vm.prank(address(0x2));
        booth.vote(false);
        vm.prank(address(0x3));
        booth.vote(true);
        vm.prank(address(0x4));
        booth.vote(true);
        vm.prank(address(0x5));
        booth.vote(true);

        // What happened?
        console2.log("****AFTER EXECUTION***");
        console2.log("final balance creator:   ", address(this).balance);
        console2.log("final balance booth:     ", address(booth).balance);
        console2.log("final balance 0x1 :      ", address(0x1).balance);
        console2.log("final balance 0x2:       ", address(0x2).balance);
        console2.log("final balance 0x3:       ", address(0x3).balance);
        console2.log("final balance 0x4:       ", address(0x4).balance);
        console2.log("final balance 0x5:       ", address(0x5).balance);
        console2.log("final Is active =>       ", booth.isActive());

        // Expected behaivor
        assert(!booth.isActive() && address(booth).balance == 0);
    }
```

```
➜  2023-12-Voting-Booth git:(main) ✗ forge test --mt test_dos -vv
[⠢] Compiling...
[⠆] Compiling 1 files with 0.8.23
[⠰] Solc 0.8.23 finished in 1.07s
Compiler run successful!

Running 1 test for test/VotingBoothTest.t.sol:VotingBoothTest
[FAIL. Reason: panic: assertion failed (0x01)] test_dos() (gas: 359073)
Logs:
  initial balance creator:  0
  initial balance booth:    10000000000000000000
  initial Is active =>      true

  ****AFTER EXECUTION***
  final balance creator:    0
  final balance booth:      4000000000000000000
  final balance 0x1 :       0
  final balance 0x2:        0
  final balance 0x3:        2000000000000000000
  final balance 0x4:        2000000000000000000
  final balance 0x5:        2000000000000000000
  final Is active =>        false

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.23ms

Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/VotingBoothTest.t.sol:VotingBoothTest
[FAIL. Reason: panic: assertion failed (0x01)] test_dos() (gas: 359073)

Encountered a total of 1 failing tests, 0 tests succeeded
➜  2023-12-Voting-Booth git:(main) ✗

```

As expected, an amount equal to `rewardPerVoter x totalVotesAgainst` is retained in the contract.

### Impact

A significant amount of the balance may be retained in the contract with no way of being recovered.

### Tools Used

Foundry (Invariants and manual)

### Recommendations

The first recommendation is to always implement a lifeboat to avoid retained balances. In smart contracts, this lifeboat is the `withdraw()` function that should include an `onlyCreator` or similar.

Secondly, the `_distributeRewards()` function must be corrected according to the logic to be followed:

1. Distribute the entire balance among the votes = true

```
uint256 rewardPerVoter = totalRewards / totalVotesFor;
```

2. Send, after the `rewardPerVoter`distribution, the rest of the balance to the creator:

```
_sendEth(s_creator, address(this).balance);
```

3. Distribute the entire balance among the voters (true and false):

```
_sendEth(s_voters[i], rewardPerVoter);
```

It will be necessary to ask the team of developers what the idea is for the given case.

Note: The proposed solutions need further fine-tuning, just one line of code has been indicated sufficient to understand the possible solution.
