 #1 Malicios user can spam orders that expire immediately or cancel them immediately to wipe out legit orders from a book with orders only on one side.

 Finding description and impact
When creating an order, there are no restrictions on a minimum time the order can be valid, meaning a user can create an order with a cancelTimestamp that is the very next second.

Also, there is an upper bound on how many limits each side of a book can have - maxNumLimitsPerSide. This is enforced during limit creation - if the book is full the worst limit will be removed:
```solidity
  /// @dev Performs the core matching and placement of a bid limit order into the book
    function _executeBidLimitOrder(Book storage ds, Order memory newOrder, LimitOrderType limitOrderType)
        internal
        returns (uint256 postAmount, uint256 quoteTokenAmountSent, uint256 baseTokenAmountReceived)
    {
        if (limitOrderType == LimitOrderType.POST_ONLY && ds.getBestAskPrice() <= newOrder.price) {
            revert PostOnlyOrderWouldFill();
        }

        // Attempt to fill any of the incoming limit order that's overlapping into asks
        (uint256 totalQuoteSent, uint256 totalBaseReceived) = _matchIncomingBid(ds, newOrder, true);

        // NOOP, there is no more size left after filling to create a limit order
        if (newOrder.amount < ds.settings().minLimitOrderAmountInBase) {
            newOrder.amount = 0;
            return (postAmount, totalQuoteSent, totalBaseReceived);
        }

        // The book is full, pop the least competitive order (or revert if incoming is the least competitive)
>       if (ds.bidTree.size() == maxNumLimitsPerSide) {
            uint256 minBidPrice = ds.getWorstBidPrice();
            if (newOrder.price <= minBidPrice) revert MaxOrdersInBookPostNotCompetitive();

            _removeNonCompetitiveOrder(ds, ds.orders[ds.bidLimits[minBidPrice].tailOrder]);
        }

        // After filling, the order still has sufficient size and can be placed as a limit
        ds.addOrderToBook(newOrder);
        postAmount = ds.getQuoteTokenAmount(newOrder.price, newOrder.amount);

        return (postAmount, totalQuoteSent, totalBaseReceived);
    } 
```
As a result this opens an attack vector where a malicios user can wipe out all legit orders from a book by submitting many orders of higher prices that expire immediately. The attacker can get around the maximum number of limits per transaction by using separate accounts.

#2. Order double-linked list is broken because order.prevOrderId is not persisted

Orders are stored as a double-linked list in the book.

However, this linking is broken as order.prevOrderId is not actually stored in EVM storage:
```solidity
function addOrderToBook(Book storage self, Order memory order) internal {
151:         Limit storage limit = _updateBookPostOrder(self, order);
152: 
153:       _updateLimitPostOrder(self, limit, order);
154:     }
```
As we can see in the above code, linked list is updated in the end. And order is passed as memory type:
```solidty
function _updateLimitPostOrder(Book storage self, Limit storage limit, Order memory order) private {
276:         limit.numOrders++;
277: 
278:         if (limit.headOrder.isNull()) {
279:             limit.headOrder = order.id;
280:             limit.tailOrder = order.id;
281:         } else {
282:             Order storage tailOrder = self.orders[limit.tailOrder];
283:             tailOrder.nextOrderId = order.id;
284:@>           order.prevOrderId = tailOrder.id;
285:             limit.tailOrder = order.id;
286:         }
287: 
288:         emit LimitOrderCreated(BookEventNonce.inc(), order.id, order.price, order.amount, order.side);
289:     }
In L284, order.prevOrderId is updated. However, it is not stored in EVM storage because order is passed as memory in L275.

This completely breaks linked list in future operations. For example, when removing orders:

File: contracts/clob/types/Book.sol

320:         OrderId prev = order.prevOrderId;
321:         OrderId next = order.nextOrderId;
322: 
323:         if (!prev.isNull()) self.orders[prev].nextOrderId = next;
324:         else limit.headOrder = next;
325: 
326:         if (!next.isNull()) self.orders[next].prevOrderId = prev;
327:         else limit.tailOrder = prev;
```

