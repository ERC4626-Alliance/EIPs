---
eip: TODO
title: Asynchronous Tokenized Vaults with Cancellations
description: Extension of asynchronous ERC-4626 vaults with cancellation requests
author: Jeroen Offerijns (@hieronx), Alina Sinelnikova (@ilinzweilin), Vikram Arun (@vikramarun), Joey Santoro (@joeysantoro)
discussions-to: TODO
status: Draft
type: Standards Track
category: ERC
created: TODO
requires: 20, 4626
---

## Abstract

The following standard extends [ERC-X](./eip-x.md) by adding support for the cancellation of pending asynchronous deposit and redemption flows. The cancellation flows are called "Cancels".

New methods are added to either synchronously cancel or asychronously request a cancel for a deposit or redemption, and if done asynchronously, view the pending status of the cancellation request. The existing deposit, mint, withdraw, and redeem ERC-4626 methods are used for executing claimable Cancellations. 

Implementations can choose to whether to add cancellation flows for deposits, redemptions, or both. 

## Motivation

The ERC-4626 Asynchronous Tokenized Vault standard was introduced to make the original ERC-4626 standard more compatible with smart contract systems with asynchronous actions or delays as a prerequisite for interfacing with the Vault (e.g. real-world asset protocols, undercollateralized lending protocols, cross-chain lending protocols, liquid staking tokens, or insurance safety modules). However, unlike ERC-4626, which by design allows for the atomic deposit/mint/withdraw/redeem from vaults, ERC-X's `request*` functionality introduces the potential for user funds be stuck in between `requested` and `claimable` stages. This can happen in varying scales for a multitude of reasons, including: the capacity for the opportunity reaching a limit, changing requirements to invest, an opportunity increasing the length to maturity, and the protocol ceasing to operate. 

## Specification

### Definitions:

The existing definitions from [ERC-X](./eip-x.md) apply. In addition, this spec defines:

- cancel: a function call that synchronously cancels a pending deposit/redemption flow
- cancel request: a function call that initiates an asynchronous cancellation of pending deposit/redemption flow
- pending cancel: the state where a cancel request has been made but is not yet claimable
- claimable cancel: the Vault's step of processing the cancel request and enabling the user to claim corresponding assets (for async deposit) or shares (for async redeem)

### Cancel Flows

EIP-XC vaults MUST implement one or both of the deposit and redemption cancel flows. If either flow is not implemented in a request pattern, it MUST use the ERC-X asynchronous interaction pattern. 

All cancellable EIP-XC asynchronous tokenized vaults MUST implement ERC-X. While some implementations may support synchronous cancels, not all will, so we further define the Asynchronous Cancel lifecycle. 

### Asynchronous Cancel Lifecycle

After submission, Asynchronous Cancels go through Pending, Claimable, and Claimed stages. An example lifecycle for a async deposit cancel is visualized in the table below.

| **State**    | **User**                       | **Vault** |
| ------------:|:------------------------------ | ---------:|
| Pending      | requestDepositCancel()           |          |
| Claimable    |                                | <i>Internal request fulfillment</i><br>maxRedeem[msg.sender] += pendingDepositRequest[msg.sender]<br>pendingDepositRequest[msg.sender] = 0<br> |
| Claimed      | redeem(assets, receiver)       | maxRedeem[msg.sender] -= vault.previewRedeem[assets] |

An important vault inequality is that following a request(s), the cumulative requested quantity MUST be more than `pendingDepositRequest + maxDeposit - claimed`. The sources of inequality are fees and cancellation requests, otherwise this would be a strict equality.


### Methods

#### cancelDepositRequest

Cancels the outstanding `pendingDepositRequest` by the amount of `assets`, increasing `maxRedeem` and `maxWithdraw`, allowing for the synchronous `redeem` or `withdraw` from the Vault to receive `assets` that were previously locked for deposit.

MUST emit the `CancelDepositRequest` event.

```yaml
- name: cancelDepositRequest
  type: function
  stateMutability: nonpayable

    inputs:
    - name: assets
      type: uint256
```

#### cancelRedeemRequest

Cancels the outstanding `pendingRedeemRequest` by the amount of `shares`, increasing  `maxDeposit` and `maxMint`, allowing for the synchronous `deposit` or `mint` from the Vault to receive `shares` that were previously locked for redemption.

MUST emit the `CancelRedeemRequest` event.

```yaml
- name: cancelRedeemRequest
  type: function
  stateMutability: nonpayable

    inputs:
    - name: shares
      type: uint256
```

#### requestCancelDeposit

Submits an order to cancel the outstanding deposit request. When the request to cancel the deposit is fulfilled, `maxRedeem` and `maxWithdraw` will be increased, and `redeem` or `withdraw` from ERC-4626 can be used to receive `assets` that were previously locked for deposit.

MUST emit the `RequestCancelDeposit` event.

```yaml
- name: requestCancelDeposit
  type: function
  stateMutability: nonpayable
```

