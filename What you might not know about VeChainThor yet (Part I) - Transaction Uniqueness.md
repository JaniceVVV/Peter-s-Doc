# What you might not know about VeChainThor yet (Part I) - Transaction Uniqueness

It has been almost one year since the launch of the VeChainThor blockchain last June. As a co-author of its [whitepaper](https://cdn.vechain.com/vechainthor_development_plan_and_whitepaper_en_v1.0.pdf), I am still feeling that some of its important features, designed for mass adoption of blockchain technology, have been largely out of public sight, despite the fact that they have been extensively used in many applications running on VeChainThor. 

This is the first article of the series "What you might not know about VeChainThor yet" in which I will talk in details about those useful but less mentioned features of VeChainThor. By the way, if you have not known the [Multi-party Payment Protocol (MPP)](https://doc.vechainworld.io/docs/multi-party-payment-protocol-mpp) or [Multi-Task Transaction (MTT)](https://doc.vechainworld.io/docs/transaction-model), two of the most powerful features of VeChainThor, I strongly recommend you to read the relevant documents.

## Transaction uniqueness
Every blockchain system has to find a way to uniquely identify each transaction (TX), or otherwise, it would be vulnerable to the TX replay attack. Simply speaking, it is an attack where the attacker tries to use a known TX in the ledger to fool the system.
<!--In such an attack, an attacker downloads a valid TX from the public ledger and tries to reuse this TX. Very often, a TX is identified by a transaction ID (TXID) which could be the output of a cryptographic hash function.-->

For a UTXO-based blockchain like Bitcoin, TXs are linked and can be uniquely identified and verified by the associated spending history. However, such uniqueness no longer holds for an account-based blockchain like Ethereum. For such systems, we need to inject some extra information into TXs to make them uniquely identifiable. 

## Ethereum's solution
Ethereum achieves TX uniqueness by adding `AccountNonce` to each TX and setting the rule that a pending TX can only be processed when the `AccountNonce` value equals to the latest `Nonce` value of the account that sends the TX. Note that the `Nonce` value is the number of transactions the account has sent so far. A TX is then uniquely identified by `AccountNonce` together with the sender's account address. 

However, a downside of such a setting is that TXs sent from the same account are forced to stay in an artificial order and any failed TX could cause TXs sent afterwards and therefore having a larger `Nonce` value to become unacceptable. Moreover, when you have multiple pending TXs from your account and want to send a new TX, you will have to take the risk of having your new TX stuck in the TX pool if any of the pending TXs fails. 

## VeChainThor's solution
VeChainThor is an account-based blockchain system and it achieves TX uniqueness as follows. First, it defines the TX `Nonce` as a 64-bit unsigned integer that is determined totally by the TX sender. Given a TX, it computes two hashes, the hash of the RLP encoded TX data without the signature and the hash of the previously computed hash concatenated by the sender's account address. The second hash which is 256-bit long, is used as the TXID to uniquely identify the given TX on VeChainThor. Note that the calculation of the TXID does not require a private key to sign the TX.

## Why it matters?
Why does such a design matter? Well, first of all, the uniqueness of a TX is completely dependent on its contents and whether or not it will be processed by nodes is purely determined by whether its TXID has existed on-chain, rather than by the state status of the sender's account (e.g, its nonce) as well as all the pending TXs from that account. Obviously, such a change is good for dapp developers since it simplifies the jobs of writing code for interacting with blockchain and handling failures. 

It is also significant for enterprise users. Imaging you are a factory owner and want to register your products on blockchain so that they can later be traced or even verified across different parties (e.g., a logistics company or a retail company). Since products are constantly produced and registered (via TXs), it will be disastrous for your business if a single failed TX causes all TXs sent after it to become unacceptable. The way a TX is identified (by its TXID) on VeChainThor, however, makes the above process smooth and manageable since all you need to do is to make sure the TXIDs have not been used before sending the TXs and then handle the individual failed ones if needed. 

## A little demonstration
Here, I want to do a little bit of coding to demonstrate that TXxIDs are   computed in the way I just described. What I want to do is to go to a VeChainThor explorer such as [veforge](https://explore.veforge.com/) to grab a random TX, locally re-construct the TX, compute its TXID and check whether it equals the one published on the explorer. 

So, I randomly picked a TX with TXID

`0x38fac25309ab5395f1725284af46af585988983945e67b77cf9916b1ddf13d4b`

Since veforge rounded up the value transferred, I would need to find out the exact amount to reconstruct the TX. To do that, I opened [Sync](https://electronjs.org/apps/vechain-sync), a comprehensive dapp environment built for VeChainThor, and click "Toggle Developer Tools" and "Console" as illustrated below.

<p></p>

<center><img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/961c7b4b-7d57-4c92-a467-2d8df4e3a7b3.jpeg" width=60%></center>

<p></p>

<center><img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/5a275e81-1b5f-47a7-ab2e-5e4b26c8d0bd.png" width=60%></center>

<p></p>

I then typed in the console 

`connex.thor.transaction('0x38fac25309ab5395f1725284af46af585988983945e67b77cf9916b1ddf13d4b').get().then(tx => console.log(tx))`

which gave me all the details as shown below.

<p></p>

<center><img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/dfc7528e-2e2a-4295-8e26-39198474a883.png" width=60%></center> 

<p></p>

To reconstruct the TX, you need to install [thor-devkit.js](https://github.com/vechain/thor-devkit.js), a Typescript library to aid dapp development on VeChainThor. My code can be found [here](https://github.com/zzGHzz/ThorDemo1). 

The first step is to build the clauses 

```typescript
let clauses =  [{
    to: '0x564B08C9e249B563903E06D461824b5d6b7F2968',
    value: "0x2a7ee2750fca8ea00000",
    data: '0x'
}]
```
and the TX body with fields set to the values shown in the console

```typescript
let body: Transaction.Body = {
    chainTag: 74,
    blockRef: '0x002e3040a9ade438',
    expiration: 720,
    clauses: clauses,
    gasPriceCoef: 0,
    gas: 21000,
    dependsOn: null,
    nonce: '0x73541be64e72817c'
}   
```
After that, we build the TX via
```typescript
let tx = new Transaction(body)
```
The TXID is then computed by two steps: 1) compute the hash of the RLP encoded TX data without the signature
```typescript
let signingHash = cry.blake2b256(tx.encode())
```
and 2) computes the TXID as the hash of the above computed hash concatenated by the sender's account address
```typescript
const origin = '0xa4d2050f24ed7EfF313B7E912D6e5BF96ce57B95'
let id = '0x' + cry.blake2b256(signingHash, Buffer.from(origin.slice(2), 'hex')).toString('hex')
```
We now can print out the TXID and compare it with the one published on the veforge explorer.


