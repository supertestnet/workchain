# Workchain

A drivechain alternative with no softfork and no need for bitcoin miners to run special software

# Introduction: worklocks

Bitcoin script has a little-known method for validating proofs of work inside of a transaction. It takes advantage of the fact that non-taproot bitcoin signatures are encoded in a format called DER encoding, in which they have a variable length. The length of a bitcoin signature is typically 72 or 73 bytes but they can be smaller. What makes them smaller is this: signatures have two seemingly random numbers in them, called r and s, and if those numbers have lots of leading 0 byte vectors, the DER encoding format removes them. Typically, neither number has a leading 0 byte vector, so DER encoded signatures are usually their maximum size, which is 73 bytes. But often one of them does have a leading zero, by chance, so the signature is 72 bytes.

However, by creating a lot of signatures through brute force, you can, by chance, derive signatures whose numbers have many leading zeroes, making the resulting signatures relatively short. Doing this takes time and compute cycles – i.e. work – and to find a *very* short sig takes a lot of work, and therefore costs a lot of money. But the cool trick is that bitcoin script not only has a function for validating a signature (that function is called OP_CHECKSIG), it also has a function (called OP_SIZE) for requiring a signature to be a certain length. So you can make a requirement that someone not only produce a valid signature to spend some money, but they must also do work to spend it with a “short” signature. If you make the length requirement short enough, you can essentially impose a cost on anyone who wants to spend your money.

A bitcoin address that requires a short signature – one that takes a lot of work – is locked by a proof-of-work requirement. A valid signature of sufficiently short length is proof that whoever generated it did a certain amount of work. Such addresses are therefore sometimes called worklocks. And I think this little-known trick of bitcoin script can be exploited to create a sidechain with a 2 way peg – without waiting for a soft fork like bip300 (aka drivechain).

# Lifecycle of a workchain deposit

1. Alice deposits X btc into a worklock that costs 20 times the value of X to unlock
2. She ensures the worklock is also timelocked with a waiting period of 3 months
3. She announces her deposit on the workchain to get X “workchain bitcoins”
4. She does stuff on the workchain and (let’s suppose) loses ownership of her money
5. Before 3 months pass, the rightful owner of Alice’s deposit may post a withdrawal request
6. After 3 months, workchain miners must unlock the deposit and send it to the rightful owner
7. If the rightful owner requested no withdrawal, miners must resend the utxo to the workchain

# How mining usually works on a workchain

