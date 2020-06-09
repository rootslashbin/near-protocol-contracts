# near-protocol-contracts

This is using flow adapted for https://github.com/nearprotocol/near-bindgen/blob/master/examples/fun-token/src/lib.rs#L63 as token.

1) Consumer calls set_allowance on token contract to allow oracle contract to charge given account.
2) Consumer calls oracleRequest in oracle contract.
3) oracleRequest calls lock in token contract to lock tokens to be used for payment.
4) Oracle contract gets called by provider with request result (`fulfillOracleRequest`).
5) Oracle contract calls transfer_from on token contract to transfer previously locked tokens from consumer account to provider account.

Alternatively:
instead of doing lock before fulfillment and then transfer_from after fulfillment it's possible to just charge immediately. The only catch here is around what happens in case of error / request not being fulfilled.

Notes:
- 128-bit numbers confirmed to be enough for payment, nonce and dataVersion
- specId  is the same as a Job ID. The specs themselves must be defined by the node, and the requester initiates a run of that spec by providing its Job ID (or Spec ID,, these terms can be used interchangeably). The specs do not have to be from a pre-defined set, it's up to the node operator to create them. It is not possible, and not advised, for a requester to be able to pass in the full JSON of a job. That opens up the node to attack from malicious job specs that they haven't vetted.
- data should be straight JSON instead of JSON-like CBOR

## Set up ability to run on testnet
Create a NEAR testnet account with [Wallet](https://wallet.testnet.near.org).
Create a subaccounts in this fashion:

    near create_account oracle.you.testnet --masterAccount you.testnet
    near create_account oracle-client.you.testnet --masterAccount you.testnet
    near create_account oracle-node.you.testnet --masterAccount you.testnet
    near create_account near-link.you.testnet --masterAccount you.testnet

**Oracle client** will call the **oracle contract** to make a request for external data.
**Oracle client** has given the **oracle contract** allowance to take NEAR LINK from it. Before officially adding the request, it will `transfer_from` to capture the payment, keeping track of this amount in the `withdrawable_token` state variable.
The **oracle node** will be polling the state of its **oracle contract** using `get_all_requests` (which will be paginated in an update)

Build the oracle, oracle-client, and NEAR LINK contracts with:

    ./build_all.sh
    
Then deploy and instantiate like so…

NEAR LINK
###

    near deploy --accountId near-link.you.testnet --wasmFile near-link-token/res/near_link_token.wasm
    near call near-link.you.testnet new '{"owner_id": "near-link.you.testnet", "total_supply": "1000000"}' --accountId near-link.you.testnet
    
Oracle contract
###

    near deploy --accountId oracle.you.testnet --wasmFile oracle/res/oracle.wasm
    near call oracle.you.testnet new '{"link_id": "near-link.you.testnet", "owner_id": "oracle.you.testnet"}' --accountId oracle.you.testnet
    
Oracle client
###

This contract is very bare-bones and does not need an initializing call with `new`

    near deploy --accountId oracle-client.you.testnet --wasmFile oracle-client/res/oracle_client.wasm
    
## Give fungible tokens and set allowances

Give 50 NEAR LINK to oracle-client:

    near call near-link.you.testnet transfer '{"new_owner_id": "oracle-client.you.testnet", "amount": "50"}' --accountId near-link.you.testnet
    
(Optional) Check balance to confirm:

    near view near-link.you.testnet get_balance '{"owner_id": "oracle-client.you.testnet"}'
    
**Oracle client** gives **oracle contract** allowance to spend 20 NEAR LINK on their behalf:

    near call near-link.you.testnet set_allowance '{"escrow_account_id": "oracle.you.testnet", "allowance": "20"}' --accountId oracle-client.you.testnet
    
(Optional) Check allowance to confirm:

    near view near-link.you.testnet get_allowance '{"owner_id": "oracle-client.you.testnet", "escrow_account_id": "oracle.you.testnet"}'
    
**Oracle client** makes a request to **oracle contract** with payment of 10 NEAR LINK:

    near call oracle.you.testnet request '{"payment": "10", "spec_id": "dW5pcXVlIHNwZWMgaWQ=", "callback_address": "oracle-client.you.testnet", "callback_method": "token_price_callback", "nonce": "1", "data_version": "1", "data": "QkFU"}' --accountId oracle-client.you.testnet --gas 10000000000000000
    
Before the **oracle node** can fulfill the request, they must be authorized.

    near call oracle.you.testnet add_authorization '{"node": "oracle-node.you.testnet"}' --accountId oracle.you.testnet
    
(Optional) Check authorization to confirm:

    near view oracle.you.testnet is_authorized '{"node": "oracle-node.you.testnet"}'   
         
Oracle node is polling the state of **oracle contract** to see the request(s):

    near view oracle.you.testnet get_all_requests
    
It sees the `data` is `QkFU` which is the Base64-encoded string for `BAT`, the token to look up. The **oracle node** presumably makes a call to an exchange to gather the price of Basic Attention Token (BAT) and finds it is at $0.19 per token.
The data `0.19` as a Vec<u8> is `MTkuMQ==`
**Oracle node** uses its NEAR account keys to fulfill the request:

    near call oracle.you.testnet fulfill_request '{"request_id": "oracle-client.you.testnet:1", "payment": "10", "callback_address": "oracle-client.you.testnet", "callback_method": "token_price_callback", "expiration": "1906293427246306700", "data": "MTkuMQ=="}' --accountId oracle-node.you.testnet --gas 10000000000000000
    
(Optional) Check the balance of **oracle client**:

    near view near-link.you.testnet get_balance '{"owner_id": "oracle-client.you.testnet"}'
    
Expect `40`
    
(Optional) Check the allowance of **oracle contract**:

    near view near-link.you.testnet get_allowance '{"owner_id": "oracle-client.you.testnet", "escrow_account_id": "oracle.you.testnet"}'
    
Expect `10`
