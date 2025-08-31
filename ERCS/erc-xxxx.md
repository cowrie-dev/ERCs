# ERC-XXXX: PayoutRace Standard

## Abstract

This ERC defines a minimal standard for contracts that sell discrete **assets** from a single bucket in exchange for a fixed **paymentToken**. Each asset is purchased independently for the same global price. Fulfillment of assets is open-ended and delegated to external **handlers**, allowing assets to be ERC-20 transfers, ERC-777 sends, ETH payouts, Uniswap V3 fee claims, or any other callable logic. This provides a generic, composable mechanism for protocols to externalize price discovery and reward distribution, while retaining flexibility in how "assets" are realized.

---

## Motivation

The Uniswap Foundation’s UniStaker project describes a "payout race" mechanism in which a claimer pays a fixed amount of UNI to collect Uniswap V3 pool fees for distribution to UNI stakers. This ERC generalizes that pattern:

* There is a single bucket of assets, with a single contract entrypoint for discovery and execution of purchases.
* A global paymentToken and paymentAmount are defined by the owner.
* Searchers can purchase specific assets, one at a time.
* Fulfillment is pluggable via handlers, so different asset types slot into the same standard.

This enables:

* Reusable infrastructure across protocols.
* Open-ended fulfillment (e.g. calling a Uniswap V3 pool’s `claimFees`, or transferring ERC-20s).
* Simple, race-based UX: “pay X units of paymentToken, atomically get asset Y.”

---

## Specification

### Core Interface

```solidity
interface IPayoutRace /* is IERC165 */ {
    // Global payment configuration (ERC-20 only)
    function paymentToken() external view returns (address);
    function paymentAmount() external view returns (uint256);
    function paymentReceiver() external view returns (address);

    // Asset discovery
    function assetExists(bytes32 assetId) external view returns (bool);
    function availableAssetCount() external view returns (uint256);
    function assetIdAt(uint256 index) external view returns (bytes32);

    /// @notice Purchase a listed asset for the fixed global price.
    /// @param expectedPaymentAmount equality guard against config churn
    /// @dev MUST revert if the asset is not currently listed. MUST update state before any external calls.
    /// @dev Implementations MUST transfer exactly paymentAmount() of paymentToken() from msg.sender to paymentReceiver().
    function purchase(
        bytes32 assetId,
        address recipient,
        uint256 expectedPaymentAmount
    ) external returns (uint256 paid);

    // Accounting
    function totalPaid() external view returns (uint256);

    // Events
    event PaymentConfigured(address paymentToken, uint256 paymentAmount, address paymentReceiver);
    event AssetListed(bytes32 indexed assetId, address indexed handler, bytes32 key);
    event AssetRemoved(bytes32 indexed assetId);
    event AssetPurchased(bytes32 indexed assetId, address indexed buyer, address indexed recipient, uint256 paid);

    // Errors
    error UnknownAsset(bytes32 assetId);
    error WrongPaymentAmount(uint256 got, uint256 expected);
    error UnsafePaymentToken(address token);
}
```

### Handler Interface

Handlers encapsulate fulfillment logic for assets. The bucket contract delegates to the handler once payment is secured and state is updated.

```solidity
interface IAssetHandler {
    /// @notice Fulfill an asset purchase.
    /// @dev MUST revert on failure. MUST NOT reenter purchase().
    function fulfill(bytes32 key, address recipient) external;
}
```

* Each assetId maps to a `(handler, key)` pair.
* The key is an opaque identifier the handler interprets.

### Introspection

```solidity
interface IPayoutRaceIntrospection /* is IERC165 */ {
    function getHandler(bytes32 assetId) external view returns (address handler, bytes32 key);
}
```

Optional extension:

```solidity
interface IAssetDescribe {
    function describe(bytes32 key) external view returns (bytes memory data, string memory schema);
}
```

### Keys and packed identifiers

The `key` is an opaque identifier interpreted by the handler. Implementers are encouraged to **pack identifiers** needed for fulfillment directly into the `key` to minimize storage and simplify listing. Examples:

* Address packing: `bytes32(uint256(uint160(targetAddress)))` and recover with `address(uint160(uint256(key)))`.
* Address plus small domain: pack a pool address in the low 20 bytes and an 8 byte epoch in the high bits. Handlers can decode both to enforce one-claim-per-epoch semantics.
* Arbitrary selector: use the `key` to discriminate between multiple behaviors inside a single handler without additional storage.

This pattern keeps the core ERC generic. The Uniswap V3 example below demonstrates address packing, but the same approach applies to any protocol that needs to route a purchase to a specific destination.

### Interfaces used by reference handlers

```solidity
interface IERC20 { function transfer(address to, uint256 value) external returns (bool); }
interface IERC777 { function send(address to, uint256 amount, bytes calldata data) external; }
```

---

## Reference Handlers

### 1. ERC-20 Transfer Handler

