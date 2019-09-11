# What you might not know about VeChainThor yet (Part IV) - Mining gas price

*This is the fourth article of the "What you might not know about VeChainThor yet" series. You can find the link of the previous article at the end.* 

VeChainThor allows a transaction (TX) sender to mine extra gas price for his TX. In particular, the sender conducts some computation locally to search for the best [TX nonce](https://doc.vechainworld.io/docs/transaction-model) that maximizes some predefined proof of work (PoW) within a limited amount of time before he sends the TX off to the network. 

When processing the TX, the system converts any valid PoW into extra gas price for calculating the reward for packing the TX into a new block for the block generator. In this way, the TX sender can increase the priority of the TX within the TX pool at the cost of some local computation. 

## How to Mine?
Let *n* and *g* represent the values of TX fields `Nonce` and `Gas`, respectively. We use *b* to denote the number of the block indexed by TX field `BlockRef` and *h* the number of the block that includes the TX. Let &Omega; denote the TX without fields `Nonce` and  `Signature`, *S* the TX sender's account address, *P* the [base gas price](https://doc.vechainworld.io/docs/vechainthor-docs), *H* the hash function and *E* the RLP encoding function.

The PoW, *w*, is defined as:
<p></p>
<center>
<img src="https://latex.codecogs.com/png.latex?w=\min\left(2^{64}-1,\frac{2^{256}-1}{H\Big(H\big(E(\Omega\,\vert\vert\,S)\big)\,\big\vert\big\vert\,n\Big)}\right)" title="\min\left(2^{64}-1,\frac{2^{256}-1}{H\Big(H\big(E(\Omega\,\vert\vert\,S)\big)\,\big\vert\big\vert\,n\Big)}\right)" />
</center>
<p></p>
The extra gas price, &Delta;<i>P</i>, is computed as:
<p></p>
<center>
<img src="https://latex.codecogs.com/png.latex?\Delta P=P\left(\frac{1}{g}\min
\left[g,\frac{w}{10^3}\left(\frac{1}{1.04}\right)^{\frac{h-1}{360\times 24\times 30}}\right]\right)" />
</center>
<p></p>
with the following constraint
<p></p>
<center>
<img src="https://latex.codecogs.com/png.latex?\vert h-b\vert\leq 30" />
</center>
<p></p>
The VTHO reward <i>r</i> for packing the TX into a new block is computed as:
<p></p>
<center>
<img src="https://latex.codecogs.com/png.latex?r=\frac{3}{10}\,g^*\Big(P(1+\phi)+\Delta P\Big)" />
</center>
<p></p>
where <i>&phi;</i> &isin; [0, 1] is the gas price coefficient and <i>g</i><sup>*</sup> the actual amount of gas used for executing the TX.

From the above equations, we know that 

1. Since *b* is a valid block number, `BlockRef` must refer to an existing block, that is, its value must equal the first four bytes of an existing block ID;
2. The TX must be packed into a block within the period of 30 blocks after block *b*, or otherwise, the PoW would not be recognized by the system;
3. The extra gas price &Delta;*P* can not be greater than base gas price *P*.

## Demo
I have made a [demo](https://github.com/zzGHzz/ThorDemo4) that implements the relevant functions for mining gas price. It also provides a simple example of mining gas price for a sample TX. 

In the demo code, function `POW.encode` does the RLP encoding, that is, 
<p></p>
<center>
<img src="https://latex.codecogs.com/png.latex?E\left(\Omega\,\vert\vert\,S\right)" />
</center>
<p></p>

Function `POW.evalWork` computes the PoW, *w*, for a given nonce while functions `POW.workToGas` and `POW.minedGasPrice` compute the extra gas price, &Delta;*P* defined by the above second equation.

### Example
The provided example does the following things: 

1. Connect to VeChain test net and obtain a Connex instance within a NodeJs environment.

   ```
   const net = new SimpleNet("https://sync-testnet.vechain.org");
   const driver = await Driver.connect(net);
   const connex = new Framework(driver);
   ```

2. Get the genesis and latest block, and compute TX fields `ChainTag`, the last byte of the genesis block ID, and `BlockRef`, the first four bytes of the ID of the latest block.

    ```
    body.chainTag = parseInt(connex.thor.genesis.id.slice(-2), 16); 
    body.blockRef = lastBlock.id.slice(0, 18);
    ```
	
3. Call function `mine` to search for the best `Nonce` value that maximizes the PoW in the next 100-second period.

    ```
    const duration = 100;
    const rlp = pow.encode(body, origin);
    body.nonce = safeToHexString(mine(rlp, duration));
    ```
	
4. Prepare and send the TX.

5. Get the TX receipt and show results.

### Results
The following are the results I gained from running the code.

```
PoW = 15081760
nonce = 5438147356160918000
Number of rounds = 4900000
Duration (sec) = 101.459
```

The first part of the output was from function `mine`. It shows the largest PoW found and the corresponding `Nonce` value. The program then printed out the TXID:

```
TXID = 0x46bf31e8df3dfd5a31ad38cf53a3cf93b285eb0ff517c2b19d9ad133416f19bf
```

It then computed the mined gas price &Delta;*P* based the equation for estimating reward *r* and printed out its value:

```
Mined gas price computed from TX receipt = 4.3129e+14 
```

Finally, the program computed the mined gas price &Delta;*P* based on the equation for estimating &Delta;*P* and obtained the same result as shown below:

```
Mined gas price computed from TX data = 4.3129e+14
```

To verify my results, you can set 

```
const txid = '0x46bf31e8df3dfd5a31ad38cf53a3cf93b285eb0ff517c2b19d9ad133416f19bf';
body.blockRef = '0x003626bd3a756fc1';
body.nonce = '0x4b782e35383381f0';
```

and run the part of the code that computes the results.

## Previous Articles
[What you might not know about VeChainThor yet (Part I) - Transaction Uniqueness](https://medium.com/vechain-foundation/what-you-might-not-know-about-vechainthor-yet-part-i-transaction-uniqueness-7a90146f2ace)

[What you might not know about VeChainThor yet (Part II) - Forcible Transaction Dependency](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-ii-forcible-transaction-dependency-ac3e98c4c955?source=friends_link&sk=e2eccfaf216c450caa0f0261504fe99e)

[What you might not know about VeChainThor yet (Part III) - Transaction Fee Delegation (VIP-191)](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-iii-transaction-fee-delegation-vip-191-4ee71d690f1b)

<script type="text/javascript" async
src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js? 
config=TeX-MML-AM_CHTML"
</script>
