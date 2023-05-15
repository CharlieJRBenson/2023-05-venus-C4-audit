## Impact

Attacker has the ability to bid small and prevent any contesting bids, effectively able to purchase the `convertibleBaseAsset` accumulated in `RiskFund` for the price they choose.

## Proof of Concept

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L179-L194

```
if (auction.auctionType == AuctionType.LARGE_POOL_DEBT) {
    if (auction.highestBidder != address(0)) {
        uint256 previousBidAmount = ((auction.marketDebt[auction.markets[i]] * auction.highestBidBps) /
            MAX_BPS);
        erc20.safeTransfer(auction.highestBidder, previousBidAmount);
    }

    uint256 currentBidAmount = ((auction.marketDebt[auction.markets[i]] * bidBps) / MAX_BPS);
    erc20.safeTransferFrom(msg.sender, address(this), currentBidAmount);
} else {
    if (auction.highestBidder != address(0)) {
        erc20.safeTransfer(auction.highestBidder, auction.marketDebt[auction.markets[i]]);
    }

    erc20.safeTransferFrom(msg.sender, address(this), auction.marketDebt[auction.markets[i]]);
}
```

In the above code segment from the `Shortfall` contract, the auction receives token payment as a bid. Upon receiving a new highest bid, the function refunds the previously highest bidder. If the refund `erc20.safeTransfer()` reverts the transaction will be cancelled and the new bidder not accepted. 
Therefore an attacker would need to ensure their refund always reverts before the auction is closed.

There are situations in which an attacker may be able to ensure their refund transactions can be reverted. For example:
- If one of the market's underlying ERC20 token is compliant with the ERC777 token standard, in which a `tokensReceived()` hook is called when a contract receives the token, the attacker can implement this hook to revert when their malicious contract receives the refund.
This revert will cause the DOS, to deny any new higher bidders.

- If one of the market's underlying ERC20 token contains a blacklist or rate-limited transactions. The user can leverage these to allow them to place an initial bid and for the refund transaction to be reverted by the token contract itself thereafter when any competing bid is placed.

## Tools Used
Manual Review

## Recommended Mitigation Steps

Depending on the intention of the protocol:

- Keep underlying tokens used by protocol markets to a limited/trusted few ERC20's.

If the protocol is forecast to accommodate many tokens including ERC777:

- Implement the payment withdrawal pattern instead of actively sending payment within the `for` loop of the same bid function. This way the refund is in a separate transaction from the bidding. E.g. create a `withdrawPreviousBid` function.

- Or implement a catch for any reverting transactions.