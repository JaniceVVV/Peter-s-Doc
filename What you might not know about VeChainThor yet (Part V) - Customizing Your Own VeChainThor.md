# What you might not know about VeChainThor yet (Part V) - Customizing Your Own VeChainThor

*This is the fifth article of the "What you might not know about VeChainThor yet" series. You can find the link of the previous article at the end.*

[VeChainThor](https://github.com/vechain/thor) allows users to customize and deploy their own blockchain networks. The rest of this article consists of two parts. The first part focuses on customization while the second part provides an example of deploying a customized VeChainThor.

## Customization
You can customize VeChainThor through configuring a JSON file and passing it to the Thor client when you launch a node. The configuration file tells the system your settings of global parameters, authority masternodes and on-chain governance, how you would like to mint and allocate VET and VTHO tokens, and any contract(s) you want to deploy from the beginning. These choices result in the creation of the genesis block that is unique to your customized VeChainThor. 

A sample configuration file is shipped with the source code and can be found at `<THOR_DIR>/genesis/example.json`. The following are the values you can tweak in the file:

Name | Value Type | Description
--- | --- | ---
`launchTime` | Unix timestamp (in second) | Earliest launch time of your customized blockchain.
`gasLimit`| Unsigned integer | Gas limit of the genesis block.
`extraData` | String (length &le; 28) | Any words you want to put in the genesis block.
`accounts` | `account[]` | Account(s) to be created for minting and allocating tokens or deploying contract(s). 
`account.address` | 20-byte hex-string | Account address.
`account.balance` | Unsigned integer | VET balance in Wei.
`account.energy`  | Unsigned integer | VTHO balance in Wei (optional).
`account.code` | hex-string | EVM bytecode of a contract account (optional).
`account.storage` | key-value pairs with each key and value being a 32-byte hex-string | Storage values of a contract account (optional).
`authority` | `masternode[]` | List of authority masternodes.
`masternode.masterAddress` | 20-byte hex-string | Master account address.
`masternode.endorsorAddress` | 20-byte hex-string | Address of the account that holds the required VET deposit.
`masternode.identity` | 32-byte hex-string | Masternode identity information.
`params.rewardRatio` | Unsigned integer &in; [0, 1e+18] | VTHO reward ratio = `rewardRatio` / 1e18. 
`params.baseGasPrice` | Unsigned integer | [Base gas price](https://doc.vechainworld.io/docs/vechainthor-docs) value.
`params.proposerEndorsement` | Unsigned integer ( > 0 ) | Minimum VET deposit for each masternode.
`params.executorAddress` | 20-byte hex-string | Address of the [executor](https://doc.vechainworld.io/docs/built-in-smart-contracts) contract account.  
`executor.approvers` | `approver[]` | Accounts authorized to approve on-chain governance actions. For instance, the approvers defined for the mainnet represent the steering committee members of the VeChain Foundation.
`approver.address` | 20-byte hex-string | Approver account address.
`approver.identity` | 32-byte hex-string | Approver identity information.

My demo code for generating a valid JSON file can be found [here](https://github.com/zzGHzz/ThorDemo5).

## Local Deployment
After generating the JSON file, you are ready to deploy your own customized VeChainThor. You need to have at least two nodes up and syncing with each other. Moreover, at least one of the nodes has to be a masternode registered in the JSON file.

If you are to run nodes on machines with a public IP address, you just need to start the Thor client by the command

```
bin/thor --network < JSON_FILE >
```

If it was not the case, you would need to use other options of the `thor` command for deployment. As an example, I will show you step by step how to deploy a customized network that consists of two nodes running locally on a single machine.

**Notations**: 

* `N1` and `N2`: node names.
* `< CONFIG_DIR1 >` and `< CONFIG_DIR2 >`: directories for storing the master account private keys for `N1` and `N2`.
* `< DATA_DIR1 >` and `< DATA_DIR2 >`: directories for storing ledger data for `N1` and `N2`.
* `< ENODE1 >`: substring of the node ID of `N1` that begins with `"enode://"` and ends right before `"@[extip]"`.

### Step 1
Let `N1` be a masternode. We first obtain its master account address via:

```
thor master-key --config-dir < CONFIG_DIR1 >
```

Note that the master account address would be different if you used different `--config-dir` values.
<br>
<img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/7faf5ab2-60b2-43a4-b4a0-1b6f351fdf9b.png" width=70%> 
<br>
After that, we need to register the master node in the JSON file. In my [demo code](https://github.com/zzGHzz/ThorDemo5/blob/master/index.ts), it is done by:

```
const masterAddress1 = '0x929710d206f0e1133f353553353de5bc80c8460b';
...
const endorsor = '0x5e4abda5cced44f70c9d2e1be4fda08c4291945b';
const _authority: Authority[] = [
    { masterAddress: masterAddress1, endorsorAddress: endorsor, identity: strToHexStr('id1', 64) },
    ...
];
``` 

We also need to make sure that we allocate sufficient fund to the `endorsor` account to meet the masternode deposit requirement (defined by `params.proposerEndorsement`). The following code creates the account with the required amount of VET tokens.

```
const _accounts: Account[] = [
    {
        address: endorsor,
        balance: 25000000000000000000000000,
        ...
    },
    ...
];
```

Note that besides the VET balance, I also set those optional values for the `endorsor` account. It is purely for the purpose of demonstrating that you can create a contract account in the JSON file.

### Step 2 
Launch `N1` by the following command:

```
thor --network < JSON_FILE > \
     --config-dir < CONFIG_DIR1 > \
     --data-dir < DATA_DIR1 >
```

where we use options `--config-dir` and `--data-dir` to designate directories for storing keys and ledger data, respectively.
<br>
<img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/acd9dacf-2d7c-4c7a-af6a-206049d86735.png" width=70%> 
<br>
From the above information, we get `< ENODE1 > = enode://5e61...67fb`.

### Step 3
Since `N1` and `N2` are running on the same machine and `N1` has used the default API portal (http://localhost:8669) and P2P port number (11235), we need to set new values for them via options `--api-port` and `--p2p-port` when launching `N2`. The following is the command for launching `N2`:

```
thor --network < JSON_FILE > \
     --config-dir < CONFIG_DIR2 > \
     --data-dir < DATA_DIR2 > \
     --api-addr localhost:8670 \
     --p2p-port 11236 \
     --bootnode < ENODE1 >@127.0.0.1:11235
```

Option `--bootnode` passes information of the boot node from which the current node can sync with others within the same blockchain network. The screenshots below showed the information output for `N1` and then `N2`. It can be seen that they were synced with each other and new blocks generated to extend the chain.
<br>
<img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/8aa1f84d-040e-429d-9a53-fc7bdb537041.png" width=70%> 
<br>
<img src="http://bbs-prd.oss-cn-hongkong.aliyuncs.com/4cc4ac15-7091-4c08-b810-b4b55a298a42.png" width=70%> 
<br>
### Testing
Now, the customized VeChainThor is up and running. We can use tools such as [Connex REPL](https://github.com/vechain/connex-repl) to interact with the blockchain using [Connex](https://connex.vecha.in/#/) interface.

After installing Connex REPL, type the following command to connect to `N2`:

```
connex http://127.0.0.1:8670
```

To check the genesis block, you can type

```
thor.genesis
``` 

To get the balance and code of the `endorsor` account, you can type

```
await thor.account('< ENDORSOR_ADDR >').get()
await thor.account('< ENDORSOR_ADDR >').getCode()
```

You can then compare the output values with those defined in the JSON file for the validation purpose.

## Previous Articles
[What you might not know about VeChainThor yet (Part I) - Transaction Uniqueness](https://medium.com/vechain-foundation/what-you-might-not-know-about-vechainthor-yet-part-i-transaction-uniqueness-7a90146f2ace)

[What you might not know about VeChainThor yet (Part II) - Forcible Transaction Dependency](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-ii-forcible-transaction-dependency-ac3e98c4c955?source=friends_link&sk=e2eccfaf216c450caa0f0261504fe99e)

[What you might not know about VeChainThor yet (Part III) - Transaction Fee Delegation (VIP-191)](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-iii-transaction-fee-delegation-vip-191-4ee71d690f1b)

[What you might not know about VeChainThor yet (Part IV) — Mining gas price](https://medium.com/@ziheng.zhou/what-you-might-not-know-about-vechainthor-yet-part-iv-mining-gas-price-882b0177aaf0)
