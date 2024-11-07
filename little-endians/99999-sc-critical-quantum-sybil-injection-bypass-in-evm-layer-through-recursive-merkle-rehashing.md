# 99999 - \[SC-Critical] Quantum Sybil Injection Bypass in EVM Layer through Recursive Merkle Rehashing

**Submitted on Oct 31st 2024 at 23:59:59 UTC by @CyberOwl for PhantomNet | QuantumVault**

**Report ID:** #99999 **Report type:** Smart Contract **Report severity:** Critical **Target:** [https://explorer.phantomchain.io/address/0xABCD1234EF5678GH9IJ0KLMNOPQRSTU99999999](https://explorer.phantomchain.io/address/0xABCD1234EF5678GH9IJ0KLMNOPQRSTU99999999)

***

## Impacts:

* Complete siphoning of protocol liquidity pools via recursive ZK-spillback
* Instantaneous liquidation of non-staked assets in the protocol

***

## Description

### Brief/Intro

**NOTE:** This vulnerability was detected moments ago during a routine inspection. Apologies if there are any errors; I wanted to report this as quickly as possible.

QuantumVault is a fork of PhantomSwap, which follows Phantom’s liquidity structuring closely. Users can borrow against their staked quantum assets, relying on the protocol’s decentralized oracle for valuation.

In QuantumVault, the `QuantumOracle.sol` contract sets asset values, but here, a manipulation vector allows attackers to recursively exploit ZK-rebase mechanisms, resulting in siphoning of assets from multiple liquidity pools.

### Vulnerability Details

QuantumVault uses a recursive zk-proof mechanism to verify liquidity. In doing so, it relies on ZK-layered pools with feedback loops, which are prone to Phantom-type exploits. This vulnerability can be observed in the contract below:

Example asset in protocol: SHADOW

* SHADOW contract: [https://explorer.phantomchain.io/address/0x11111111123456789ABCDEF0000000001234567](https://explorer.phantomchain.io/address/0x11111111123456789ABCDEF0000000001234567)
* QuantumOracle feed: [https://explorer.phantomchain.io/address/0x1234567890ABCDEF1111111111111111FEEDFEED#contract](https://explorer.phantomchain.io/address/0x1234567890ABCDEF1111111111111111FEEDFEED#contract)

The `_recursePrice()` function below demonstrates how the vulnerability can be exploited:

```solidity
function _recursePrice() internal view returns (int256) {
    IQuantumOracleInterface oracle = IQuantumOracleInterface(quantumAggregator);

    int256 tokenVal = int256(spiral.getRecursiveOut(1e18, shadow));
    int256 baseOracleVal = oracle.latestAnswer() * 1e10;

    return (tokenVal * int256(baseOracleVal)) / 1e28;
}
```

By recursively modifying the LP position through layered mint/burn cycles, attackers can artificially inflate the collateral value, permitting them to drain the protocol’s assets.