#3.   Order double-linked list is broken because order.prevOrderId is not persisted
order.prevOrderId is updated only in memory and not saved to storage, breaking the linked list. This causes order adding and removal issues, potentially leading to denial of service when the order book is full and the limit’s tailOrder becomes invalid.

Orders are stored as a double-linked list in the book.
```solidity
 function addOrderToBook(Book storage self, Order memory order) internal {
151:         Limit storage limit = _updateBookPostOrder(self, order);
152: 
153:@>       _updateLimitPostOrder(self, limit, order);
154:     }
As we can see in the above code, linked list is updated in the end. And order is passed as memory type:

File: contracts/clob/types/Book.sol

275:     function _updateLimitPostOrder(Book storage self, Limit storage limit, Order memory order) private {
276:         limit.numOrders++;
277: 
278:         if (limit.headOrder.isNull()) {
279:             limit.headOrder = order.id;
280:             limit.tailOrder = order.id;
281:         } else {
282:             Order storage tailOrder = self.orders[limit.tailOrder];
283:             tailOrder.nextOrderId = order.id;
284:@>           order.prevOrderId = tailOrder.id;
285:             limit.tailOrder = order.id;
286:         }
287: 
288:         emit LimitOrderCreated(BookEventNonce.inc(), order.id, order.price, order.amount, order.side);
289:     }
In L284, order.prevOrderId is updated. However, it is not stored in EVM storage because order is passed as memory in L275.

This completely breaks linked list in future operations. For example, when removing orders:
```

#4. Dust orders can block order posting
When matching incoming orders, maker orders can be reduced below the minimum limit without checks, resulting in dust positions remaining in the order book. These dust orders can block new incoming orders, as matching them may revert with a ZeroCostTrade error if the quote amount rounds down to zero.

Description
When matching an incoming order, maker order's amount can be set below minLimitOrderAmountInBase, as there is no min amount check:
```solidity
833:@>       bool orderRemoved = matchData.baseDelta == matchedBase;
834: 
835:         // Handle token accounting for maker.
836:         if (takerOrder.side == Side.BUY) {
837:             TransientMakerData.addQuoteToken(makerOrder.owner, matchData.quoteDelta);
838: 
839:             if (!orderRemoved) ds.metadata().baseTokenOpenInterest -= matchData.baseDelta;
840:         } else {
841:             TransientMakerData.addBaseToken(makerOrder.owner, matchData.baseDelta);
842: 
843:             if (!orderRemoved) ds.metadata().quoteTokenOpenInterest -= matchData.quoteDelta;
844:         }
845: 
846:         if (orderRemoved) ds.removeOrderFromBook(makerOrder);
847:@>       else makerOrder.amount -= matchData.baseDelta;
In L833, order is removed only when matched amount is equal to maker order's amount
In L847, maker order's amount is decreased by matched amount. The result amount can be dust, as there is no min amount check done afterwards.
As such, maker order's amount can be a dust amount (up to lotSizeInBase config).

This means there can exist a dust position in the order book.

What happens if this dust order is matched to another incoming order?

File: contracts/clob/CLOB.sol

819:             matchData.baseDelta = (matchedBase.min(takerOrder.amount) / lotSize) * lotSize;
820:             matchData.quoteDelta = ds.getQuoteTokenAmount(matchedPrice, matchData.baseDelta);
In L819, matchData.baseDelta can be as small as lotSize, since makeOrder.amount can be down to lotSize
In L820, matchData.quoteDelta can be zero due to rounddown:
File: contracts/clob/types/Book.sol

471:     function getQuoteTokenAmount(Book storage self, uint256 price, uint256 baseAmount)
472:         internal
473:         view
474:         returns (uint256 quoteAmount)
475:     {
476:         return baseAmount * price / self.config().baseSize;
477:     }
If matchData.quoteDelta is 0, the trade(or order posting) will revert with ZeroCostTrade error:

File: contracts/clob/CLOB.sol

439: if (totalQuoteSent == 0 || totalBaseReceived == 0) revert ZeroCostTrad
```