I think steps 1 through 5 are simple enough but I do have more to say about steps 6-7. Nonetheless, first let me talk a little about “regular mining” on the workchain. Between the time money first enters the workchain and the time money first leaves the workchain, workchain miners must process blocks whereby money moves *on* the workchain. A workchain miner should *ordinarily* create new workchain blocks in the same manner as how [my spacechain launcher](https://github.com/supertestnet/spacechain-launcher/) works: there is a dust amount in an ACSA (anyone_can_spend address) and miners move it out of the ACSA and back into the ACSA in every bitcoin block (if they can keep up). Every time they move it, they may include a 32 byte string in the witness data corresponding to the merkle root of a block of workchain transactions, and in this merkle tree they may include a “coinbase transaction” that pays themselves fees taken from every transaction in that block. As long as their workchain block is valid, and they included its merkle root in the bitcoin transaction that moves the ACSA output, the miner gets paid for mining the workchain.

Also, like with my spacechain launcher, if a troll steals the ACSA utxo, workchain miners can make a new one. A consensus rule of the workchain states that the first ACSA output that appears on bitcoin’s blockchain after a theft is, by consensus, the new one that all workchain miners must use.

# More about steps 6-7

Now that we know how workchain miners ordinarily mine blocks, let’s get back to step 6. When the 3 months on a deposit transaction pass by, miners must not spend the ACSA output in the next block, like they did before, they must instead unlock the depositor’s utxo and send it to whatever “withdrawal address” the rightful owner designated in their withdrawal request. Or, if the rightful owner did not post a withdrawal request, miners must redeposit the money back into the workchain.

Like miners do in the previous section, miners of a step 6 transaction may include in this transaction’s witness data a 32 byte value that constitutes the merkle root of a block of workchain transactions. And, as long as their workchain block is valid, the miner gets fees on the workchain for mining a block. Any miner who steals the withdrawer’s coins does *not* get any fees – and their block, if they posted one, is considered invalid because they did not process a withdrawal or redeposit the funds as they were meant to. (A drawback of this, but not a fatal one I think, is that no more than one withdrawal from the workchain can be processed per bitcoin block.)

By keeping the amount of time bitcoin can be in a worklock low – to about 3 months – and the value of the worklock that protects that deposit high – 20x the value it protects – the value of the original deposit is unlikely to get so high it exceeds the cost of unlocking it. If that happened, it would be dangerous, because reasonable thieves might try to unlock it in order to steal the sats inside, and it would be worth it if the value of the worklocked sats got higher than the cost of unlocking their worklock. By keeping every worklock very expensive in comparison to the bitcoins they protect, the only way to *profitably* unlock a workchain deposit is to get compensated for it somehow – such as by including the merkle root of a block of workchain transactions in your unlocking tx, that way you get paid in workchain fees for the work you did to move the deposit.

# Common deposit values

It would be bad if a person deposited a large bitcoin utxo worth like $1 million onto the workchain and therefore had to make a worklock that costs $20 million to unlock. Workchain transaction fees are unlikely, in a timely manner, to aggregate to an amount that can compensate miners for the cost of unlocking such a very valuable utxo. Therefore, there is a ceiling of some sort on the value of a utxo that can be safely deposited onto the workchain. I think a reasonable ceiling should be chosen and made a consensus rule of the workchain. Therefore, let there be a rule that deposits are only considered valid if they are worth about $50, and they must be encumbered with a worklock that costs about $2000 to unlock.

# Value splitting

Utxos deposited to the workchain can naturally be split up *on* the workchain and sent to many users. But the original deposit remains, from the perspective of bitcoin’s blockchain, one singular utxo. At withdrawal time, perhaps it will no longer be true that one person “owns” the original deposit. Various portions of it will probably belong to many people. Naively, this might seem to cause a problem if the owners of one part of a deposit want to withdraw their part, but the other owners do not.

However, this is not a significant problem. Workchain miners must merely split the utxo when they unlock it, then send the portions that are associated with withdrawal requests to their rightful owners, then redeposit the remainder back into the workchain. If the value of the remainder exceeds $50 because bitcoin’s price rose, the miner must split it up into several redeposits such that each one is worth $50. If necessary, they must supply the difference if the last redeposit is worth less than $50 on its own. To accomplish this safely, the miner can just add an extra input to the deposit tx and deposit that portion (from the workchain’s perspective) to one of his own workchain addresses.

# Trolls, trust, and tension

The existence of trolls may result in the following scenario: even though no one is incentivized *monetarily* to steal a worklocked deposit utxo (because doing so costs more than you can gain), a troll may do so anyway just to wreak havoc on the workchain by preventing some unlucky fellow(s) from getting their money. Every deposit to the workchain technically sits in an “anyone can spend” address that is only protected by a worklock and not by an unknown private key somewhere. That means anyone who does the work can spend those sats. The worklock on that address is meant to ensure that only workchain miners actually do so, because anyone else who does it will lose money, but trolls can do it too, just to be trolls, and that will probably happen sometimes.

Consequently, workchains are not trustless. They rely on incentives toward non-troll behavior. Thus workchain users might feel a bit of suspense, hoping that if/when a troll comes along to try to steal coins for the sake of wreaking havoc, at least (hopefully) it’s unlikely that they’ll do that right when you are next in line to withdraw. I don’t know how to solve this problem but maybe this sidechain model is good enough even with this flaw that I can leave it unsolved for now.

# The secret sauce and the boostrapping problem

The secret sauce of workchain is this: if the value of each deposit is only $50 and the cost to move it is $2000, miners are unlikely to steal it because they would lose money by doing so, but they probably *would* move it to earn a *right to withdraw* more than $2000 from the workchain in transaction fees. So we leverage that incentive to make the workchain *assign* miners transaction fees when they mine blocks properly. Miners can then withdraw their earnings at their leisure.

However, this incentive model introduces a bootstrapping problem: blocks on the sidechain *cost* at least $2000 to mine, therefore miners will only do so if they earn more than that in revenue. And, since there is no coinbase reward, that revenue must all come from transaction fees. But it's *hard* for a blockchain to get so popular that its miners earn $2000 in transaction fees per block. Your blockchain has to have a lot of usage, and, at least in the beginning stages, they usually don't.

To compensate for this problem, perhaps the sidechain's developers could *start it out* as a standard, federated sidechain, but shift to the workchain model when transaction fees regularly exceed $2000 per block. That might work, but, if the federation holds a lot of money, they might not *want* to make the shift when the time comes. Moreover, if liquid and rsk are any indication, getting enough users to pay $2000 per block in transaction fees may never happen. So the bootstrapping problem may just be insurmountable.

# Acknowledgements

Robin Linus for [showing](https://gist.github.com/RobinLinus/95de641ed1e3d9fde83bdcf5ac289ce9) that it is possible to express and verify proof of work within bitcoin script today.

Ruben Somsen for bitcoin wizardry and maintaining a helpful Telegram group of other fellow wizards.

Vman for maintaining my interest in worklocks and talking through [ideas](https://github.com/VzxPLnHqr/sig-pow#user-content-fnref-precision_note2-4ba98315fde2e55b0754b9c46ef44753) for how to make a sidechain with them.
