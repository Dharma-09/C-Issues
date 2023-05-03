# Low Risk Issues

## L001 - Unsafe ERC20 Operation(s)

### Description

ERC20 operations can be unsafe due to different implementations and
vulnerabilities in the standard.

It is therefore recommended to always either use OpenZeppelin's `SafeERC20`
library or at least to wrap each operation in a `require` statement.

To circumvent ERC20's `approve` functions race-condition vulnerability use
OpenZeppelin's `SafeERC20` library's `safe{Increase|Decrease}Allowance`
functions.

In case the vulnerability is of no danger for your implementation, provide
enough documentation explaining the reasonings.

### Example

🤦 Bad:
```solidity
IERC20(token).transferFrom(msg.sender, address(this), amount);
```

🚀 Good (using OpenZeppelin's `SafeERC20`):
```solidity
import {SafeERC20} from "openzeppelin/token/utils/SafeERC20.sol";

// ...

IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
```

🚀 Good (using `require`):
```solidity
bool success = IERC20(token).transferFrom(msg.sender, address(this), amount);
require(success, "ERC20 transfer failed");
```

### Background Information

- [OpenZeppelin's IERC20 documentation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol#L43)


## L002 - FeeOnTransfer Tokens not Supported

### Description

There exist ERC20 tokens that charge a fee for every `transfer()` or
`transferFrom()`.

If this tokens are unsupported, ensure there is proper documentation about it.

### Example

🤦 Bad:
```solidity
IERC20(token).transferFrom(msg.sender, address(this), amount);
internalBalance += amount;
```

🚀 Good:
```solidity
uint256 balanceBefore = IERC20(token).balanceOf(address(this));
IERC20(token).transferFrom(msg.sender, address(this), amount);
uint256 balanceAfter = IERC20(token).balanceOf(address(this));
require(balanceAfter - amount == balanceBefore, "feeOnTransfer token not supported");
internalBalance += amount;
```

### Background Information

- ["The Danger of Token Integration"](https://blog.openzeppelin.com/workshop-recap-secure-development-workshop-1/)
- [RariCapital's solcurity - D8](https://github.com/Rari-Capital/solcurity#defi)


## L003 - Unspecific Compiler Version Pragma

### Description

Avoid floating pragmas for non-library contracts.

While floating pragmas make sense for libraries to allow them to be included
with multiple different versions of applications, it may be a security risk for
application implementations.

A known vulnerable compiler version may accidentally be selected or security
tools might fall-back to an older compiler version ending up checking
a different EVM compilation that is ultimately deployed on the blockchain.

It is recommended to pin to a concrete compiler version.

### Example

🤦 Bad:
```solidity
pragma solidity ^0.8.0;
```

🚀 Good:
```solidity
pragma solidity 0.8.4;
```

### Background Information

- [Consensys Audit of 1inch](https://consensys.net/diligence/audits/2020/12/1inch-liquidity-protocol/#unspecific-compiler-version-pragma)
- [Solidity docs](https://docs.soliditylang.org/en/latest/layout-of-source-files.html?highlight=pragma#version-pragma)


## L004 - Use Two-Step Transfer Pattern for Access Controls

### Description

Contracts implementing access control's, e.g. `owner`, should consider
implementing a Two-Step Transfer pattern.

Otherwise it's possible that the role mistakenly transfers ownership to the
wrong address, resulting in a loss of the role.

### Example

🤦 Bad:
```solidity
address owner;

// ...

function setOwner(address newOwner) external {
    require(msg.sender == owner, "!owner");
    emit NewOwner(newOwner);
    owner = newOwner;
}
```

🚀 Good:
```solidity
address owner;
address pendingOwner;

// ...

function setPendingOwner(address newPendingOwner) external {
    require(msg.sender == owner, "!owner");
    emit NewPendingOwner(newPendingOwner);
    pendingOwner = newPendingOwner;
}

function acceptOwnership() external {
    require(msg.sender == pendingOwner, "!pendingOwner");
    emit NewOwner(pendingOwner);
    owner = pendingOwner;
    pendingOwner = address(0);
}
```


## L005 - Do not use Deprecated Library Functions

### Description

The usage of deprecated library functions should be discouraged.

This issue is mostly related to OpenZeppelin libraries.

### Example

🤦 Bad:
```solidity
use SafeERC20 for IERC20;

// ...

IERC20(token).safeApprove(spender, value);
```

🚀 Good:
```solidity
use SafeERC20 for IERC20;

// ...

IERC20(token).safeIncreaseAllowance(spender, value);
```


## L006 - Check that Contract Exists before using `solmate`'s `SafeTransferLib`

### Description

Functions in `solmate`'s `SafeTransferLib` library do not check whether a token
has code at all. This responsibility is delegated to the caller.

As a call to an address with no code will be a no-op, since low-level calls to
non-contracts always return true, a transfer of tokens using `solmate`'s
`SafeTransferLib` will succeed if the token does not have any code.

Therefore, it is recommended to verify that a contract exists before using any
`SafeTransferLib` functions.

### Example

🤦 Bad:
```solidity
// Note that this function is for demonstration purposes only and should not be used as is.
function _fetchTokens(address token, address from, uint amount) internal {
    ERC20(token).safeTransferFrom(from, amount);
}
```

🚀 Good:
```solidity
// Note that this function is for demonstration purposes only and should not be used as is.
function _fetchTokens(address token, address from, uint amount) internal {
    require(token.code.length != 0, "Token does not exist");
    ERC20(token).safeTransferFrom(from, amount);
}
```

### Background Information

- [Comment in `solmate`'s `SafeTransferLib`](https://github.com/Rari-Capital/solmate/blob/main/src/utils/SafeTransferLib.sol#L9)
- [First Issue in QA Report for backed-protocol](https://github.com/code-423n4/2022-04-backed-findings/issues/134)