#5. Missing Per-Transaction Limit Enforcement in `amend()` Allows Order-Flood Denial-of-Service
CLOB.postLimitOrder() enforces a per-transaction cap (maxLimitsPerTx) on how many unique price levels a caller may touch, throttling spam order creation.

However, CLOB.amend() never calls BookLib.incrementLimitsPlaced(). This means, the amend() function on CLOB markets bypasses the per-transaction “max limits” throttle that postLimitOrder() enforces.

A malicious trader can therefore:

Seed thousands of tiny, deep-in-the-book limit orders over many cheap transactions (each respecting the cap).
In one later transaction, mass-amend every order’s price (or side) so they all land at, or just inside, the best bid/ask level.

```solidity
/// @notice Posts a limit order for `account`
    function postLimitOrder(address account, PostLimitOrderArgs calldata args)
        external
        onlySenderOrOperator(account, OperatorRoles.CLOB_LIMIT)
        returns (PostLimitOrderResult memory)
    {
        Book storage ds = _getStorage();

        ds.assertLimitPriceInBounds(args.price);
        ds.assertLimitOrderAmountInBounds(args.amountInBase);
        ds.assertLotSizeCompliant(args.amountInBase);

        // Max limits per tx is enforced on the caller to allow for whitelisted operators
        // to implement their own max limit logic.
        ds.incrementLimitsPlaced(address(factory), msg.sender);

        uint256 orderId;
        if (args.clientOrderId == 0) {
            orderId = ds.incrementOrderId();
        } else {
            orderId = account.getOrderId(args.clientOrderId);
            ds.assertUnusedOrderId(orderId);
        }

        Order memory newOrder = args.toOrder(orderId, account);

        if (newOrder.isExpired()) revert OrderExpired();

        emit LimitOrderSubmitted(CLOBEventNonce.inc(), account, orderId, args);

        if (args.side == Side.BUY) return _processLimitBidOrder(ds, account, newOrder, args);
        else return _processLimitAskOrder(ds, account, newOrder, args);
    }
```
In postLimitOrder(), the call ds.incrementLimitsPlaced(address(factory), msg.sender); enforces maxLimitsPerTx in BookLib.

However, amend → _processAmend → _executeAmendNewOrder / _executeAmendAmount has no call to incrementLimitsPlaced; unlimited amendments in same tx. This means, the amend() entrypoint never calls incrementLimitsPlaced().

Recommendations
Mirror the cap in amend(). Before each order-price move, call ds.incrementLimitsPlaced(address(factory), msg.sender); reverting if the caller already hit maxLimitsPerTx in that tx.

 function amend(address account, AmendArgs calldata args)
     external onlySenderOrOperator(account, OperatorRoles.CLOB_LIMIT)
     returns (int256 quoteDelta, int256 baseDelta)
 {
     Book storage ds = _getStorage();
+    // Enforce maxLimitsPerTx on amends, just like on new posts
+    ds.incrementLimitsPlaced(address(factory), msg.sender);

     Order storage order = ds.orders[args.orderId.toOrderId()];
     // …existing validations…
     (quoteDelta, baseDelta) = _processAmend(ds, order, args);

 }

 
 #7. Removing only the tail order from a limit does not reduce tree size, allowing order book to grow indefinitely
When the tree size limit is reached, the CLOB only removes the least competitive order (tail order) from a limit, but does not remove the limit itself if it contains multiple orders. As a result, the tree size does not decrease, allowing the order book to bypass the intended size restriction and grow without bound.

