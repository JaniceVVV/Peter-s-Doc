# What you might not know about VeChainThor yet (Part III) - Transaction Fee Delegation (VIP-191)

*This is the third article of the "What you might not know about VeChainThor yet" series. You can find the links of the previous articles at the end.* 

In this article, I will mainly talk about the "Transaction (TX) Fee Delegation" mechanism and particularly, the newly implemented "Designated Gas Payer" protocol on VeChainThor. 

Put simply, the TX-fee-delegation mechanism is a mechanism that allows ordinary people to be able to use decentralized applications (dapps) without directly paying the TX fee caused during their interactions with dapps. In this way, users, when using dapps, could have the same kind of experiences when they are using normal mobile or web-based apps nowadays, which is crucial for mass adoption of blockchain technology. 

There are currently two protocols running on VeChainThor that enable such a mechanism: the *Multi-Party Payment* (MPP) protocol and the *Designated Gas Payer* protocol. The former exists as a built-in protocol from day one while the latter is proposed in [VIP-191](https://github.com/vechain/VIPs/blob/master/vips/VIP-191.md) by [Totient Labs](https://www.totientlabs.com/), a major VeChain ecosystem player and contributor. 

In the rest of this article, I will first briefly talk about how MPP works which motivates the invention of VIP-191. I will then discuss how the protocol works about and compare it with MPP. After that, I will focus on the implementation details of VIP-191. At last, a demo will be provided as usual for you to have a more intuitive understanding.

## How MPP works
MPP allows an account to pay fees for TXs sent from some designated account. 

Let's call the account paying the fee `PAYER` and the one sending TXs `USER`. Let's also define the master of `PAYER`, from which the TX fee is actually deducted, as either `PAYER` itself if it is a normal account or the account deploys `PAYER` if it is a contract. Of course, the current master is allowed to transfer the mastership to another account. The figure below illustrates the relationships between `PAYER`, `USER` and the master of `PAYER`. 

</br>
<center><img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/604ce03e-7452-4a26-ae4c-08519396714e.png" width=60%></center> 
</br>

Conceptually, we can think of VeChainThor maintaining two virtual tables (as illustrated below) on chain for MPP. The left table records the payer-user relationships while the right one stores information about how much `PAYER` is willing to pay. I am not going to describe each column in the tables. Detailed descriptions can be found [here](https://doc.vechainworld.io/docs/multi-party-payment-protocol-mpp).

</br>
<center><img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/01e1f25c-5a56-43e9-a32c-abc9e9106a7f.png" width=80%></center> 
</br>

Let's have a look at an example of using MPP to allow `PAYER 0x76C3` to fund TXs from `USER 0x9D6D`. To do that, the master of `0x76C3 ` needs to call functions of the built-in contract `Prototype` to add an entry in each table. The added entry marked in the left red box associates `PAYER 0x76C3` with `USER 0x9D6D`, telling the system that `0x76C3` is willing to pay for TXs from `0x9D6D` while the the added entry marked in the right red box says how much `0x76C3` would like to fund those TXs. After that, the MPP protocol is activated and TXs from `0x9D6D` will be funded by the master of `0x76C3`. 

## Motivation behind VIP-191
It is clear that MPP was designed from the point of view of a dapp owner who controls a bunch of contracts running on chain. It is the sole responsibility of the owner to set up MPP and the protocol can only affect TXs sent to those contracts. 

Moreover, since MPP requires writing data on chain and therefore, causes certain overhead cost, it is more cost-effective to use the protocol for a relatively stable relationship between a user and the dapp, rather than some temporary arrangement. 

VIP-191 is aimed to supplement MPP to provide more flexibility for TX fee delegation on VeChainThor. In particular, it allows a TX sender to seek for an arbitrary party, not necessarily the TX recipient, who is willing to pay for the TX. As stated in article "[VIP191: The Key to Mass Adoption](https://medium.com/@totientlabs/vip191-the-key-to-mass-adoption-1beccd902a78)", the protocol 
> enables a 3rd party, such as a dApp or a wallet, to pay for transaction fees on behalf of the sender, while still allowing the sender to maintain full control over their transaction signing and sending lifecycle.

## How VIP-191 works
The protocol works quite simple actually. It requires both the TX sender and the party who wants to pay for the TX fee to put their digital signatures in the TX. The sender also need to turn on the VIP-191 feature (which will be discussed later) to inform the system that this is a VIP-191 enabled TX. Let's call the party *the Gas-Payer* to be consistent with the naming in the source code. Once the TX is accepted and executed, the TX fee will be deducted from the gas-payer's VeThor balance.

## Comparison with MPP
Compared with MPP, VIP-191 gives control back to TX senders to activate the protocol. Moreover, it does not introduce any overhead cost. However, it requires the TX sender and gas-payer to be both online to complete the TX while MPP does not. In terms of transparency, MPP is the better option since the gas-payer will have to explicitly put his/her intension to fund TXs from a particular account on chain.

## Implementation of VIP-191
The designated-gas-payer protocol or VIP-191 has been implemented in the newly released [VeChainThor v1.1.2](https://github.com/vechain/thor/releases/tag/v1.1.2). At the time I am writing this article, it can be tested on the test net. However, you will have to wait for a few days to use it on the main net. 

There have been two major changes made to implement VIP-191:

1. Extending the TX model, and 
2. Adding extra logic for deciding the actual gas-payer for a VIP-191 enabled TX.

### TX Model Extension
Field `Reserved` in the TX body structure has been re-defined to be of type `reserved` as shown below:

```
type reserved struct {
	Features Features
	Unused   []rlp.RawValue
}
``` 

Within the structure, we define field `Features` as 32-bit unsigned integer. We can think of it as a bit map. Each bit marks the status (1 for on and 0 for off) of a particular feature. For VIP-191, the least significant bit is used.

Recall that VIP-191 requires two valid signatures to be included in the TX. In practice, the TX sender's signature is concatenated by the gas-payer's signature and assigned to field `Signature` as usual. Moreover, the protocol requires the gas-payer to sign the TXID which is a unique identifier of the TX. Details of TXID can be found in my first [article](https://medium.com/vechain-foundation/what-you-might-not-know-about-vechainthor-yet-part-i-transaction-uniqueness-7a90146f2ace).

### Gas-Payer-Deciding Logic
The gas-payer-deciding logic brought by VIP-191 is added in function `BuyGas` in the Go source file `THORDIR/runtime/resolved_tx.go`. I copy-paste the actual code below for your convenience. 

```
if r.Delegator != nil {
	if energy.Sub(*r.Delegator, prepaid) {
		return baseGasPrice, gasPrice, *r.Delegator, func(rgas uint64) { doReturnGas(rgas) }, nil
	}
	return nil, nil, thor.Address{}, nil, errors.New("insufficient energy")
}
```

It can be seen that the system first checks whether there is a designated gas-payer (`r.Delegator`). If there is one, it will try to deduct the initial cost of the TX from the gas-payer's VeThor balance. If the balance is too low, the system will stop processing the TX and return an error. Otherwise, it will mark the gas-payer in the runtime context associated with the TX and pass on the context to the code that executes individual clauses. 

Listing all changes related to VIP-191 is out of the scope of this article. Interested readers are encouraged to have a look at the [source code](https://github.com/vechain/thor/) for full details.

## Demo
As usual, I created a demo to demonstrate the TX-fee-delegation mechanism (both VIP-191 and MPP) on VeChainThor. The demo is built using tools [connex-framework](https://github.com/vechain/connex-framework) and [connex.driver-nodejs](https://github.com/vechain/connex.driver-nodejs) that implement the [Connex](https://github.com/vechain/connex) interface in a NodeJS environment. The source code can be downloaded [here](https://github.com/zzGHzz/ThorDemo3). 

### A Walk-Through
There are three accounts used in the demo: 

1. `SENDER` whose address begins with `0xd551`,
2. `DELEGATOR` (or gas-payer) whose address begins with `0xe466`, and 
3. `RECIPIENT` whose address begins with `0x9143`.

To demonstrate VIP-191, I created one TX from `SENDER` to `RECIPIENT`. The TX was co-signed by `SENDER` and `DELEGATOR` to allow TX fee paid by `DELEGATOR`.

To demonstrate MPP, I created two TXs sent from `DELEGATOR` to call functions of the built-in contract `Prototype` to add a payer-user relation and funding plan as described above. I then created one TX from `SENDER` to `DELEGATOR` to demonstrate the effect of MPP. Note that `DELEGATOR` is the master of itself since it is a normal account rather than a contract. The TX can only be sent to `DELEGATOR` or otherwise it wouldn't be funded by `DELEGATOR` according to the rules set by MPP.

### Connex Interface for VIP-191
The Connex interface has been updated to support VIP-191. I want to show you that it is pretty easy to use Connex to construct a VIP-191 enabled TX. 

There are two extra things you need to do on top of the normal procedure of constructing a TX:

1. Create your own function with the following definition:

	`function (unsignedTx: { raw: string, origin: string }): Promise<{ signature: string }>`
	
	This function is typically responsible for send data to the gas-payer, waiting for its response and returns a `Promise`, if resolved, carrying the gas-payer's digital signature on the corresponding TXID.
2. Pass the function to the instance of `Connex.Vendor.TxSigningService`, as you've already created for TX construction, via the `delegate` method. For instance, you may add a line such as:

	`signingService.delegate(MyFunc);`

That's it! 

Note that the function I made in the demo code is NOT a typical function you would see in a real application. It is created purely for this demo and should not be considered as an example of creating such a function. 

### Results
Here I copy-paste the results output by the demo code after I ran it. 

```
--------------------------
VIP-191
--------------------------
TXID       = 0xb58e1d1bf9da3414c24df51926003ebcbac7eb10246dd25548ccfd9202d4276e
From       = 0xd55100eedb61f1e553a38c33a234ce07952c43f2
To         = 0x91436f1E5008B2E6093E114A25842F060012685d
GasPayer   = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
GasUsed    = 21000
...
```

The first part shows information of the VIP-191 enabled TX. It can be seen that the actual gas-payer was indeed `DELEGATOR` rather than `SENDER`.

```
--------------------------
MPP - Add User
--------------------------
TXID       = 0x508f50692c88054c5f8df982937b2871cae3cde002a4bc58e975277acca53c87
From       = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
To         = 0x000000000000000000000050726f746f74797065
GasPayer   = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
GasUsed    = 25074
...
--------------------------
MPP - Add Credit Plan
--------------------------
TXID       = 0x57329507daadb47a8ea68375577f9699fce3f1329df4703451be36d3804aa853
From       = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
To         = 0x000000000000000000000050726f746f74797065
GasPayer   = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
GasUsed    = 44811
...
```
The second part shows information of the two TXs `DELEGATOR` sent to the built-in contract `Prototype` to activate MPP for `SENDER`. The overhead costs for adding a user and a credit plan are 25k and 45k gas, respectively. 

```
--------------------------
MPP - User Sends TX
--------------------------
TXID       = 0x87b68a65105f2cc5746b88e1e0fd5e1cf4f6d2dbf8459bb8ef2599957ffcb655
From       = 0xd55100eedb61f1e553a38c33a234ce07952c43f2
To         = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
GasPayer   = 0xe4660c72dea1d9fc2a0dc2b3a42107d37edc6327
GasUsed    = 21000
```

The last part shows the TX sent from `SENDER` to `DELEGATOR` is indeed paid by `DELEGATOR` under MPP.

## Previous Articles
[What you might not know about VeChainThor yet (Part I) - Transaction Uniqueness](https://medium.com/vechain-foundation/what-you-might-not-know-about-vechainthor-yet-part-i-transaction-uniqueness-7a90146f2ace)

[What you might not know about VeChainThor yet (Part II) - Forcible Transaction Dependency](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-ii-forcible-transaction-dependency-ac3e98c4c955?source=friends_link&sk=e2eccfaf216c450caa0f0261504fe99e)
