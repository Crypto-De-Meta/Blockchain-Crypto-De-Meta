# Build Blockchain for Crypto De Meta

In this project, you will watch the Crypto De Meta token create the blockchain network and turn it into a coin, some lines have explanation lines and bare codes for everyone to understand. All developments are made by Aegirsoft company.

## Step 1: A chain of blocks?


```javascript
class Block {
  constructor(blockchain, parentHash, nonce = sha256(new Date().getTime().toString()).toString()) {
    this.blockchain = blockchain;
    this.nonce = nonce;
    this.parentHash = parentHash;
    this.hash = sha256(this.nonce + this.parentHash).toString()
  }
```




## Step 2: So what is THE blockchain?


```javascript
class Blockchain {
  longestChain() {
    const blocks = values(this.blocks)
    const maxByHeight = maxBy(prop('height'))
    const maxHeightBlock = reduce(maxByHeight, blocks[0], blocks)
    const getParent = (x) => {
      if (x === undefined) {
        return false
      }

      return [x, this.blocks[x.parentHash]]
    }
    return reverse(unfold(getParent, maxHeightBlock))
  }
}
```

## Step 3: Not a free-for-all


```javascript
class Block {
  isValid() {
    return this.parentHash === 'root' ||
      (this.hash.substr(-DIFFICULTY) === "0".repeat(DIFFICULTY) &&
      this.hash === sha256(this.nonce + this.parentHash).toString())
  }

  setNonce(nonce) {
    this.nonce = nonce
    this._setHash()
  }

  _setHash() {
    this.hash = sha256(this.nonce + this.parentHash).toString()
  }
}
```


## Step 4: What do I mine?


```javascript
class Block {
  isValid() {
    return this.parentHash === 'root' ||
      (this.hash.substr(-DIFFICULTY) === "0".repeat(DIFFICULTY) &&
      this.hash === this._calculateHash())
  }

  createChild(coinbaseBeneficiary) {
    return new Block({
      blockchain: this.blockchain,
      parentHash: this.hash,
      height: this.height + 1,
      coinbaseBeneficiary
    })
  }

  _calculateHash() {
    return sha256(this.nonce + this.parentHash + this.coinbaseBeneficiary).toString()
  }
}
```


```javascript
class UTXOPool {
  constructor(utxos = {}) {
    this.utxos = utxos
  }

  addUTXO(publicKey, amount) {
    if (this.utxos[publicKey]) {
      this.utxos[publicKey].amount += amount
    } else {
      const utxo = new UTXO(publicKey, amount)
      this.utxos[publicKey] = utxo
    }
  }

  clone() {
    return new UTXOPool(clone(this.utxos))
  }
}

class Blockchain {
  _addBlock(block) {
    if (!block.isValid())
      return
    if (this.containsBlock(block))
      return

    // check that the parent is actually existent and the advertised height is correct
    const parent = this.blocks[block.parentHash];
    if (parent === undefined && parent.height + 1 !== block.height )
      return

    // Add coinbase coin to the pool of the parent
    const newUtxoPool = parent.utxoPool.clone();
    newUtxoPool.addUTXO(block.coinbaseBeneficiary, 12.5)
    block.utxoPool = newUtxoPool;

    this.blocks[block.hash] = block;
    rerender()
  }
}
```


### Step 5: You get a coin! You get a coin!

```javascript
class Transaction {
  constructor(inputPublicKey, outputPublicKey, amount) {
    this.inputPublicKey = inputPublicKey
    this.outputPublicKey = outputPublicKey
    this.amount = amount
    this._setHash()
  }

  _setHash() {
    this.hash = this._calculateHash()
  }

  _calculateHash() {
    return sha256(this.inputPublicKey + this.outputPublicKey + this.amount).toString()
  }
}
```


```javascript
class UTXOPool {
  handleTransaction(transaction) {
    if (!this.isValidTransaction(transaction.inputPublicKey, transaction.amount))
      return
    const inputUTXO = this.utxos[transaction.inputPublicKey];
    inputUTXO.amount -= transaction.amount
    if (inputUTXO.amount === 0)
      delete this.utxos[transaction.inputPublicKey]
    this.addUTXO(transaction.outputPublicKey, transaction.amount)
  }

  isValidTransaction(inputPublicKey, amount) {
    const utxo = this.utxos[inputPublicKey]
    return utxo !== undefined && utxo.amount >= amount && amount > 0
  }
}
```


```javascript
class Blockchain {
  _addBlock(block) {
    // ...
    const newUtxoPool = parent.utxoPool.clone();
    block.utxoPool = newUtxoPool;

    // Add coinbase coin to the pool
    block.utxoPool.addUTXO(block.coinbaseBeneficiary, 12.5)

    // Reapply transactions to validate them
    const transactions = block.transactions
    block.transactions = {}
    let containsInvalidTransactions = false;

    Object.values(transactions).forEach(transaction => {
      if (block.isValidTransaction(transaction.inputPublicKey, transaction.amount)) {
        block.addTransaction(transaction.inputPublicKey, transaction.outputPublicKey, transaction.amount)
      } else {
        containsInvalidTransactions = true
      }
    })

    // If we found any invalid transactions, dont add the block
    if (containsInvalidTransactions)
      return
    // ...
  }
}
```


```javascript
class Blockchain {
  constructor() {
    // ...
    subscribeTo("TRANSACTION_BROADCAST", ({ transaction, blockchainName }) => {
      if (blockchainName === this.name) {
        this.pendingTransactions[transaction.hash] = new Transaction(
          transaction.inputPublicKey,
          transaction.outputPublicKey,
          transaction.amount
        );
      }
    });
  }
}
```


```javascript
class Block {
  addTransaction(inputPublicKey, outputPublicKey, amount, fee) {
    if (!this.isValidTransaction(inputPublicKey, amount, fee))
      return
    const transaction = new Transaction(inputPublicKey, outputPublicKey, amount, fee)
    this.transactions[transaction.hash] = transaction
    this.utxoPool.handleTransaction(transaction, this.coinbaseBeneficiary)
    this._setHash();
  }
}

class UTXOPool {
  handleTransaction(transaction, feeReceiver) {
    if (!this.isValidTransaction(transaction.inputPublicKey, transaction.amount, transaction.fee))
      return
    const inputUTXO = this.utxos[transaction.inputPublicKey];
    inputUTXO.amount -= transaction.amount
    inputUTXO.amount -= transaction.fee
    if (inputUTXO.amount === 0)
      delete this.utxos[transaction.inputPublicKey]
    this.addUTXO(transaction.outputPublicKey, transaction.amount)
    this.addUTXO(feeReceiver, transaction.fee)
  }
}
```

# Step 8: Don't touch my money

```javascript
class Transaction {
  constructor(inputPublicKey, outputPublicKey, amount, fee, signature) {
    this.inputPublicKey = inputPublicKey;
    this.outputPublicKey = outputPublicKey;
    this.amount = amount;
    this.fee = fee;
    this.signature = signature;
    this._setHash();
  }

  hasValidSignature() {
    return (
      this.signature !== undefined &&
      verifySignature(this.hash, this.signature, this.inputPublicKey)
    );
  }
}

class Block {
  isValidTransaction(transaction) {
    return (
      this.utxoPool.isValidTransaction(transaction) &&
      transaction.hasValidSignature()
    );
  }
}
```

## To run

This project is based upon `create-react-app` so a simple `yarn start` will pretty much start everything. You'll only need to start a simple `socket.io` server with `node src/server.js`. 