```solidity
contract ERC20AssetHandler is IAssetHandler {
    struct Spec { address token; uint256 amount; }
    mapping(bytes32 => Spec) public specs;
    address public immutable bucket;

    constructor(address _bucket) { bucket = _bucket; }
    modifier onlyBucket() { require(msg.sender == bucket, "ONLY_BUCKET"); _; }

    function set(bytes32 key, address token, uint256 amount) external onlyBucket {
        specs[key] = Spec(token, amount);
    }

    function fulfill(bytes32 key, address recipient) external override onlyBucket {
        Spec memory s = specs[key];
        IERC20(s.token).transfer(recipient, s.amount);
        delete specs[key];
    }
}
```

### 2. ERC-777 Send Handler

```solidity
contract ERC777AssetHandler is IAssetHandler {
    struct Spec { address token; uint256 amount; bytes data; }
    mapping(bytes32 => Spec) public specs;
    address public immutable bucket;

    constructor(address _bucket) { bucket = _bucket; }
    modifier onlyBucket() { require(msg.sender == bucket, "ONLY_BUCKET"); _; }

    function set(bytes32 key, address token, uint256 amount, bytes calldata data) external onlyBucket {
        specs[key] = Spec(token, amount, data);
    }

    function fulfill(bytes32 key, address recipient) external override onlyBucket {
        Spec memory s = specs[key];
        IERC777(s.token).send(recipient, s.amount, s.data);
        delete specs[key];
    }
}
```

### 3. ETH Payout Handler

```solidity
contract ETHAssetHandler is IAssetHandler {
    mapping(bytes32 => uint256) public amountWei;
    address public immutable bucket;

    constructor(address _bucket) { bucket = _bucket; }
    modifier onlyBucket() { require(msg.sender == bucket, "ONLY_BUCKET"); _; }

    function set(bytes32 key, uint256 weiAmount) external onlyBucket payable {
        require(msg.value == weiAmount, "FUND_ETH");
        amountWei[key] = weiAmount;
    }

    function fulfill(bytes32 key, address recipient) external override onlyBucket {
        uint256 amt = amountWei[key];
        delete amountWei[key];
        (bool ok,) = recipient.call{value: amt}("");
        require(ok, "ETH_SEND_FAIL");
    }

    receive() external payable {}
}
```

### 4. Uniswap V3 Claim Handler

> Note: Implementers MAY pack additional identifier data directly into the `assetId`/`key`. For a Uniswap V3 style pool, that can be the pool address encoded as `bytes32(uint256(uint160(pool)))`, which the handler decodes. This example is provided only to illustrate key packing and does not impose any Uniswap-specific requirements on this ERC.

```solidity
interface IClaimFeesLike { function claimFees(address recipient) external; }

contract UniV3ClaimHandler is IAssetHandler {
    address public immutable bucket;

    constructor(address _bucket) { bucket = _bucket; }
    modifier onlyBucket() { require(msg.sender == bucket, "ONLY_BUCKET"); _; }

    // key encodes the pool address in the low 20 bytes of the bytes32
    function fulfill(bytes32 key, address recipient) external override onlyBucket {
        address pool = address(uint160(uint256(key)));
        IClaimFeesLike(pool).claimFees(recipient);
    }
}
```

Implementers that prefer explicit storage can keep a mapping `key → pool` and set it when listing. The packed-key variant avoids that storage at the cost of deriving the address from the key.

---

## Rationale

* **Global fixed price** simplifies integration. Governance can update `paymentAmount` as needed. Buyers guard with `expectedPaymentAmount`.
* **Per-asset purchase only**: removes ambiguity of “buy all.” Each asset is its own claimable unit.
* **Handlers** allow arbitrary logic (token transfers, claims, etc.). This makes the standard flexible enough for both Uniswap’s pools and simpler ERC-20 rewards.
* **Separation of concerns**: the bucket tracks listings, payments, and accounting; handlers execute fulfillment.

---

## Security Considerations

Implementations MUST:

* Follow **Checks-Effects-Interactions (CEI)**: update state (delist asset, update accounting, emit events) before calling `transferFrom` or `fulfill`.
* Use a **reentrancy guard** on `purchase`.
* Require that `paymentToken` is a plain ERC-20 (no ERC-777 hooks, no fee-on-transfer). If unsafe, revert with `UnsafePaymentToken`.
* Note that handlers may transfer ERC-777 or ETH, which can trigger recipient callbacks. Since state is updated first, this is safe.
* For ETH fulfillment, use `.call` and revert on failure. Never use `delegatecall`.
* For multi-use assets, handlers should decrement units themselves rather than relist.

---

## Backwards Compatibility

* This ERC generalizes UniStaker’s payout race. A UniStaker integration is just a bucket where each listed asset maps to a Uniswap pool in the `UniV3ClaimHandler`.
* Other protocols can instead list ERC-20 balances or ETH transfers.

---

## Reference Implementation

See included reference contracts above. A full reference implementation includes the `PayoutRace` contract that maintains handler mappings and enforces CEI + nonReentrant ordering.

---

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
