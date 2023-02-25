# 8.鎖

區塊鏈有兩種鎖定交易：哈希鎖定交易和秘密鎖定交易。


## 8.1 哈希鎖 Hash Lock

哈希鎖定交易可以讓交易在稍後公布。交易會以哈希值的形式存儲在每個節點的部分快取中，直到交易被公布。交易被鎖定，在 API 節點上不進行處理，直到它被所有共同簽署者簽署。它不會鎖定帳戶所擁有的代幣，但交易發起者需要支付10 XYM的押金。當哈希鎖定交易完全簽署時，被鎖定的資金將退還給發起交易的帳戶。哈希鎖定交易的最大有效期為約48小時，如果交易在此期限內未完成，則10 XYM押金將會丟失。


### 創建聚合綁定交易。

```js
bob = sym.Account.generateNewAccount(networkType);

tx1 = sym.TransferTransaction.create(
    undefined,
    bob.address,  //Send to Bob
    [ //1XYM
      new sym.Mosaic(
        new sym.NamespaceId("symbol.xym"),
        sym.UInt64.fromUint(1000000)
      )
    ],
    sym.EmptyMessage, //mptyMessage
    networkType
);

tx2 = sym.TransferTransaction.create(
    undefined,
    alice.address,  //Send to Alice
    [],
    sym.PlainMessage.create('thank you!'), //Message
    networkType
);

aggregateArray = [
    tx1.toAggregate(alice.publicAccount), //Sent from Alice
    tx2.toAggregate(bob.publicAccount), //Sent from  Bob
]

//Aggregate Bonded Transaction
aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
    aggregateArray,
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

//Signature
signedAggregateTx = alice.sign(aggregateTx, generationHash);
```

當兩筆交易 tx1 和 tx2 排列在 AggregateArray 中時，指定發送方賬戶的公鑰。 參考賬戶章節通過API提前獲取公鑰。 在區塊批准期間，按此順序驗證排列的交易的完整性。

例如，可以在交易1中從 Alice 發送一個 NFT 給 Bob，然後在交易2中從 Bob 發送給 Carol，但如果更改彙總交易的順序為交易2、交易1，將導致錯誤。此外，如果彙總交易中有任何不一致的交易，整個彙總交易將失敗，並且不會被批准進入區塊鏈。

### 哈希鎖交易的創建、簽署和公告
```js
//Creation of Hash Lock TX
hashLockTx = sym.HashLockTransaction.create(
  sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(new sym.NamespaceId("symbol.xym"),sym.UInt64.fromUint(10 * 1000000)), //10xym by default
    sym.UInt64.fromUint(480), // Lock expiry date
    signedAggregateTx,// Register this hash value
    networkType
).setMaxFee(100);

//Signature
signedLockTx = alice.sign(hashLockTx, generationHash);

//Announcing Hash Lock TX
await txRepo.announce(signedLockTx).toPromise();
```

### 聚合綁定交易的公告。

與例如檢查後 Explorer，向網絡宣布保稅交易。
```js
await txRepo.announceAggregateBonded(signedAggregateTx).toPromise();
```


### 聯署
從指定賬戶 (Bob) 共同簽署鎖定的交易。

```js
txInfo = await txRepo.getTransaction(signedAggregateTx.hash,sym.TransactionGroup.Partial).toPromise();
cosignatureTx = sym.CosignatureTransaction.create(txInfo);
signedCosTx = bob.signCosignatureTransaction(cosignatureTx);
await txRepo.announceAggregateBondedCosignature(signedCosTx).toPromise();
```

### Note
Hash Lock Transactions can be created and announced by anyone, not just the account that initially creates and signs the transaction. But make sure that the Aggregate Transaction includes a transaction for whom the account is the signer. Dummy transactions with no mosaic transmission and no message are valid.


## 8.2 Secret Lock・Secret Proof

