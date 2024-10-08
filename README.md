# Introduction to TON Security

TON is Telegram's fully decentralized Layer 1 blockchain for billions of users. It boasts ultra-fast transaction speeds, meager fees, and easy-to-use applications. While there are similarities between Ether and TON in terms of their overall structure and the functionality they provide, their key technical aspects and network design approaches are very different. In this study, we build a proprietary vulnerability repository for TON security and auditing in terms of security auditing and vulnerability protection.

In this study we chose the descriptive research method to start the study, which is mainly due to the specificity of the auditing industry, TON ecology lacks a large number of samples, so we need to understand and validate the existing security vulnerabilities, give a narrative and explain. In this study, we will directionally reveal TON ecology's security risks and popularize smart contract security knowledge in TON.

We will also use empirical research and case study methodology to assist the research, such as verifying the vulnerabilities through empirical testing and documenting them in the vulnerability repository.

## TON Characterization

### 1. Smart contracts need to pay gas fees for internal storage

In blockchain, any data stored on the chain is eternal, and then someone needs to pay for that eternally stored data. In Ether, the contract deployer only needs to pay once when the contract is deployed, while the subsequent cost of storing the data is paid by the miners as an infrastructure cost.

In TON, each smart contract is required to hold a certain amount of TON token balance, and the smart contract's balance is used to pay for storage on the chain. If the balance of a smart contract is not enough to pay for this storage fee, then the contract will enter a frozen state, and it is necessary to transfer several TON tokens that is enough to pay for the storage fee to be able to unfreeze the contract, and if it is not unfrozen after a certain period, then the contract will be deleted.

### 2. Each user's wallet is a smart contract