Description
When the tree size reaches the limit, CLOB will remove the most non-competitive order from the book:
```solidity
   // The book is full, pop the least competitive order (or revert if incoming is the least competitive)
633:         if (ds.askTree.size() == maxNumLimitsPerSide) {
634:             uint256 maxAskPrice = ds.getWorstAskPrice();
635:             if (newOrder.price >= maxAskPrice) revert MaxOrdersInBookPostNotCompetitive();
636: 
637:@>           _removeNonCompetitiveOrder(ds, ds.orders[ds.askLimits[maxAskPrice].tailOrder]);
638:         }
The problem is that it only remove tailOrder from the limit. If a limit contains multiple orders, the limit will not be removed from the tree:

File: contracts/clob/types/Book.sol

307:@>       if (limit.numOrders == 1) {
308:             if (order.side == Side.BUY) {
309:@>               delete self.bidLimits[price];
310:                 self.bidTree.remove(price);
311:             } else {
312:@>               delete self.askLimits[price];
313:                 self.askTree.remove(price);
314:             }
315:             return;
316:         }
Since RedBlackTree.size() returns total number of nodes (i.e. limits), tree size will not be reduced even after removing the tail order.

Indeed, in order to keep the tree size limit, the code should remove all orders from the given limit, not just the tail.
Recommended mitigation steps
Consider implementing _removeNonCompetitiveLimit instead of removing just the tail order.

#8. Removing only the tail order from a limit does not reduce tree size, allowing order book to grow indefinitely

When the tree size limit is reached, the CLOB only removes the least competitive order (tail order) from a limit, but does not remove the limit itself if it contains multiple orders. As a result, the tree size does not decrease, allowing the order book to bypass the intended size restriction and grow without bound.

Description
When the tree size reaches the limit, CLOB will remove the most non-competitive order from the book:

File: contracts/clob/CLOB.sol

632:         // The book is full, pop the least competitive order (or revert if incoming is the least competitive)
633:         if (ds.askTree.size() == maxNumLimitsPerSide) {
634:             uint256 maxAskPrice = ds.getWorstAskPrice();
635:             if (newOrder.price >= maxAskPrice) revert MaxOrdersInBookPostNotCompetitive();
636: 
637:@>           _removeNonCompetitiveOrder(ds, ds.orders[ds.askLimits[maxAskPrice].tailOrder]);
638:         }
The problem is that it only remove tailOrder from the limit. If a limit contains multiple orders, the limit will not be removed from the tree:

File: contracts/clob/types/Book.sol

307:@>       if (limit.numOrders == 1) {
308:             if (order.side == Side.BUY) {
309:@>               delete self.bidLimits[price];
310:                 self.bidTree.remove(price);
311:             } else {
312:@>               delete self.askLimits[price];
313:                 self.askTree.remove(price);
314:             }
315:             return;
316:         }
Since RedBlackTree.size() returns total number of nodes (i.e. limits), tree size will not be reduced even after removing the tail order.

Recommended mitigation steps
Consider implementing _removeNonCompetitiveLimit instead of removing just the tail order

#9. FOK orders wrongly revert on dust residual amounts below lot size
The CLOB contract unnecessarily reverts FOK orders when the remaining unmatched amount is gte (pun intended) 0, but smaller than lotSizeInBase, contrary to the documented behavior. This leads to reverts of multi-hop swaps where exact output amounts from UniswapV2 cannot be adjusted to match CLOB's lot size.
It can be understood in either one of the following ways:

If the FOK fill order amount is slightly less than the matching limit order, the transaction should not revert
If the FOK fill order amount is slightly greater than the matching limit order, the transaction should not revert
However, the current implementation does not behave as described.

For example, let's consider there are a limit ask order and a fill bid order, with lotSizeInBase = 1e6

Case 1 If fill bid order amount is slightly less than limit ask order:

#10.Flawed Zero-Cost Trade Prevention

In CLOB.sol, the validation logic incorrectly uses bitwise operations to detect zero-value trades, creating two critical issues:
If baseTokenAmountReceived = 1 and quoteTokenAmountSent = 2, then baseTokenAmountReceived & quoteTokenAmountSent = 0.
```solidity
 if (baseTokenAmountReceived != quoteTokenAmountSent && baseTokenAmountReceived & quoteTokenAmountSent == 0) {
            revert ZeroCostTrade();
        }

b/main/contracts/clob/CLOB.sol#L544-L546

@>        if (baseTokenAmountSent != quoteTokenAmountReceived && baseTokenAmountSent & quoteTokenAmountReceived == 0) {
            revert ZeroCostTrade();
        }
```




```
 