The secret lock creates a common password in advance and locks the designated mosaic. This allows the recipient to receive the locked mosaic if they can prove that they possess the password before the lock expiry date.

This section describes how Alice locks 1XYM and Bob unlocks the transaction to receive the funds.

First, create a Bob account to interact with Alice.
Bob needs to announce the transaction to unlock the transaction, so please request 10XYM from the faucet.

```js
bob = sym.Account.generateNewAccount(networkType);
console.log(bob.address);

//FAUCET URL outlet
console.log("https://testnet.symbol.tools/?recipient=" + bob.address.plain() +"&amount=10");
```

### Secret Lock

Create a common pass for locking and unlocking.

```js
sha3_256 = require('/node_modules/js-sha3').sha3_256;

random = sym.Crypto.randomBytes(20);
hash = sha3_256.create();
secret = hash.update(random).hex(); //Lock keyword
proof = random.toString('hex'); //Unlock keyword
console.log("secret:" + secret);
console.log("proof:" + proof);
```

###### Sample output
```js
> secret:f260bfb53478f163ee61ee3e5fb7cfcaf7f0b663bc9dd4c537b958d4ce00e240
  proof:7944496ac0f572173c2549baf9ac18f893aab6d0
```

Creating, signing and announcing transaction
```js
lockTx = sym.SecretLockTransaction.create(
    sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(
      new sym.NamespaceId("symbol.xym"),
      sym.UInt64.fromUint(1000000) //1XYM
    ), //Mosaic to lock
    sym.UInt64.fromUint(480), //Locking period (number of blocks)
    sym.LockHashAlgorithm.Op_Sha3_256, //Algorithm used for lock keyword generation
    secret, //Lock keyword
    bob.address, //Forwarding address to unlock:Bob
    networkType
).setMaxFee(100);

signedLockTx = alice.sign(lockTx,generationHash);
await txRepo.announce(signedLockTx).toPromise();
```

The LockHashAlgorithm is as follows
```js
{0: 'Op_Sha3_256', 1: 'Op_Hash_160', 2: 'Op_Hash_256'}
```

At the time of locking, the unlock destination is specified by Bob, thus the destination account (Bob) cannot be changed even if an account other than Bob unlocks the transaction.

The maximum lock period is 365 days (counting number of blocks in days).

Check the approved transactions.
```js
slRepo = repo.createSecretLockRepository();
res = await slRepo.search({secret:secret}).toPromise();
console.log(res.data[0]);
```
###### Sample output
```js
> SecretLockInfo
    amount: UInt64 {lower: 1000000, higher: 0}
    compositeHash: "770F65CB0CC0CA17370DE961B2AA5B48B8D86D6DB422171AB00DF34D19DEE2F1"
    endHeight: UInt64 {lower: 323495, higher: 0}
    hashAlgorithm: 0
    mosaicId: MosaicId {id: Id}
    ownerAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    recordId: "6260A1D3205E94BEA3D9E3E9"
    secret: "F260BFB53478F163EE61EE3E5FB7CFCAF7F0B663BC9DD4C537B958D4CE00E240"
    status: 0
    version: 1
```
It shows that Alice who locked the transaction is recorded in ownerAddress and the Bob is recorded in recipientAddress.
The information about the secret is published and Bob informs the network of the corresponding proof.


### Secret Proof

Unlock the transaction using the secret proof. Bob must have obtained the secret proof in advance.


```js
proofTx = sym.SecretProofTransaction.create(
    sym.Deadline.create(epochAdjustment),
    sym.LockHashAlgorithm.Op_Sha3_256, //Algorithm used for lock keyword generation
    secret, //Lock keyword
    bob.address, //Deactivated accounts (receiving accounts)
    proof, //Unlock keyword
    networkType
).setMaxFee(100);

signedProofTx = bob.sign(proofTx,generationHash);
await txRepo.announce(signedProofTx).toPromise();
```