In Ethereum, the address is divided into an external account address and a contract address, the external account address has its own private key and public key, held by the user, the user through the private key to control the address to send out transactions to the smart contract. In TON, there is no external account address, the wallet held by the user is also a smart contract, and the wallet address is calculated from the wallet contract code and its initialization parameters (including the user's public key), the user needs to send transactions to the wallet contract through the private key, and then forward the transactions to other contracts.

### 3. Calls between smart contracts are asynchronous

In Ethernet, every transaction is atomic, when a smart contract calls a method of a different smart contract, the call will be immediately processed in the same transaction, where an exception in every link will lead to the rollback of the whole transaction, which has given rise to the birth of lightning loans, a rather defi charismatic business.
In TON, calls between smart contracts are completely asynchronous, and calls between contracts may be split across multiple different blocks. By the time a contract receives a message sent by another contract, several blocks may have passed, and the state may have changed multiple times by this time, so potential errors will occur more frequently.

### 4. No synchronized status acquisition between contracts

In Ether, it is very common for contracts to get real-time status from each other, such as checking the token balance or checking the price of a pair of transactions, etc. However, in TON, this is nearly impossible due to the asynchronous message sending. However, in TON, this is nearly impossible due to the asynchronous sending of messages. When a contract receives the state from another contract, the state at that time may not be the same as when the message was sent due to the difference in blocks.

### 5. Smart contracts in TON are all mutable

The code of smart contracts in Ethernet is established as immutable at the very beginning of the design, and can never be modified, of course, the emergence of scalable contracts solves this limitation, but it also makes the operation cumbersome.TON discards the feature of the immutability of smart contracts, and in the func language, the developer can choose to write the contract code as a variable in the contract, and when there is a variable of the code in the contract, the contract is mutable. When there is a code variable in the contract, the contract is mutable, and this design abandons the complexity of scalable contracts.

## Difference between TVM and EVM

### 1. Storage structure

EVM: Uses a 256-bit key to a 256-bit value mapping method, the basic storage unit is a 256-bit integer.

TVM: relies on the "Bags of Cell" architecture, the underlying data is a 1023-bit cell, a cell can refer to four other cells, and each cell can only be referenced once, through the continuous downward tree structure to expand the infinite data structure.

![image-20240813162620479](./README.assets/image-20240813162620479.png)

### 2. Fallback mechanisms

EVM: any inter-contract call in the same transaction is atomic, a fallback in the entire transaction generated by the state changes are all rolled back.
TVM: a fallback will only rollback within the current contract call and send a bounce message to the previous contract, if it receives the bounce message, it can manually rollback the previous state, if it does not receive it, it may lead to problems with the state of the previous and previous contracts.

![image-20240813162638217](./README.assets/image-20240813162638217.png)

### 3. Gas calculations

EVM: The Ethernet Virtual Machine processes each transaction one at a time, based on the instructions identified in the transaction, each of which has an explicitly defined amount of Gas consumed. For example, performing an additional operation consumes 3Gas, so the amount of Gas consumed by a transaction depends entirely on the cumulative Gas of all the instructions in the transaction, and the VM will provide feedback on the total amount of Gas consumed when the transaction is completed.

TVM: Unlike EVM, the execution of a transaction in TVM is divided into multiple phases, each of which generates a corresponding amount of gas, using the official formula for calculating gas as an example:

![image-20240813162647646](./README.assets/image-20240813162647646.png)

storage_fees: Fees paid for storing smart contracts on the blockchain.

In_fwd_fees: Fees for importing messages from outside the blockchain.

computation_fees: Fees for executing code in a virtual machine.

action_fees: Fees charged by the smart contract for sending external messages.

out_fwd_fees: Fees for sending messages from the TON blockchain to external services and external blockchains (currently 0).

TON can change the gas configuration on the chain via a vote on the main net, which may result in a large change in the gas fee.

### 4. Split Chain

EVM: Ethernet completes transactions on a single blockchain, and the scaling solution relies heavily on sidechain and layer2 solutions.

TVM: TON has a main chain and allows the creation of up to 2^32 working chains, each of which can be subdivided into up to 2^60 slices. Currently, only the main chain (workchain_id=-1) and occasional basic workchains (workchain_id=0) run in the TON blockchain. Each smart contract address is preceded by a 32-bit chain id that is used to identify and link to smart contract addresses in different working chains. Below is an example of a raw smart contract address using the workchain ID and account ID (denoted as workchain_id and account_id):

-1:fcb91a3a3816d0f7b8c2c76108b8a9bc5a6b7a55bd79f8ab101c52db29232260

-1 indicates the chain ID belonging to the master chain.

### 5. Data Types 

EVM: supports data structures such as signed/unsigned integers, booleans, addresses, arrays, and custom structures.

TVM: TON natively supports only integers, cells, slices, Continuation, Tensors, and other data structures, func currently does not support customized data structures, tact adds boolean, string, address, and other common data structures for smart contracts based on the above types.

# Vulnerability Type
## GAS Limitation

The gas in the TON chain comes from ctx.value, that is to say, the msg.value on the ETH chain and the gas fee are combined into one. First of all, if you need to recharge the TON native coin TON, you need to subtract the gas value needed for this transaction from this basis, otherwise, it is equivalent to this transaction by the contract to pay for the gas. and since the gas fee is controlled by the user, you can control the gas fee to make the contract in a certain step of the gas fee is not enough so that the problem of backing out occurs. For example:

Tact:

```tact
const ACC_PRECISION: Int = pow(10, 20);
contract masterChef{
    totalsupply: Int as uint64 = 0;
    minichef: Address;
    jettonWallet: Address;
    lastRewardBlock: Int as uint64 = 0 ;
    rewardPerSecond: Int as uint64 = 0;
    accRewardPerShare: Int as uint64 = 0;
    init(minichef: Address, jettonWallet: Address) {
        self.minichef = minichef;
        self.jettonWallet = jettonWallet;
    }
    receive(msg: Deposit) { 
+        require(context().value>= ton("0.05"),"not enough gas"); 
        self.updatePool();
        require(context().sender == self.jettonWallet,"not jettonWallet");
        self.totalsupply += msg.amount;
        msg.toCell().send_to(self.minichef);
    }

    receive(msg: Withdraw) { 
+        require(context().value>= ton("0.05"),"not enough gas"); 
        self.updatePool();
        self.totalsupply -= msg.amount;
        msg.toCell().send_to(self.minichef);
    }

    receive(msg: WithdrawReply) { 
        require(context().sender == self.minichef,"incorrect sender");
        self.sendJetton(msg.amount,msg.sender);
    }
    inline fun updatePool() {
        if (self.totalsupply > 0 ) {
            let reward: Int = (now() - self.lastRewardBlock) * self.rewardPerSecond;
            let rewardAmount: Int = ACC_PRECISION * reward /self.totalsupply;
            self.accRewardPerShare = self.accRewardPerShare + rewardAmount;
        } 
        self.lastRewardBlock = now();
    }
}
contract miniChef{
    owner: Address;
    masterchef: Address;
    amount: Int as coins = 0;
    init(owner: Address,masterchef: Address) {
        self.owner = owner;
        self.masterchef = masterchef;
        
    }
    receive(msg: Deposit) { 
        require(sender() == self.masterchef ,"not masterchef");
        self.amount += msg.amount;
    }

    receive(msg: Withdraw) { 
        require(sender() == self.masterchef ,"not masterchef");
        require(self.amount >= msg.amount,"not enough balance");
        self.amount -= msg.amount;
        WithdrawReply {
            amount: msg.amount,
            sender: msg.sender
        }.toCell().send_to(self.masterchef);
    }
}
```

Func:

```func
;;masterChef
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (int total_supply, slice miniChef,slice jetton_wallet, int last_reward_block, int reward_per_second, int acc_reward_per_share) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer_notification() ){;;Deposit
+        if (msg_value <= common_fee()){
+          send_jetton();
+            return();
+        }
        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73, equal_slices(sender_address, jetton_wallet));
        update_pool();
        total_supply += amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),miniChef,0,initDepositMsg(from_address, amount));
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    if ( op == op::withdraw() ){
+        if (msg_value <= common_fee()){
+          send_jetton();
+            return();
+        }
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();

        update_pool();
        total_supply -= jetton_amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),from_address,0,initWithdrawMsg(amount,sender));
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    if ( op == op::withdraw_reply() ){
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();

        send_jetton(amount,sender);
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    throw(0xffff);
}


;;miniChef
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, slice master_chef, int balance) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        throw_unless(73, equal_slices(sender_address, master_chef));
        slice from_address = in_msg_body~load_msg_addr();
        int jetton_amount = in_msg_body~load_coins();

        balance += jetton_amount;
        save_data(owner,master_chef,balance);
        return();
    }

    if ( op == op::withdraw() ){
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();
        throw_unless(73, equal_slices(sender_address, master_chef));
        throw_unless(500,balance >= amount);

        balance -= amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),master_chef,0,initWithdrawReplyMsg(amount,sender));
        save_data(owner,master_chef,balance);
        return();
    }

    throw(0xffff);
}
```

In the above two contracts, the `masterChef` contract is the entry contract, `miniChef` is the contract owned by each user individually, and the default `Deposit` message has the same opcode as `JettonTransferNotification`, that is, the default user has already been transferred to jetton at this time.

When the wallet of `masterChef` sends a `Deposit` to `masterChef`, the logic in the `masterChef` contract is executed successfully, and the `Deposit `message is sent to `miniChef`, but there is insufficient gas during the execution of the `Deposit` message in `miniChef`, which will lead to the situation that user's jetton has been sent but there is not enough gas in the `miniChef` does not have the corresponding balance of the corresponding user, resulting in the user not being able to withdraw the money, and the sent jetton can not be retrieved, and the `totalSupply` in the `masterChef` contract has been increased, which will also lead to errors in the calculation of the subsequent rewards.

When the user deposits the jetton normally, at this time call the `Withdraw` message but pass in a small gas fee, let the transaction execution in the `miniChef` in the `Withdraw` message with insufficient gas failure, will not cause a loss of the user's deposit, but will lead to the `totalSupply` in the `masterChef` contract to reduce repeated execution can be made `totalSupply` a very small value, resulting in more rewards being issued for the user.

**Risk level: Critical**

## UnBounced Message

Due to the asynchronous nature of the TON chain, it is very easy to have different states in a complex call flow, to solve this situation, TON natively provides a bounce handling mechanism, so that the contract can manually call back to the previously changed state variables. However, the current bounce message can only carry 224 bits of valid data, while an address in TON has 256 bits, the data that can be carried is very limited, which is a big limitation that may be fixed later.

For example:

Tact:

```tact
contract masterChef{
    totalsupply: Int as uint64 = 0;
    minichef: Address;
    jettonWallet: Address;
    lastRewardBlock: Int as uint64 = 0 ;
    rewardPerSecond: Int as uint64 = 0;
    accRewardPerShare: Int as uint64 = 0;
    init(minichef: Address, jettonWallet: Address) {
        self.minichef = minichef;
        self.jettonWallet = jettonWallet;
    }
    receive(msg: Deposit) { 
        self.updatePool();
        require(context().sender == self.jettonWallet,"not jettonWallet");
        self.totalsupply += msg.amount;
        msg.toCell().send_to(self.minichef);
    }

    receive(msg: Withdraw) { 
        self.updatePool();
        self.totalsupply -= msg.amount;
        msg.toCell().send_to(self.minichef);
    }

    receive(msg: WithdrawReply) { 
        require(context().sender == self.minichef,"incorrect sender");
        self.sendJetton(msg.amount,msg.sender);
    }
+    bounced(src: bounced<Withdraw>) {
+        self.totalsupply = self.totalsupply + src.amount;
+   }
+    bounced(src: bounced<Deposit>) {
+        self.totalsupply = self.totalsupply - src.amount;
+    }
    inline fun updatePool() {
        if (self.totalsupply > 0 ) {
            let reward: Int = (now() - self.lastRewardBlock) * self.rewardPerSecond;
            let rewardAmount: Int = ACC_PRECISION * reward /self.totalsupply;
            self.accRewardPerShare = self.accRewardPerShare + rewardAmount;
        } 
        self.lastRewardBlock = now();
    }
}
contract miniChef{
    owner: Address;
    masterchef: Address;
    amount: Int as coins = 0;
    init(owner: Address,masterchef: Address) {
        self.owner = owner;
        self.masterchef = masterchef;
        
    }
    receive(msg: Deposit) { 
        require(sender() == self.masterchef ,"not masterchef");
        self.amount += msg.amount;
    }

    receive(msg: Withdraw) { 
        require(sender() == self.masterchef ,"not masterchef");
        require(self.amount >= msg.amount,"not enough balance");
        self.amount -= msg.amount;
        WithdrawReply {
            amount: msg.amount,
            sender: msg.sender
        }.toCell().send_to(self.masterchef);
    }
+    bounced(src: bounced<WithdrawReply>) {
+        self.amount = self.amount + src.amount;
+    }
}
```

Func:

```Func
;;masterChef
+() on_bounce(slice in_msg_body) impure {
+  in_msg_body~skip_bits(32); ;; 0xFFFFFFFF
+  (int total_supply, slice miniChef,slice jetton_wallet, int last_reward_block, int reward_per_second, int acc_reward_per_share) = load_data();
+  int op = in_msg_body~load_uint(32);
+  throw_unless(709, (op == op::deposit()) | (op == op::withdraw()));
+  int amount = in_msg_body~load_coins();
+  if (op == op::deposit()){
+    total_supply -= amount;
+  }else{
+   total_supply += amount;
+  }
+  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
+}
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (int total_supply, slice miniChef,slice jetton_wallet, int last_reward_block, int reward_per_second, int acc_reward_per_share) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer_notification() ){;;Deposit
        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73, equal_slices(sender_address, jetton_wallet));
        update_pool();
        total_supply += amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),miniChef,0,initDepositMsg(from_address, amount));
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    if ( op == op::withdraw() ){
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();

        update_pool();
        total_supply -= jetton_amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),from_address,0,initWithdrawMsg(amount,sender));
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    if ( op == op::withdraw_reply() ){
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();

        send_jetton(amount,sender);
        save_data(total_supply,miniChef,jetton_wallet,last_reward_block,reward_per_second,acc_reward_per_share);
        return();
    }

    throw(0xffff);
}


;;miniChef
+() on_bounce(slice in_msg_body) impure {
+  in_msg_body~skip_bits(32); ;; 0xFFFFFFFF
+  (slice owner, slice master_chef, int balance) = load_data();
+  int op = in_msg_body~load_uint(32);
+  throw_unless(709, op == op::deposit());
+  int amount = in_msg_body~load_coins();
+  balance += amount;
+  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
+}
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, slice master_chef, int balance) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        throw_unless(73, equal_slices(sender_address, master_chef));
        slice from_address = in_msg_body~load_msg_addr();
        int jetton_amount = in_msg_body~load_coins();

        balance += jetton_amount;
        save_data(owner,master_chef,balance);
        return();
    }

    if ( op == op::withdraw() ){
        int amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();
        throw_unless(73, equal_slices(sender_address, master_chef));
        throw_unless(500,balance >= amount);

        balance -= amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),master_chef,0,initWithdrawReplyMsg(amount,sender));
        save_data(owner,master_chef,balance);
        return();
    }

    throw(0xffff);
}
```

When sending `Withdraw` messages for withdrawals, since `miniChef`'s `Withdraw` message has a require check that checks `self.amount` against `msg.amount`, if the amount passed in by the user is greater than their contract balance, `miniChef` will keep backing out, which will cause the `totalSupply` variable to continually becomes smaller, and the user receives an abnormally large reward.

**Risk Level: Critical**

## Competitive Conflict

Because all contracts in a TON transaction are executed asynchronously, it is possible to continue to send the same message to bypass limits such as balances or boolean values after a contract in the transaction has been executed, if the state has not been modified. For example:

Tact:

```tact
contract JettonWallet{
    balance: Int as uint64 = 0;
    owner: Address;
    master: Address;
    init(owner: Address, master: Address){
        self.owner = owner;
        self.master = master;
    }

    receive(msg: Transfer){
        require(self.balance >= msg.amount, "not enough balance");
        require(self.owner == context().sender,"not owner");
        InternalTransfer{
            amount: msg.amount,
            sender: self.owner
        }.toCell().send_to(msg.receiverWallet);
    }

    receive(msg: InternalTransfer){
        require(context().sender == contractAddress(initOf JettonWallet(msg.sender, self.master)),"incorrect wallet");
        self.balance += msg.amount;
        TransferReply{
            amount: msg.amount,
            sender: self.owner
        }.toCell().send_to(context().sender);
    }

    receive(msg: TransferReply){
        require(context().sender == contractAddress(initOf JettonWallet(msg.sender, self.master)),"incorrect wallet");
        self.balance -= msg.amount;
    }
}
```

Func:

```func
;;jettonWallet
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, int balance,cell jetton_wallet_code, slice master) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer() ){
        slice recv_wallet = in_msg_body~load_msg_addr();
        int jetton_amount = in_msg_body~load_coins();
        throw_unless(500, balance >= jetton_amount);
        throw_unless(73, equal_slices(sender_address, owner));

        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),recv_wallet,0,initInternalTransferMsg(amount,owner));
        save_data(owner, balance, jetton_wallet_code, master);
        return();
    }
    if ( op == op::internal_transfer() ){
        int amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        ;;check wallet
        throw_unless(73, equal_slices(sender_address, getJettonWallet_Address(from_address, master, jetton_wallet_code)));
        balance += amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),sender_address,0,initTransferReplyMsg(amount,owner));
        save_data(owner, balance, jetton_wallet_code, master);
        return();
    }

    if ( op == op::transfer_reply() ){
        int amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        ;;check wallet
        throw_unless(73, equal_slices(sender_address, getJettonWallet_Address(from_address, master, jetton_wallet_code)));
        balance -= amount;
        save_data(owner, balance, jetton_wallet_code, master);
        return();
    }

    throw(0xffff);
}
```

In this simple Jetton implementation, the contract implements the following flow: 

![image-20240813161704417](./README.assets/image-20240813161704417.png)

The above flow may have competing problems, e.g., wallet A has a balance of 5 Jetton, it can send a message to wallet B to transfer 5 Jetton twice, and since A's Jetton balance is not deducted in the first step, both of them can send `InternalTransfer` messages to wallet B without any problem, and wallet B's balance can be increased twice, and B will receive 10 Jetton, even if there is an exception in the third step when the balance is not enough, it will not be rolled back.

**Risk level: Critical**

## Unbounded Data Structure

The underlying TVM uses a cell approach to store data, the mapping is implemented as a unit tree, and writing to a leaf in the tree requires writing a new hash along the entire height. If an attacker tries to spam transaction keys in the mapping, some user balances will be pushed so deep into the tree that updating them will exceed the gas limit (currently capped at 1TON per contract). Example:

Tact:

```tact
contract stakePool{
    userBalance: map<Address,Int>;

    receive("deposit") {
        let ctx: Context = context();
        require(ctx.value > ton("0.001"), "not enough gas");
        let stakeAmnount: Int = ctx.value - ton("0.001");
        let prevBal: Int? = self.userBalance.get(ctx.sender);
        if (prevBal == null){
            self.userBalance.set(ctx.sender,stakeAmnount);
        }else{
            self.userBalance.set(ctx.sender,prevBal!! +stakeAmnount);
        }

    }

    receive(msg: Withdraw) {
        let ctx: Context = context();
        let prevBal: Int = self.userBalance.get(ctx.sender)!!;
        require(prevBal >= msg.amount,"not enough balance");
        require(ctx.value > ton("0.001"),"not enough gas");
        self.userBalance.set(ctx.sender, prevBal- ctx.value);
        send(SendParameters{
                to: msg.receiver,
                value: msg.amount,
                mode: SendRemainingValue + SendIgnoreErrors,
                bounce: false,
                body: WithdrawExcesses {
                    amount: msg.amount
                }.toCell()
            });

    }  
}
```

Func:

```func
;;stakePool
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (cell userBalance) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        throw_unless(500, msg_value >= deposit_fee());
        int stake_amount = msg_value -deposit_fee();
        (slice pre_value,int flag) = userBalance.dict_get?(sender_address);
        if (pre_value == null()){
            userBalance~dict_set(256,sender_address,stake_amount);
            save_data(total_supply, userBalance);
            return();
        }
        userBalance~dict_set(256,sender_address,stake_amount + pre_value);
        save_data(userBalance);
        return();
    }
    if ( op == op::withdraw() ){
        int amount = in_msg_body~load_coins();
        slice receiver = in_msg_body~load_msg_addr();
        (slice pre_value,int flag) = userBalance.dict_get?(sender_address);
        
        throw_unless(501,flag);
        throw_unless(500, pre_value >= amount);
        userBalance~dict_set(256,sender_address,pre_value - amount);
        	sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),sender_address,amount,initWithdrawExcessMsg(amount));
        save_data(userBalance);
        return();
    }
    throw(0xffff);
}
```

It's a very simple pledge contract, and this architecture is common on Ether, but in the TON, once too many user addresses are involved, or too much spam data is added by malicious actors, it can lead to some user's data being pushed into deep references in the cell store, making modifying their data consume so much gas that it exceeds the gas limit, making the balance inaccessible to those users. 

To avoid generating too much data, it is recommended to use contract slicing, that is, splitting a logical contract into multiple different chunks. Taking the pledge contract as an example, if each user's data, including takeAmount, rewardDebet, reward, etc., is stored in the same contract with totalSupply in the pool when the number of users increases to a certain amount When the number of users increases to a certain number, it may lead to excessive gas consumption when querying the data, therefore, a separate contract can be set up for each user, such as the miniChef contract above, which stores the user's data separately from the pool data, and will not cause unnecessary gas consumption or even dos problems due to too many users, and the implementation of the TON Jettons also uses this model.

**Risk Level: Critical**

## Over-reliance States

In Solidity, state queries between contracts occur very frequently, e.g., to query the contract balance, to query the current price of a pair, etc. In TON, however, relying too much on the state of other contracts can cause serious problems. However, in TON, over-reliance on the states of other contracts can cause serious problems, e.g.:

Tact:

```tact
contract aggregator{
    pair: Address;
    USDCWallet: Address;
    init(pair: Address, USDCWallet: Address) {
        self.pair = pair;
        self.USDCWallet = USDCWallet;
    }
    receive(msg: SwapJettonNotification) { 
        require(sender() == self.USDCWallet, "Unauthorized sender"); 
        GetAmountOut {
            amountIn: msg.amount,
            minReturnAmount : msg.minReturnAmount,
            sender: msg.sender
        }.toCell().send_to(self.pair);
    }

    receive(msg: SwapReply) { 
        require(sender() == self.pair, "Unauthorized sender"); 
        /*
        Execute the subsequent swap logic
        */
    }
}

contract pair{
    reserve0: Int as coins;
    reserve1: Int as coins;
    aggregator: Address;
    init(aggregator: Address) {
        self.aggregator = aggregator;
        self.reserve0 = 0;
        self.reserve1 = 0;
    }
    receive(msg: GetAmountOut) { 
        let returnAmount: Int =msg.amountIn *self.reserve0/self.reserve1;
        require(returnAmount>= msg.minReturnAmount,"not enough Amount");
        SwapReply {
            amountIn: msg.amountIn,
            sender: msg.sender
        }.toCell().send_to(self.aggregator);
    }
}
```

Func:

```func
;;aggregator
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice pair, slice USDC_wallet) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer_notification() ){
        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        slice forward_payload = in_msg_body;
        int min_return_amount = forward_payload~load_coins();

        throw_unless(73, equal_slices(sender_address, USDC_wallet));
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),pair,0,initGetAmountOutMsg(jetton_amount, min_return_amount, from_address));
        save_data(pair, USDC_wallet);
        return();
    }
    if ( op == op::swap_reply() ){
        throw_unless(73,equal_slices(sender_address, pair));
        ;;Execute the subsequent swap logic
        save_data(pair, USDC_wallet);
        return();
    }
    throw(0xffff);
}
;;pair
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice aggregator, int reserve0, int reserve1) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::get_amount_out() ){
        int amount = in_msg_body~load_coins();
        int min_return_amount = in_msg_body~load_coins();
        slice sender = in_msg_body~load_msg_addr();

        int return_amount = min_return_amount * reserve0/reserve1;
        throw_unless(73,return_amount >= min_return_amount);
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),aggregator,0,initSwapReplyMsg(amount, sender));
        save_data(aggregator, reserve0, reserve1);
        return();
    }

    throw(0xffff);
}
```

Since the interaction between contracts in TON can only be carried out by sending internal messages and cannot be directly queried by get and other functions, the demo performs swap transactions in the following order: the `aggregator` contract receives the `SwapJettonNotification` message - sends a `GetAmountOut` message to the pair contract to query whether the current price meets the `minReturnAmount` requirement - if the price meets the requirement, sends a `SwapReply` to the `aggregator` contract to execute the subsequent swap logic. Send a `GetAmountOut` message to the `pair` contract to check whether the current price meets the requirement of `minReturnAmount` -- If the price meets the requirement, then send `SwapReply` to the `aggregator` contract to execute the subsequent swap logic.

In a public chain with atomic transactions, this process does not cause problems, but due to the asynchronous nature of the TON, none of the above three steps may be in a single block. When the price is queried in the second step, the price may still change, resulting in the tokens eventually being exchanged to the user's hands not meeting the minimum return tokens, but at this time the transaction does not cause problems but is executed normally.

**Risk Level: High**

## Message Pattern Error

There are many message patterns for delivering messages in TON, and you need to choose the message pattern carefully when you send a message. For example, there is `SendRemainingBalance`, which will forward all the balances in the contract to the receiver, and it is usually used together with the message pattern `SendDestroyIfZero` to destroy the contract, which destroys the contract and sends the remaining TON to the receiver, similar to the self-destruct function in the previous Ether. It will destroy the contract and send the remaining TON to the receiver, similar to the previous self-destruct function on Ether. One of the most commonly used message mode is `SendRemainingValue`, using this mode to send a message will forward the remaining gas fee together to the receiver, it will be very convenient to execute in a linear message flow, and it is also a method to recover the gas at the end of the message flow. However, if other processing logic still exists after the message is sent, such as sending events or changing state, it will consume the balance of the contract itself. Example:

Tact:

```tact
contract userWallet{
    owner: Address;
    master: Address;
    amount: Int as coins = 0;
    init(owner: Address,master: Address) {
        self.owner = owner;
        self.master = master;
        
    }
    receive(msg: Deposit) { 
        require(sender() == self.master ,"not masterchef");
        self.amount += msg.amount;
    }

    receive(msg: Transfer) { 
        require(sender() == self.owner ,"not masterchef");
        require(context().value >= ton("0.005"),"not enough gas");
        self.amount -= msg.amount;
        send(SendParameters{
                to: self.master,
-                value: msg.amount,
-                mode: SendRemainingValue + SendIgnoreErrors,
+                value: context().value-ton("0.0001"),
+                mode: SendIgnoreErrors,
                bounce: false,
                body: Transfer {
                    amount: msg.amount,
                    receiverWallet: msg.receiverWallet
                }.toCell()
            });
        emit(TransferEvent{sender: self.owner, receiver: msg.receiverWallet, amount: msg.amount}.toCell());
    }

}
```

Func:

```func
;;userWallet
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, slice master, int balance) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        int amount = in_msg_body~load_coins();
        throw_unless(73,equal_slices(sender_address, master));

        balance += amount;
        save_data(owner, master, balance);
        return();
    }

    if ( op == op::transfer() ){
        throw_unless(500,msg_value >= transfer_fee());
        throw_unless(73,equal_slices(sender_address, owner));
        int amount = in_msg_body~load_coins();
        slice receiverWallet = in_msg_body~load_msg_addr();
        slice responce_addr = in_msg_body~load_msg_addr();

        balance -= amount;
-        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),master,0,initTransferMsg(amount, receiverWallet));
+		 sendMsg(bounce::false(),sendMode::NONE(),master,msg_value- event_fee(),initTransferMsg(amount, receiverWallet));
        sendMsg(bounce::false(),sendMode::NONE(),responce_addr,event_fee(),initTransferEventMsg(owner, receiverWallet, amount));
        save_data(owner, master, balance);
        return();
    }

    throw(0xffff);
}
```

In the `userWallet` contract, the Transfer message uses `SendRamainingValue` to send a `Transfer` message to the master and triggers the `TransferEvent` event at the end of the contract. Since the gas fee carried by this message has already been sent to the `master` contract with the message, the triggering of this event consumes the internal balance of the contract, and as the number of transactions increases, each event will consume a part of the contract balance, resulting in the contract balance going to zero and freezing.

**Risk level:  High**

## Variable Override

In func, when processing message logic, the contract state variables need to be extracted or saved by load_data() or save_data(), and the variables that need to be extracted and saved are listed in order, and the order can't be changed, or else there will be the problem of wrong variable assignment. Also, func supports redeclaring variables, which is not a problem with this design, but it can be confusing when saving global variables with save_data(), for example:

```FunC
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (int total_supply, slice jetton_master,slice jetton_wallet) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::mint() ){
-        slice jetton_wallet = in_msg_body~load_msg_addr();
+        slice receiver_wallet = in_msg_body~load_msg_addr();
        int amount = in_msg_body~load_coins();
        throw_unless(500, msg_value >= (amount + mint_gas()));

        total_supply += amount;
-        sendMsg(bounce::true(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),jetton_master,0,initMintMsg(jetton_wallet, amount));
+        sendMsg(bounce::true(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),jetton_master,0,initMintMsg(receiver_wallet, amount));
        save_data(total_supply,jetton_master,jetton_wallet)
        return();
    }

    if ( op == op::transfer_notification() ){;;burn
        throw_unless(73, equal_slices(sender_address, jetton_wallet));
        throw_unless(500, msg_value >= burn_gas());

        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();

        total_supply -= jetton_amount;
        sendMsg(bounce::false(),sendMode::NONE(),from_address,jetton_amount,initReplyMsg(jetton_amount));
        save_data(total_supply,jetton_master,jetton_wallet)
        return();
    }

    throw(0xffff);
}
```

This is a very simple `master` contract, where the user can exchange his ton for a jetton on a one-for-one basis, transfer the ton to the contract to get the jetton via the `mint` message, and then send the jetton to the contract to get the ton back. the above logic i.e. there is a variable override problem, in the `mint` message, the ` jetton_wallet` variable from the message body, due to the same name, the globally stored field is shaded by the local variable, and finally, in the `save_data()` function, the `jetton_wallet` in the message overwrites the stored field in the contract, allowing the attacker to modify the state in the contract, and to use the pseudo jetton contract to take out the ton.

**Risk level:  High**

## Incorrect Checksum Rules

The Jetton standard in TON does not currently have a transferFrom function, so many contracts use the transfer_notification message in the Jetton standard as the start message for a swap or deposit transaction. other problems due to asynchrony. However, if a fallback issue occurs in the execution flow, it can result in the user's assets sustaining a loss, for example:

Tact:

```tact
contract stakePool{
    totalsupply: Int as uint64;
    userPool: Address;
    lpTokenWallet: Address;
    init(userPool: Address,lpTokenWallet: Address) {
        self.userPool = userPool;
        self.totalsupply = 0;
        self.lpTokenWallet = lpTokenWallet;

    }
    receive(msg: JettonNotification) { 
-        require(context().value >= ton("0.05"),"insufficient gas");
+        if(context().value >= ton("0.05")){
+            self.sendJetton(msg.amount, msg.sender);
+            return;
+        }
        require(context().sender == self.lpTokenWallet,"incorrect jetton");
        self.totalsupply += msg.amount;
        msg.toCell().send_to(self.userPool);
    }

    receive(msg: Withdraw) { 
        require(context().value >= ton("0.05"),"insufficient gas");
        msg.toCell().send_to(self.userPool);
    }

    receive(msg: WithdrawReply) { 
        require(context().sender == self.userPool,"not userPool");
        self.totalsupply -= msg.amount;
        self.sendJetton(msg.amount,msg.sender);
    }
}
contract userPool{
    owner: Address;
    stakePool: Address;
    amount: Int as coins = 0;
    init(owner: Address,stakePool: Address) {
        self.owner = owner;
        self.stakePool = stakePool;
        
    }
    receive(msg: Deposit) { 
        require(sender() == self.stakePool ,"not masterchef");
        self.amount += msg.amount;
    }

    receive(msg: Withdraw) {
        require(sender() == self.stakePool ,"not masterchef");
        require(self.amount >= msg.amount,"not enough balance");
        self.amount -= msg.amount;
        WithdrawReply {
            amount: msg.amount,
            sender: msg.sender
        }.toCell().send_to(self.stakePool);
    }
}
```

Func:

```func
;;stakePool
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (int total_supply, slice user_pool, slice lp_wallet) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer_notification() ){
        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73,equal_slices(sender_address, lp_wallet));
-        throw_unless(500, msg_value >= common_fee());
+        if(msg_value <= common_fee()){
+            send_jetton(jetton_amount,from_address);
+            return();
+        }

        total_supply += amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),user_pool,0,initDepositMsg(amount, from_address));
        save_data(total_supply, user_pool, lp_wallet);
        return();
    }

    if ( op == op::withdraw() ){
        throw_unless(500,msg_value >= common_fee());
        int amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();

        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),user_pool,0,initWithdrawMsg(amount, from_address));
        save_data(total_supply, user_pool, lp_wallet);
        return();
    }

    if ( op == op::withdraw_reply() ){
        throw_unless(73,equal_slices(sender_address, user_pool));
        int amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();

        send_jetton(amount, from_address);
        save_data(total_supply, user_pool, lp_wallet);
        return();
    }

    throw(0xffff);
}

;;userPool
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (int balance, slice owner, slice stake_pool) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73,equal_slices(sender_address, stake_pool));

        balance += amount;
        save_data(balance, owner, stake_pool);
        return();
    }

    if ( op == op::withdraw() ){
        throw_unless(73,equal_slices(sender_address, stake_pool));
        int amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73,balance >= amount);

        balance -= amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),stake_pool,0,initWithdrawReplyMsg(amount, from_address));
        save_data(total_supply, user_pool, lp_wallet);
        return();
    }

    throw(0xffff);
}
```

The `stakePool` contract uses `JettonNotification` as the entry point for deposits and checks whether the gas is enough to satisfy the entire message flow. The contract uses require in the message to check whether the gas meets the requirements, but at this time the user has already transferred Jetton to the corresponding wallet of the contract, this time throwing an exception will result in the user transferring tokens not being taken out and thus locked up. The correct way to do it is to use the if check, and if it triggers the if block, then the user's incoming jetton will be sent back to the user's wallet.

**Risk level: Medium**

## Erroneous Assertions in the Message Flow

Due to the current problems with the bounce mechanism, it cannot be used as a means of preventing message processing failures and incomplete execution. Therefore, in the entry message of the entire message flow, it is necessary to check as much as possible all the contents that can be checked in the first step, to minimize the possibility of errors in subsequent messages, and minimize the use of required and other checking statements in the subsequent message, because a require failure may lead to the termination of the entire message flow. Because a required failure may lead to the termination of the entire message flow, it is prudent to check before use whether this fallback will lead to a state exception in the part of the message flow before this. For example:

Tact:

```tact
contract master{
    owner: Address;
    master: Address;
    jettonWallet: Address;
    walletAddress: map<Address, Address>;
    init(owner: Address,master: Address, jettonWallet: Address) {
        self.owner = owner;
        self.master = master;
        self.jettonWallet = jettonWallet;
    }
    receive(msg: JettonNotification) { 
        require(sender() == self.jettonWallet, "incorrect jetton");
        if(context().value >= ton("0.05")){
            self.sendJetton(msg.amount,msg.sender);
            return;
        }
        let wallet: Address = self.walletAddress.get(msg.sender)!!;
        Deposit{
            amount: msg.amount,
            sender: msg.sender
        }.toCell().send_to(wallet);
    }

    receive(msg: Transfer) { 
        require(sender() == self.walletAddress.get(msg.senderWallet)!! ,"not masterchef");
        Deposit{
            amount: msg.amount,
            sender: msg.senderWallet
        }.toCell().send_to(self.walletAddress.get(msg.receiverWallet)!!);
    }

   
}
contract userWallet{
    owner: Address;
    master: Address;
    amount: Int as coins = 0;
    init(owner: Address,master: Address) {
        self.owner = owner;
        self.master = master;
        
    }
    receive(msg: Deposit) { 
        require(sender() == self.master ,"not masterchef");
        self.amount += msg.amount;
    }

    receive(msg: Transfer) { 
        require(sender() == self.owner ,"not masterchef");
        require(context().value >= ton("0.005"),"not enough gas");
        self.amount -= msg.amount;
        msg.toCell().send_to(self.master);
    }

   
}
```

Func:

```
;;master
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, slice jetton_wallet, cell wallet_addrs) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::transfer_notification() ){
        int query_id = in_msg_body~load_uint(64);
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73,equal_slices(sender_address, jetton_wallet));
        if (msg_value <= common_fee()){
            send_jetton();
            return();
        }

        (slice wallet, int flag) = wallet_addrs.dict_get(from_address);
        if (wallet == null()){
            send_jetton();
            return();
        }
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),wallet,0,initDepositMsg(jetton_amount, from_address));
        save_data(owner, jetton_wallet, wallet_addrs);
        return();
    }

    if ( op == op::transfer() ){
        int amount = in_msg_body~load_coins();
        slice sender_wallet = in_msg_body~load_msg_addr();
        slice receiver_wallet = in_msg_body~load_msg_addr();
        (slice sender, int flag) = wallet_addrs.dict_get(sender_wallet);
        throw_unless(501, flag);
        throw_unless(73,equal_slices(sender_address, sender));

        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),receiver_wallet,0,initDepositMsg(amount, sender_wallet));
        save_data(owner, jetton_wallet, wallet_addrs);
        return();
    }

    throw(0xffff);
}

;;userWallet
() recv_internal(int my_balance,int msg_value, cell in_msg_full, slice in_msg_body) impure { 
    (slice owner, slice master, int balance) = load_data();
    
    var ctx = var (flags,sender_address,fwd_fee) = load_in_msg_full(in_msg_full);
    if (flags & 1) { ;;ignore  bounced messages
        return ();
    }  

    if (in_msg_body.slice_empty?()) { 
        return ();
    }

    int op = in_msg_body~load_uint(32);

    ;;begin to handle message
    if ( op == op::deposit() ){
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(73,equal_slices(sender_address, master));

        balance += jetton_amount;
        save_data(owner, master, balance);
        return();
    }

    if ( op == op::transfer() ){
        throw_unless(73,equal_slices(sender_address, owner));
        throw_unless(500, msg_value >= common_fee());
        int amount = in_msg_body~load_coins();
        slice sender_wallet = in_msg_body~load_msg_addr();
        slice receiver_wallet = in_msg_body~load_msg_addr();

        balance -= jetton_amount;
        sendMsg(bounce::false(),sendMode::CARRY_ALL_REMAINING_INCOMING_VALUE(),master,0,initTransferMsg(amount, sender_wallet, receiver_wallet));
        save_data(owner, master, balance);
        return();
    }

    throw(0xffff);
}
```

In the above code, after the user deposits the jetton into the `userWallet` contract through the `master` contract, at this point, when the user wants to transfer his internal balance to another user through the `Transfer` message when the message is successfully forwarded to the `master` contract if there is an error in the sender in the message, an exception is thrown and no corresponding amount is added to the recipient's The `userWallet` balance will not be increased by the corresponding amount, but at this point the `userWallet` has already been deducted from the user's balance, resulting in the assets that the user wants to transfer being locked up in the contract.

**Risk Level: Medium**

## Gas Recovery
Due to the above vulnerability, the developer will limit the number of incoming gas at the beginning of the message flow, even if it is the same entry function, the message flow will not execute all branches every time under certain circumstances, and some messages will be skipped in the middle of the flow, and the gas saved by these skipped messages will need to be refunded to the user. Strictly speaking, this will not lead to some serious problems in the contract, it will only keep the remaining gas in the contract, but it will make the user pay more commission than normal, so it is still recommended to refer to Jetton's standard implementation to return the excess gas as an EXIT message to the user, which is implemented by Jetton as follows:

Tact:

```tact
// 0xd53276db -- Cashback to the original Sender
        if (msg.response_destination != null) { 
            send(SendParameters {
                to: msg.response_destination, 
                value: msgValue,  
                bounce: false,
                body: TokenExcesses { 
                    queryId: msg.queryId
                }.toCell(),
                mode: SendIgnoreErrors
            });
        }
```

Func:

```func
if ((response_address.preload_uint(2) != 0) & (msg_value > 0)) {

    var msg = begin_cell()
      .store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 010000
      .store_slice(response_address)
      .store_coins(msg_value)
      .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
      .store_uint(op::excesses(), 32)
      .store_uint(query_id, 64);

    send_raw_message(msg.end_cell(), 2);
  }
```

**Risk Level: Low**

# Vulnerability List

| Vulnerability Name                      | Description                                                  | Risk Level |
| --------------------------------------- | ------------------------------------------------------------ | ---------- |
| Gas Limitation                          | Not limiting the message flow gas can lead to inconsistent contract states. | Critical   |
| UnBouncedMessage                        | Handle the necessary bounces to ensure complete message execution. | Critical   |
| Competitive Conflict                    | Asynchronous execution can have competing problems.          | Critical   |
| Unbounded Data Structure                | Unbounded data structures can appear `dos`                   | Critical   |
| Over-reliance States                    | Over-reliance on contract status can be exploited.           | High       |
| Message Pattern Error                   | Incorrect message mode selection may consume all `Ton`.      | High       |
| Variable Override                       | Local variables may override global storage.                 | High       |
| Incorrect CheckSum Rules                | Assertions in some message portals may result in loss of user balances. | Medium     |
| Erroneous Assertion in the Message Flow | Assertions in the message flow can lead to incomplete message execution. | Medium     |
| Gas Recovery                            | Recovery of unused gas.                                      | Low        |

# Checklist

- [ ] Calculate the contract message flow gas consumption, and verify that each message flow port correctly limits the gas.
- [ ] Calibrate whether the location of each thrown exception affects the complete execution of the message flow.
- [ ] Identify potential competitive conditions: list all the process steps of a transaction and the state changes of each step, create a two-dimensional table, and analyze whether there may be competitive condition issues in concurrent states by looking at the intersection of the states of each step.
- [ ] Calibrate each data structure such as mapping present in the contract for possible unbounded data.
- [ ] Note the use of message patterns for each message sent.
- [ ] Compare the names of local and stored variables for any overlap.
- [ ] Assume that an exception occurs in each message flow and determine if the exception will be handled correctly.
- [ ] Avoid using exception termination for cases where the contract has already received an asset.
- [ ] Calibrate whether excess gas is recovered after each message flow.
- [ ] Verify that `load_data` and `save_data` are in the correct order of data.
- [ ] Check that functions in FunC contain the `impure` modifier, otherwise the function may be skipped.
- [ ] When calling a function, make sure that the function is a modified or unmodified method.
- [ ] When executing the `rand` function, make sure that each seed is randomized; a fixed-value seed can be brute-force broken.