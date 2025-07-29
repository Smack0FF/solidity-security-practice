# Reentrancy Exploit — POC

## 🔍 Vulnerability Description

The `VulnerableBank` contract allows users to deposit and withdraw Ether. However, its `withdraw()` function sends Ether **before** resetting the user's balance, violating the Checks-Effects-Interactions pattern.  
This creates a **reentrancy vulnerability**, allowing attackers to recursively call `withdraw()` before their balance is updated.

---

## ⚙️ Exploit Summary

The `Attacker` contract:
1. Deposits 1 ETH into `VulnerableBank`
2. Calls `withdraw()` to trigger a refund
3. When Ether is received via `.call`, `receive()` is triggered
4. `receive()` checks if the bank still has funds and recursively calls `withdraw()` again

This continues until the entire balance of the bank is drained.

---

## 🧠 Why It Happens

In `VulnerableBank.withdraw()`:

```solidity
(bool success, ) = msg.sender.call{value: amount}("");
balances[msg.sender] = 0;
```

* The Ether is sent before the balance is cleared

* msg.sender is a contract (Attacker) with a receive() function that calls withdraw() again

* The recursive call happens while the original withdraw() is still executing, so the balance hasn’t been reset

---

## ✅ Fix Recommendation

Apply the Checks-Effects-Interactions pattern:

```solidity
function withdraw() public {
    uint amount = balances[msg.sender];
    require(amount > 0, "Nothing to withdraw");

    balances[msg.sender] = 0; // Effect first

    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}
```

Or implement a mutex lock to block reentrancy:

```solidity
bool private locked;
modifier noReentrancy() {
    require(!locked, "No reentrancy");
    locked = true;
    _;
    locked = false;
}
```

Or use .transfer() instead of .call, which automatically limits gas to 2300 and prevents complex fallback execution:
```solidity
payable(msg.sender).transfer(amount);
```
⚠️ However, .transfer() is now considered less flexible due to EIP-1884 and possible gas cost changes — so .call with proper protections is preferred in modern contracts.

---

## 📁 Files

- `VulnerableBank.sol`: the flawed contract  
- `Attacker.sol`: the exploit contract  
- `README.md`: this explanation


## 🎯 Outcome

After deploying both contracts in Remix and sending 1 ETH via `attack()`,
the attacker drains the entire balance of `VulnerableBank`.

```sql
Attacker initial deposit: 1 ETH  
Final Attacker balance: > 1 ETH  
Final Bank balance: 0
```


## 📚 References

- [Solidity Docs: Reentrancy](https://docs.soliditylang.org/en/latest/security-considerations.html#re-entrancy)
- [OpenZeppelin ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)
- [Rekt.news: reentrancy attack examples](https://rekt.news)