Confirm the approval result.
```js
txInfo = await txRepo.getTransaction(signedProofTx.hash,sym.TransactionGroup.Confirmed).toPromise();
console.log(txInfo);
```
###### Sample output
```js
> SecretProofTransaction
  > deadline: Deadline {adjustedValue: 12669305546}
    hashAlgorithm: 0
    maxFee: UInt64 {lower: 20700, higher: 0}
    networkType: 152
    payloadSize: 207
    proof: "A6431E74005585779AD5343E2AC5E9DC4FB1C69E"
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    secret: "4C116F32D986371D6BCC44CE64C970B6567686E79850E4A4112AF869580B7C3C"
    signature: "951F440860E8F24F6F3AB8EC670A3D448B12D75AB954012D9DB70030E31DA00B965003D88B7B94381761234D5A66BE989B5A8009BB234716CA3E5847C33F7005"
    signer: PublicAccount {publicKey: '9DC9AE081DF2E76554084DFBCCF2BC992042AA81E8893F26F8504FCED3692CFB', address: Address}
  > transactionInfo: TransactionInfo
        hash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
        height: UInt64 {lower: 323805, higher: 0}
        id: "6260CC7F60EE2B0EA10CCEDA"
        merkleComponentHash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
    type: 16978
```

The SecretProofTransaction does not contain information about the amount of any mosaics received. Check the amount in the receipt created when the block is generated. Search for receipts addressed to Bob with receipt type:LockHash_Completed.


```js
receiptRepo = repo.createReceiptRepository();

receiptInfo = await receiptRepo.searchReceipts({
    receiptType:sym.ReceiptTypeLockHash_Completed,
    targetAddress:bob.address
}).toPromise();
console.log(receiptInfo.data);
```
###### Sample output
```js
> data: Array(1)
  >  0: TransactionStatement
        height: UInt64 {lower: 323805, higher: 0}
     >  receipts: Array(1)
          > 0: BalanceChangeReceipt
                amount: UInt64 {lower: 1000000, higher: 0}
            > mosaicId: MosaicId
                  id: Id {lower: 760461000, higher: 981735131}
              targetAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
              type: 8786
```

ReceiptTypes are as follows:

```js
{4685: 'Mosaic_Rental_Fee', 4942: 'Namespace_Rental_Fee', 8515: 'Harvest_Fee', 8776: 'LockHash_Completed', 8786: 'LockSecret_Completed', 9032: 'LockHash_Expired', 9042: 'LockSecret_Expired', 12616: 'LockHash_Created', 12626: 'LockSecret_Created', 16717: 'Mosaic_Expired', 16718: 'Namespace_Expired', 16974: 'Namespace_Deleted', 20803: 'Inflation', 57667: 'Transaction_Group', 61763: 'Address_Alias_Resolution', 62019: 'Mosaic_Alias_Resolution'}

8786: 'LockSecret_Completed' : LockSecret is completed
9042: 'LockSecret_Expired'　：LockSecret is expired
```

## 8.3 Tips for use


### Paying the transaction fee instead

Generally blockchains require transaction fees for sending transactions. Therefore, users who want to use blockchains need to obtain the native currency of the chain to pay fees (e.g. Symbol's native currency XYM) from the exchange in advance. If the user is a company, the way it is managed might be an issue from an operational point of view. If using Aggregate Transactions, service providers can cover hash lock and transaction fees on behalf of users.

### Scheduled transactions

Secret locks are refunded to the account that created the transaction after a specified number of blocks.
When the service provider charges the cost of the lock for the Secret Lock account, the amount of tokens owned by the user for the lock will increase after the expiry date has passed. On the other hand, announcing a secret proof transaction before the deadline has passed is treated as a cancellation as the transaction is completed and the funds are returned to the service provider.

### Atomic swaps
Secret locks can be used to exchange  mosaics (tokens) with other chains. Please note that other chains refer to this as a hash time lock contract (HTLC)  not to be mistaken for a Symbol Hash Lock.