#### requestCancelRedeem

Submits an order to cancel the outstanding redemption request. When the request to cancel the redeem is fulfilled, `maxDeposit` and `maxMint` will be increased, and `deposit` or `mint` from ERC-4626 can be used to receive `shares` that were previously locked for redemption.

MUST emit the `RequestCancelRedeem` event.

```yaml
- name: requestCancelRedeem
  type: function
  stateMutability: nonpayable
```

### Events


#### CancelDepositRequest

`operator` has cancelled the pending deposit request by some amount of `assets`.

MUST be emitted when a cancel deposit request is submitted using the `cancelDepositRequest` method.

```yaml
- name: CancelDepositRequest
  type: event

  inputs:
    - name: operator
      indexed: true
      type: address
    - name: assets
      indexed: true
      type: uint256
```

#### CancelRedeemRequest

`operator` has cancalled pending redemption request by some amount of `shares`.

MUST be emitted when a cancel redemption request is submitted using the `cancelRedeemRequest` method.

```yaml
- name: CancelRedeemRequest
  type: event

  inputs:
    - name: operator
      indexed: true
      type: address
    - name: shares
      indexed: true
      type: uint256
```

#### RequestDepositCancel

`operator` has requested to cancel the pending deposit request.

MUST be emitted when a cancel deposit request is submitted using the `requestDepositCancel` method.

```yaml
- name: RequestDepositCancel
  type: event

  inputs:
    - name: operator
      indexed: true
      type: address
```

#### RequestRedeemCancel

`operator` has requested to cancel the pending redeem request. 

MUST be emitted when a cancel redemption request is submitted using the `requestRedeemCancel` method.

```yaml
- name: RequestRedeemCancel
  type: event

  inputs:
    - name: operator
      indexed: true
      type: address
```

## Rationale

### Symmetry and Non-inclusion of requestWithdraw and requestMint

In ERC-4626, the spec was written to be fully symmetrical with respect to converting assets and shares by including deposit/withdraw and mint/redeem.

Due to the asynchronous nature of requests, the vault can only operate with certainty on the quantity that is fully known at the time of the request (`assets` for `deposit` and `shares` for `redeem`). The deposit request flow cannot work with a `mint` call, because the amount of `assets` for the requested `shares` amount may fluctuate before the fulfillment of the request. Likewise, the redemption request flow cannot work with a `withdraw` call.

### Optionality of flows and cancels

Certain use cases are only asynchronous on one flow but not the other between request and redeem. A good example of an asynchronous redemption vault is a liquid staking token. The unstaking period necessitates support for asynchronous withdrawals, however, deposits can be fully synchronous.

In many cases, canceling a request may not be straightforward or even technically feasible, therefore cancel operations are optional. Defining the cancel flow is still important for certain classes of use cases for which the fulfillment of a Request can take a considerable amount of time.

### Request Implementation flexibility

The standard is flexible enough to support a wide range of interaction patterns for request flows. Pending requests can be handled via internal accounting, globally or on per-user levels, use ERC-20 or [ERC-721](./eip-721.md), etc.

Likewise yield on redemption requests can accrue or not, and the exchange rate of any request may be fixed or variable depending on the implementation.

### Not allowing short-circuiting for claims
If claims can short circuit, this creates ambiguity for integrators and  complicates the interface with overloaded behavior on request functions.

Instead there can be router contracts which atomically check for claimable amounts immediately upon request. Frontends can dynamically route requests in this way depending on the state and implementation of the vault.

### Operator function parameter on requestDeposit and requestRedeem

To support flows where a smart contract manages the request lifecycle on behalf of a user, the `operator` parameter is included in the `requestDeposit` and `requestRedeem` functions. This is not called `owner` because the `assets` or `shares` are not transferred from this account on request submission, unlike the behaviour of an `owner` on `redeem`. It is also not called `receiver` because the `shares` or `assets` are not necessarily transferred on claiming the request, this can be chosen by the operator when they call `deposit`, `mint`, `redeem`, or `withdraw`.

## Backwards Compatibility

The interface is fully backwards compatible with [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626). The specification of the `deposit`, `mint`, `redeem`, and `withdraw` methods is different as described in [Specification](#specification).

## Reference Implementation

Centrifuge has been developing [an implementation](https://github.com/centrifuge/liquidity-pools/blob/72f1ddddf3493db5e166f6c3317a6a5c27675eeb/src/LiquidityPool.sol) that can provide a reference.

## Security Considerations

The methods `pendingDepositRequest` and `pendingRedeemRequest` are estimates useful for display purposes, and can be outdated due to the asynchronicity.

In general, asynchronicity concerns make state transitions in the vault much more complex and vulnerable to security risks. Access control on vault operations, clear documentation of state transitioning, and invariant checks should all be performed to mitigate these risks.

It is worth highlighting again here that the claim functions for any asynchronous flows MUST enforce that `msg.sender == operator/owner` to prevent theft of claimable `assets` or `shares`

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).