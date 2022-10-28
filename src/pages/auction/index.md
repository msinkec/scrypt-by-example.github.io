---
title: Auction
version: 0.1.0
description: Auction in sCrypt
---

The auction contract implements two public functions:

- `bid` - If a higher bid is found, the current winner is updated and the previous highest bidder is refunded.
- `close` - The auctioneer can close the auction after it expires and take the offer.


```javascript
// Auction: highest bid before deadline wins
contract Auction {
    @state
    Ripemd160 bidder;

    PubKey auctioner;
    int auctionDeadline;

    // bid with a higher offer
    public function bid(Ripemd160 bidder, int bid, int changeSats, SigHashPreimage txPreimage) {
        int highestBid = SigHash.value(txPreimage);
        require(bid > highestBid);

        Ripemd160 highestBidder = this.bidder;
        this.bidder = bidder;

        // auction continues with a higher bidder
        bytes stateScript = this.getStateScript();
        bytes auctionOutput = Utils.buildOutput(stateScript, bid);

        // refund previous highest bidder
        bytes refundScript = Utils.buildPublicKeyHashScript(highestBidder);
        bytes refundOutput = Utils.buildOutput(refundScript, highestBid);

        bytes changeScript = Utils.buildPublicKeyHashScript(bidder);
        bytes changeOutput = Utils.buildOutput(changeScript, changeSats);

        bytes output = auctionOutput + refundOutput + changeOutput;

        require(this.propagateState(txPreimage, output));
    }

    // withdraw after bidding is over
    public function close(Sig sig, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        require(SigHash.nLocktime(txPreimage) >= this.auctionDeadline);
        require(checkSig(sig, this.auctioner));
    }

    function propagateState(SigHashPreimage txPreimage, bytes outputs) : bool {
        require(Tx.checkPreimage(txPreimage));
        return (hash256(outputs) == SigHash.hashOutputs(txPreimage));
    }
}
```
