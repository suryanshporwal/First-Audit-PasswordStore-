# PasswordStore — Security Findings

## Findings Overview

| ID | Severity | Finding |
|---|---|---|
| S-01 | 🔴 High | `s_password` is publicly readable despite being `private` |
| S-02 | 🔴 High | Anyone can overwrite the password via `setPassword()` |
| I-01 | 🟡 Informational | Incorrect NatSpec on `getPassword()` |

---

# [S-01] `PasswordStore::s_password` is publicly readable despite being `private`.

### Severity

**High**

### Summary

`PasswordStore` assumes that declaring `s_password` as `private` prevents external users from reading the password.

This assumption is incorrect.

> `private` only restricts **Solidity-level access**. It does not make blockchain storage private.

The password is stored in plaintext in contract storage and can therefore be read directly from the blockchain without calling `getPassword()`.

### Vulnerable Code

```solidity
string private s_password;
```

The intended access path is:

```text
                    ┌───────────────┐
Owner ─────────────►│ getPassword() │
                    └───────┬───────┘
                            │
                            ▼
                       s_password
```

However, the actual security model is:

```text
                 ┌──────────────────┐
                 │  Blockchain RPC   │
                 └────────┬─────────┘
                          │
                          ▼
                 Contract Storage
                          │
                          ▼
                    s_password
```

Anyone able to query the chain can inspect the storage directly.

### Impact

The password is **not confidential**.

An attacker does not need:

- ownership of the contract;
- access to `getPassword()`;
- Solidity-level access to `s_password`; or
- any privileged role.

The attacker only needs access to the blockchain state.

This completely defeats the core security assumption of `PasswordStore`.

### Proof of Code(PoC)

Start Anvil:

```bash
make anvil
```

Deploy the contract:

```bash
make deploy
```

The password can then be recovered by reading the relevant storage slot directly rather than calling:

```solidity
passwordStore.getPassword();
```

This proves that the `private` modifier does not provide confidentiality.

### Root Cause

The contract treats Solidity visibility as a confidentiality mechanism.

Solidity visibility modifiers are **not cryptographic privacy mechanisms**.

There is no concept of "private storage" on a normal public EVM blockchain.

### Recommendation

Do not store plaintext secrets on-chain.

If the application genuinely requires recoverable secret data, consider an architecture where:

1. The secret is encrypted before being stored.
2. The encryption key is kept off-chain and securely managed.
3. Only ciphertext or a cryptographic commitment is stored on-chain.
4. Plaintext secrets are not exposed through public contract functions.

For example:

```text
        Secret
           │
           ▼
      Encrypt off-chain
           │
           ▼
     Ciphertext
           │
           ▼
       Blockchain
```

Simply changing `private` to another Solidity visibility modifier will **not** solve this issue.

---

# [S-02] `PasswordStore::setPassword()` can be called by anyone

### Severity

**High**

### Summary

`PasswordStore::setPassword()` is intended to be callable only by the contract owner, but the function performs no access-control check.

### Vulnerable Code

```solidity
function setPassword(string memory newPassword) external {
    s_password = newPassword;

    emit SetNewPassword();
}
```

There is no validation of:

```solidity
msg.sender == s_owner
```

Therefore:

```text
Attacker
   │
   │ setPassword("attackerPassword")
   ▼
PasswordStore
   │
   ▼
s_password = "attackerPassword"
```

### Impact

Any address can overwrite the stored password.

This allows an unauthorized attacker to change the password without being the contract owner, violating the intended authorization model.

### Proof of Code(PoC)

```solidity
function test_non_owner_can_set_password(address randomAddress) public {
    vm.assume(owner != randomAddress);

    vm.prank(randomAddress);

    string memory expectedPassword = "myPassword";

    passwordStore.setPassword(expectedPassword);

    vm.prank(owner);

    string memory actualPassword = passwordStore.getPassword();

    assertEq(actualPassword, expectedPassword);
}
```

### Recommendation

Add an explicit owner check:

```solidity
function setPassword(string memory newPassword) external {
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }

    s_password = newPassword;

    emit SetNewPassword();
}
```

Or use a standard access-control implementation such as OpenZeppelin's `Ownable`:

```solidity
function setPassword(string memory newPassword) external onlyOwner {
    s_password = newPassword;

    emit SetNewPassword();
}
```

---

# [I-01] `PasswordStore::getPassword()` contains incorrect NatSpec

### Severity

**Informational**

### Summary

The NatSpec for `PasswordStore::getPassword()` documents a `newPassword` parameter even though the function accepts no parameters.

### Current Documentation

```solidity
/**
 * @notice This allows only the owner to retrieve the password.
 * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {}
```

The actual function signature is:

```solidity
function getPassword() external view returns (string memory)
```

There is no `newPassword` parameter.

### Impact

No direct security impact.

However, incorrect NatSpec can mislead developers and produce inaccurate generated documentation.

### Recommendation

Remove the invalid `@param` tag:

```diff
/**
 * @notice This allows only the owner to retrieve the password.
- * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {}
```

If `PasswordStore::getPassword()` is actually intended to be owner-only, that restriction must also be enforced in the Solidity implementation.

---

# Final Assessment

The contract has two core security problems:

### 1. Confidentiality failure

The plaintext password `PasswordStore::s_password` in file `PasswordStore.sol` is stored on-chain and can be recovered independently of Solidity's `private` visibility.

### 2. Authorization failure

Any address can overwrite the password because `PasswordStore::setPassword()` has no access control.

The NatSpec issue is informational and does not affect runtime security.

> **Key takeaway:** Solidity's `private` keyword provides access restriction at the language level — it does not make blockchain data private.