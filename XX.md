NIP-XX - Escrow
======
`draft` `optional` `author:PatrickGeyer` 

The purpose of this NIP is to allow two users of the nostr network to agree on an escrow service to sit between a value transfer. Reputation systems might be helpful, but for marketplace, hotel bookings and ride-sharing, stronger guarantees might be needed.

------------------
## Terms

- `buyer` - user initiating a transfer of value
- `seller` - user receiving a transfer of value
- `escrow` - entity which can complete or reverse the associated payment

An escrow can publish these events:
| Kind    |                  | Description                                                                                                   | NIP                                     |
|---------|------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| `0    ` | `set_meta`       | Announces itself (similar with any `nostr` public key).                                               | [NIP01       ](https://github.com/nostr-protocol/nips/blob/master/01.md)                            |
| `30017` | `set_service`      | Create or update an escrow service.                                                                                     | [NIP33](https://github.com/nostr-protocol/nips/blob/master/33.md) (Parameterized Replaceable Event) |
| `4    ` | `direct_message` | Communicate with the customer. The messages can be plain-text or JSON. | [NIP09](https://github.com/nostr-protocol/nips/blob/master/09.md)                                            |
| `5    ` | `delete`         | Delete an escrow service.                                                                                  | [NIP05](https://github.com/nostr-protocol/nips/blob/master/05.md)                                   |


A buyer and seller will both publish an event to indicate agreement on a specific escrow service. They do not need to do this at the time of trade. Like looking up who a nostr user follows, clients can lookup which escrows a user trusts.

| Kind    |                  | Description                                                                                                   | NIP                                     |
|---------|------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| `7    ` | `reaction`       | Announce agreement to an escrow service. Reference event id of escrow service announcement                                | [NIP07       ](https://github.com/nostr-protocol/nips/blob/master/07.md)                            |

Apps that use nostr escrows could automatically signal trust in a default escrow service on behalf of a user for the sake of ease of use.
E.g. A marketplace nostr app might run its own escrow service as well.
When the buyer is browsing a seller's stall on a client, the client will match up escrow services that the buyer and seller both trust.
It then knows which payment methods are available. If the seller has not indicated support for any escrows, the client should indicate that escrow is not available for this transaction and direct payment is the only option.

When the buyer wants to initiate a payment, it first must get an event id for the action it wants to escrow.
E.g. send the NIP-15 checkout message to indicate intent to purchase.

Then, send a message to the seller that you want to initiate an escrow transaction with a given service event id.
The below json goes in content of [NIP04](https://github.com/nostr-protocol/nips/blob/master/04.md).

```json
{
    "id": <String, UUID generated by the customer>,
    "type": 0,
    "escrow_service_id": <32-bytes hex of an event id>
    "order_id": <32-bytes hex of an event id, for example the NIP-15 checkout message>
}

```
This is to allow the buyer to choose the escrow for this transaction. Perhaps this could be set in a tag in the original transaction message?

The seller then contacts the escrow via direct message with the following content

```json
{
    "id": <String, UUID generated by the customer>,
    "type": 0,
    "order_id": <32-bytes hex of an event id, for example the NIP-15 checkout message>,
    "payment_hash" <optional hash of an ln invoice>
}

```

The escrow can then direct message the buyer with the following message: 
```json
{
    "id": <String, UUID of the order>,
    "type": 1,
    "message": <String, message to customer, optional>,
    "payment_options": [
        {
            "type": <String, option type>,
            "link": <String, url, btc address, ln invoice, etc>
        },
        {
            "type": <String, option type>,
            "link": <String, url, btc address, ln invoice, etc>
        },
                {
            "type": <String, option type>,
            "link": <String, url, btc address, ln invoice, etc>
        }
    ]
}
```

The buyer then satisfies the payment.

The escrow, upon payment receipt, then sends direct messages to both buyer and seller
```json
{
    "id": <String, UUID of the order>,
    "type": 1,
    "message": ACCEPTED/REVERSED/FORWARDED
    "proof": <String, optional proof of payment e.g. preimage>
}
```

If there is a dispute, both seller and buyer can direct message the escrow before the payment times out.