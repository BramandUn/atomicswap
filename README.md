**NOTICE Mar 1 2018:** The atomic swap contract has been updated to specify the
secret sizes to prevent fraudulent swaps between two cryptocurrencies with
different maximum data sizes.  Old contracts will not be usable by the new tools
and vice-versa.  Please rebuild all tools before conducting new atomic swaps.

# Stakenet cross-chain atomic swapping

This repo contains utilities to manually perform cross-chain atomic swaps
between Stakenet and other cryptocurrencies.  At the moment, support exists for
the following coins and wallets:

* Bitcoin ([Bitcoin Core](https://github.com/bitcoin/bitcoin))
* Bitcoin Cash ([Bitcoin ABC](https://github.com/Bitcoin-ABC/bitcoin-abc), [Bitcoin Unlimited](https://github.com/BitcoinUnlimited/BitcoinUnlimited), [Bitcoin XT](https://github.com/bitcoinxt/bitcoinxt))
* Litecoin ([Litecoin Core](https://github.com/litecoin-project/litecoin))
* Monacoin ([Monacoin Core](https://github.com/monacoinproject/monacoin))
* Particl ([Particl Core](https://github.com/particl/particl-core))
* Polis ([Polis Core](https://github.com/polispay/polis))
* Vertcoin ([Vertcoin Core](https://github.com/vertcoin/vertcoin))
* Viacoin ([Viacoin Core](https://github.com/viacoin/viacoin))
* Zcoin ([Zcoin Core](https://github.com/zcoinofficial/zcoin))
* Stakenet ([XSN Stakenet Core](https://github.com/X9Developers/XSN))

Pull requests implementing support for additional cryptocurrencies and wallets
are encouraged.  See [GitHub project
1](https://github.com/decred/atomicswap/projects/1) for the status of coins
being considered.

These tools do not operate solely on-chain.  A side-channel is required between
each party performing the swap in order to exchange additional data.  This
side-channel could be as simple as a text chat and copying data.  Until a more
streamlined implementation of the side channel exists, such as the Lightning
Network, these tools suffice as a proof-of-concept for cross-chain atomic swaps
and a way for early adopters to try out the technology.

Due to the requirements of manually exchanging data and creating, sending, and
watching for the relevant transactions, it is highly recommended to read this
README in its entirety before attempting to use these tools.  The sections
below explain the principles on which the tools operate, the instructions for
how to use them safely, and an example swap between Stakenet and Bitcoin.

## Build instructions

Pre-requirements:

  - Go 1.9 or later
  - btcd with Stakenet support ([btcd](https://github.com/X9Developers/btcd))
  - btcutil ([btcutil](https://github.com/btcsuite/btcutil))
  - btcwallet ([btcwallet](https://github.com/btcsuite/btcwallet))
  - btclog ([btclog](https://github.com/btcsuite/btclog))
  - golangcrypto ([golangcrypto](https://github.com/btcsuite/golangcrypto))
  - goleveldb ([goleveldb](https://github.com/btcsuite/goleveldb))
  - go-socks ([go-socks](https://github.com/btcsuite/go-socks))
  - snappy-go ([snappy-go](https://github.com/btcsuite/snappy-go))
  - websocket ([websocket](https://github.com/btcsuite/websocket))
  - `xsnwallet` 1.1.0 or later (for `xsnatomicswap`)

```
$ cd $GOPATH/src/github.com/xsncoin (create if doesn't exist)
$ git clone https://github.com/X9Developers/atomicswap && cd atomicswap
$ go install ./cmd/... (e.g <go install ./cmd/xsnatomicswap/>)
```

## Theory

A cross-chain swap is a trade between two users of different cryptocurrencies.
For example, one party may send XSN to a second party's XSN address, while
the second party would send Bitcoin to the first party's Bitcoin address.
However, as the blockchains are unrelated and transactions can not be reversed,
this provides no protection against one of the parties never honoring their end
of the trade.  One common solution to this problem is to introduce a
mutually-trusted third party for escrow.  An atomic cross-chain swap solves this
problem without the need for a third party.

Atomic swaps involve each party paying into a contract transaction, one contract
for each blockchain.  The contracts contain an output that is spendable by
either party, but the rules required for redemption are different for each party
involved.

One party (called counterparty 1 or the initiator) generates a secret and pays
the intended trade amount into a contract transaction.  The contract output can
be redeemed by the second party (called countryparty 2 or the participant) as
long as the secret is known.  If a period of time (typically 48 hours) expires
after the contract transaction has been mined but has not been redeemed by the
participant, the contract output can be refunded back to the initiator's wallet.

For simplicity, we assume the initiator wishes to trade Bitcoin for Stakenet with
the participant.  The initiator can also trade Stakenet for Bitcoin and the steps
will be the same, but with each step performed on the other blockchain.

The participant is unable to spend from the initiator's Bitcoin contract at this
point because the secret is unknown by them.  If the initiator revealed their
secret at this point, the participant could spend from the contract without ever
honoring their end of the trade.

The participant creates a similar contract transaction to the initiator's but on
the Stakenet blockchain and pays the intended Stakenet amount into the contract.
However, for the initiator to redeem the output, their own secret must be
revealed.  For the participant to create their contract, the initiator must
reveal not the secret, but a cryptographic hash of the secret to the
participant.  The participant's contract can also be refunded by the
participant, but only after half the period of time that the initiator is
required to wait before their contract can be refunded (typically 24 hours).

With each side paying into a contract on each blockchain, and each party unable
to perform their refund until the allotted time expires, the initiator redeems
the participant's Stakenet contract, thereby revealing the secret to the
participant.  The secret is then extracted from the initiator's redeeming Stakenet
transaction providing the participant with the ability to redeem the initiator's
Bitcoin contract.

This procedure is atomic (with timeout) as it gives each party at least 24 hours
to redeem their coins on the other blockchain before a refund can be performed.

The image below provides a visual of the steps each party performs and the
transfer of data between each party.

<img src="workflow.svg" width="100%" height=650 />

## Command line

Separate command line utilities are provided to handle the transactions required
to perform a cross-chain atomic swap for each supported blockchain.  For a swap
between Bitcoin and Stakenet, the two utilities `btcatomicswap` and
`xsnatomicswap` are used. Both tools must be used by both parties performing
the swap.

Different tools may require different flags to use them with the supported
wallet.  For example, `xsnatomicswap` as well as`btcatomicswap` includes flags for the RPC username and
password while e.g. `dcratomicswap` does not.  Running a tool without any parameters
will show the full usage help.

All of the tools support the same six commands.  These commands are:

```
Commands:
  initiate <participant address> <amount>
  participate <initiator address> <amount> <secret hash>
  redeem <contract> <contract transaction> <secret>
  refund <contract> <contract transaction>
  extractsecret <redemption transaction> <secret hash>
  auditcontract <contract> <contract transaction>
```

**`initiate <participant address> <amount>`**

The `initiate` command is performed by the initiator to create the first
contract.  The contract is created with a locktime of 48 hours in the future.
This command returns the secret, the secret hash, the contract script, the
contract transaction, and a refund transaction that can be sent after 48 hours
if necessary.

Running this command will prompt for whether to publish the contract
transaction.  If everything looks correct, the transaction should be published.
The refund transaction should be saved in case a refund is required to be made
later.

For the xsnatomicswap, btcatomicswap and ltcatomicswap tools the wallet must
already be unlocked. 

**`participate <initiator address> <amount> <secret hash>`**

The `participate` command is performed by the participant to create a contract
on the second blockchain.  It operates similarly to `initiate` but requires
using the secret hash from the initiator's contract and creates the contract
with a locktime of 24 hours.

Running this command will prompt for whether to publish the contract
transaction.  If everything looks correct, the transaction should be published.
The refund transaction should be saved in case a refund is required to be made
later.

For the xsnatomicswap, btcatomicswap and ltcatomicswap tools the wallet must
already be unlocked. 

**`redeem <contract> <contract transaction> <secret>`**

The `redeem` command is performed by both parties to redeem coins paid into the
contract created by the other party.  Redeeming requires the secret and must be
performed by the initiator first.  Once the initiator's redemption has been
published, the secret may be extracted from the transaction and the participant
may also redeem their coins.

Running this command will prompt for whether to publish the redemption
transaction. If everything looks correct, the transaction should be published.

For the xsnatomicswap, btcatomicswap and ltcatomicswap tools the wallet must
already be unlocked. 

**`refund <contract> <contract transaction>`**

The `refund` command is used to create and send a refund of a contract
transaction.  While the refund transaction is created and displayed during
contract creation in the initiate and participate steps, the refund can also be
created after the fact in case there was any issue sending the transaction (e.g.
the contract transaction was malleated or the refund fee is now too low).

Running this command will prompt for whether to publish the redemption
transaction. If everything looks correct, the transaction should be published.

**`extractsecret <redemption transaction> <secret hash>`**

The `extractsecret` command is used by the participant to extract the secret
from the initiator's redemption transaction.  With the secret known, the
participant may claim the coins paid into the initiator's contract.

The secret hash is a required parameter so that "nonstandard" redemption
transactions won't confuse the tool and the secret can still be discovered.

**`auditcontract <contract> <contract transaction>`**

The `auditcontract` command inspects a contract script and parses out the
addresses that may claim the output, the locktime, and the secret hash.  It also
validates that the contract transaction pays to the contract and reports the
contract output amount.  Each party should audit the contract provided by the
other to verify that their address is the recipient address, the output value is
correct, and that the locktime is sensible.

## Example

The first step is for both parties to exchange addresses on both blockchains. If
party A (the initiator) wishes to trade Bitcoin for Stakenet, party B (the
participant) must provide their Bitcoin address and the initiator must provide
the participant their Stakenet address.

_Party A runs:_
```
$ xsn-cli --testnet --rpcuser=<xsnd_user> --rpcpassword=<xsnd_pass> getnewaddress "" legacy
yNVi1yTAQGkBmiG5yXSUYDjFqn3WDFSsqK
```

_Party B runs:_
```
$ bitcoin-cli --regtest --rpcuser=<bitcoind_user> --rpcpassword=<bitcoind_pass> getnewaddress "" legacy
mqh7yjWX2rrAqkHj3B1JTBJbeSvvMCBWdP
```

*Note:* It is normal for neither of these addresses to show any activity on
block explorers.  They are only used in nonstandard scripts that the block
explorers do not recognize.

A initiates the process by using `btcatomicswap` to pay 1.0 BTC into the Bitcoin
contract using B's Bitcoin address, sending the contract transaction, and
sharing the secret hash (*not* the secret), contract, and contract transaction
with B.  The refund transaction can not be sent until the locktime expires, but
should be saved in case a refund is necessary.

_Party A runs:_
```
$ btcatomicswap --regtest --rpcuser=<bitcoind_user> --rpcpass=<bitcoind_pass> initiate mqh7yjWX2rrAqkHj3B1JTBJbeSvvMCBWdP 1
warning: falling back to mempool relay fee policy
Secret:      6489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2
Secret hash: 053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af

Contract fee: 0.00000188 BTC (0.00001005 BTC/kB)
Refund fee:   0.00000297 BTC (0.00001021 BTC/kB)

Contract (2MtHDUDTJ39HuNzFukZBthu3mrJKKXsLKc2):
6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a9146f9d80594e7e0359bec5058824cd487a3f0c9ba9670431b8385bb17576a914ae53dc0df8d89b2726600cc90dfb9419fe2affea6888ac

Contract transaction (137aa0e9a673b97ac352a83ac9efce50d9c84f37adfc89fb9f6221438c8f5fff):
0200000001b6e4e374030e6a707c10dd937da630ab837c3c9625f2e833e8e8e82e0e3f5cdf0000000048473044022016bce2c94d65d3e8ae58f4f6af263ca58338ac6a0ccf0d22975be1518fe1507902207e39071e0e0c9c51d867ab82d08caabedf9c341ca5f0cb9c8d7aefca3033940701feffffff02441010240100000017a914791205b2a871d2d7bf8a931eb68f347a20ab8d868700e1f5050000000017a9140b5886633426dacc8329990dd1eb8af5fdcf60978700000000

Refund transaction (ce27763f2c8ebcfc3818a0caf9e5ff6b5481808dd9f0bab182deea353c291a85):
0200000001ff5f8f8c4321629ffb89fcad374fc8d950ceefc93aa852c37ab973a6e9a07a1301000000ce4730440220228d10844d042159c49b5cbd05714569f9ae44eb549d997072ffb725cb2a630f02200be81dbde233a32b1c3d4f03755edf33a4ee2301a68a3ef4b4833ceb07569d6201210292851a67f8337fc756f1c3a2ed5520561fa4132632efb5ba43491224fa15ea47004c616382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a9146f9d80594e7e0359bec5058824cd487a3f0c9ba9670431b8385bb17576a914ae53dc0df8d89b2726600cc90dfb9419fe2affea6888ac0000000001d7dff505000000001976a9147b3fa7f6bcf104c67527a934a181682bfab1712a88ac31b8385b

Publish contract transaction? [y/N] y
Published contract transaction (137aa0e9a673b97ac352a83ac9efce50d9c84f37adfc89fb9f6221438c8f5fff)
```

Once A has initialized the swap, B must audit the contract and contract
transaction to verify:

1. The recipient address was the BTC address that was provided to A
2. The contract value is the expected amount of BTC to receive
3. The locktime was set to 48 hours in the future

_Party B runs:_
```
$ btcatomicswap --regtest --rpcuser=<bitcoind_user> --rpcpass=<bitcoind_pass> auditcontract 6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a9146f9d80594e7e0359bec5058824cd487a3f0c9ba9670431b8385bb17576a914ae53dc0df8d89b2726600cc90dfb9419fe2affea6888ac 0200000001b6e4e374030e6a707c10dd937da630ab837c3c9625f2e833e8e8e82e0e3f5cdf0000000048473044022016bce2c94d65d3e8ae58f4f6af263ca58338ac6a0ccf0d22975be1518fe1507902207e39071e0e0c9c51d867ab82d08caabedf9c341ca5f0cb9c8d7aefca3033940701feffffff02441010240100000017a914791205b2a871d2d7bf8a931eb68f347a20ab8d868700e1f5050000000017a9140b5886633426dacc8329990dd1eb8af5fdcf60978700000000
Contract address:        2MtHDUDTJ39HuNzFukZBthu3mrJKKXsLKc2
Contract value:          1 BTC
Recipient address:       mqh7yjWX2rrAqkHj3B1JTBJbeSvvMCBWdP
Author's refund address: mwQiL6gnXxkgVmHMmL2qJRo8oezRT2WLcY

Secret hash: 053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af

Locktime: 2018-07-01 11:17:05 +0000 UTC
Locktime reached in 47h53m1s
```

Auditing the contract also reveals the hash of the secret, which is needed for
the next step.

Once B trusts the contract, they may participate in the cross-chain atomic swap
by paying the intended Stakenet amount (1.0 in this example) into a Stakenet
contract using the same secret hash.  The contract transaction may be published
at this point.  The refund transaction can not be sent until the locktime
expires, but should be saved in case a refund is necessary.

_Party B runs:_
```
$ xsnatomicswap --testnet --rpcuser=<xsnd_user> --rpcpass=<xsnd_pass> participate yNVi1yTAQGkBmiG5yXSUYDjFqn3WDFSsqK 10 053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af
warning: falling back to mempool relay fee policy
Contract fee: 0.00000166 BTC (0.00000672 BTC/kB)
Refund fee:   0.00000297 BTC (0.00001021 BTC/kB)

Contract (8eamvP1pRfSaD8xMHUZNZEWch1nFFori7y):
6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888ac

Contract transaction (962d1bb3eb51ff1ba2a37bebc85391e05285c56b81e91e90e0b15ecac6db7f17):
0200000000010150fa0de5fc6d67d68005b3016c38d908fbc082cdc711de712a1ccaa9c3db574200000000171600142adaffdb0f3d8620ce44e51cee5b080aa60b48d5feffffff0200ca9a3b0000000017a91401c2f251dc6760aa12b5f2d0cf2d840dcc072aed87d678fafa1600000017a914969a87d464c7adce58a0a4e8cddaca4092cbc65d87024730440220554fbb1ac14e73fa1c5b88a144e963caae9c20de9301c0932c2ab6c6e28b2877022010e3ce717f2ada6c5378012f2151a3f7d1a3eb6af7ae20a301dcaae7143b4b97012102177fba6756f2a9498a95f1b5692be34cb9bc2ff0f55670fb689d87325f1362da00000000

Refund transaction (991865d361f7c7c7f735d2211a9c454f07575a72336392abcfa6e5b31f15fd21):
0200000001177fdbc6ca5eb1e0901ee9816bc58552e09153c8eb7ba3a21bff51ebb31b2d9600000000ce4730440220187b7e4db214342c5abbd7d3d493adac0f13635b50d106ead4d056198743286002206b617659f090543e2bff5b023030f9df44a493b48ded0aadfacf9c9ae03f4b9d012102a46b9fc2de33a6ec18a3bb44b536999f2c0c47272890b093a1158f36ccb94d8a004c616382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888ac0000000001d7c89a3b000000001976a9147e166c078b2e2660ee3f70faf6ae36d7925d7c9288ac6c69375b

Publish contract transaction? [y/N] y
Published contract transaction (962d1bb3eb51ff1ba2a37bebc85391e05285c56b81e91e90e0b15ecac6db7f17)
```

B now informs A that the Stakenet contract transaction has been created and
published, and provides the contract details to A.

Just as B needed to audit A's contract before locking their coins in a contract,
A must do the same with B's contract before withdrawing from the contract.  A
audits the contract and contract transaction to verify:

1. The recipient address was the XSN address that was provided to B
2. The contract value is the expected amount of XSN to receive
3. The locktime was set to 24 hours in the future
4. The secret hash matches the value previously known

_Party A runs:_
```
$ xsnatomicswap --testnet --rpcuser=<xsnd_user> --rpcpass=<xsnd_pass> auditcontract 6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888ac 0200000000010150fa0de5fc6d67d68005b3016c38d908fbc082cdc711de712a1ccaa9c3db574200000000171600142adaffdb0f3d8620ce44e51cee5b080aa60b48d5feffffff0200ca9a3b0000000017a91401c2f251dc6760aa12b5f2d0cf2d840dcc072aed87d678fafa1600000017a914969a87d464c7adce58a0a4e8cddaca4092cbc65d87024730440220554fbb1ac14e73fa1c5b88a144e963caae9c20de9301c0932c2ab6c6e28b2877022010e3ce717f2ada6c5378012f2151a3f7d1a3eb6af7ae20a301dcaae7143b4b97012102177fba6756f2a9498a95f1b5692be34cb9bc2ff0f55670fb689d87325f1362da00000000
Contract address:        8eamvP1pRfSaD8xMHUZNZEWch1nFFori7y
Contract value:          10 BTC
Recipient address:       yNVi1yTAQGkBmiG5yXSUYDjFqn3WDFSsqK
Author's refund address: ycPfAUtEM7jo6fJDvAYwA83zjAmXfnScas

Secret hash: 053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af

Locktime: 2018-06-30 11:28:44 +0000 UTC
Locktime reached in 23h57m8s
```

Now that both parties have paid into their respective contracts, A may withdraw
from the Stakenet contract.  This step involves publishing a transaction which
reveals the secret to B, allowing B to withdraw from the Bitcoin contract.

_Party A runs:_
```
$ xsnatomicswap --testnet --rpcuser=<xsnd_user> --rpcpass=<xsnd_pass> redeem 6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888ac 0200000000010150fa0de5fc6d67d68005b3016c38d908fbc082cdc711de712a1ccaa9c3db574200000000171600142adaffdb0f3d8620ce44e51cee5b080aa60b48d5feffffff0200ca9a3b0000000017a91401c2f251dc6760aa12b5f2d0cf2d840dcc072aed87d678fafa1600000017a914969a87d464c7adce58a0a4e8cddaca4092cbc65d87024730440220554fbb1ac14e73fa1c5b88a144e963caae9c20de9301c0932c2ab6c6e28b2877022010e3ce717f2ada6c5378012f2151a3f7d1a3eb6af7ae20a301dcaae7143b4b97012102177fba6756f2a9498a95f1b5692be34cb9bc2ff0f55670fb689d87325f1362da00000000 6489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2
warning: falling back to mempool relay fee policy
Redeem fee: 0.0000033 BTC (0.00001015 BTC/kB)

Redeem transaction (2359bd89e2766bbac5971d031d83d7f0164bc0a1a2982154cf3a4a02efe8dbe1):
0200000001177fdbc6ca5eb1e0901ee9816bc58552e09153c8eb7ba3a21bff51ebb31b2d9600000000f048304502210091e06c35787fec69d791702c3951bc8f20133a648f37109a719803a628036e6c0220467fb5755148be259f13b5e5719fb60af9689faf0789502881e6e9387e3e13c101210290d101672e3870c71bf0b4449c417404bd0ee66d32ba502a2527e9bdc8748f11206489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2514c616382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888acffffffff01b6c89a3b000000001976a914aba5c61d206d779c90b9da37164d302e5dc11bd388ac6c69375b

Publish redeem transaction? [y/N] y
Published redeem transaction (2359bd89e2766bbac5971d031d83d7f0164bc0a1a2982154cf3a4a02efe8dbe1)
```

Now that A has withdrawn from the Stakenet contract and revealed the secret, B
must extract the secret from this redemption transaction.  B may watch a block
explorer to see when the Stakenet contract output was spent and look up the
redeeming transaction.

_Party B runs:_
```
$ xsnatomicswap --testnet --rpcuser=<xsnd_user> --rpcpass=<xsnd_pass> extractsecret 0200000001177fdbc6ca5eb1e0901ee9816bc58552e09153c8eb7ba3a21bff51ebb31b2d9600000000f048304502210091e06c35787fec69d791702c3951bc8f20133a648f37109a719803a628036e6c0220467fb5755148be259f13b5e5719fb60af9689faf0789502881e6e9387e3e13c101210290d101672e3870c71bf0b4449c417404bd0ee66d32ba502a2527e9bdc8748f11206489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2514c616382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a91417e112bc28f4952c3f92d6ab5725b119924b1f9d67046c69375bb17576a914b04df4e0be908ed7d815b862d7d2e625a3d87a906888acffffffff01b6c89a3b000000001976a914aba5c61d206d779c90b9da37164d302e5dc11bd388ac6c69375b 053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af
Secret: 6489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2
```

With the secret known, B may redeem from A's Bitcoin contract.

_Party B runs:_
```
$ btcatomicswap --regtest --rpcuser=<bitcoind_user> --rpcpass=<bitcoind_pass> redeem 6382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a9146f9d80594e7e0359bec5058824cd487a3f0c9ba9670431b8385bb17576a914ae53dc0df8d89b2726600cc90dfb9419fe2affea6888ac 0200000001b6e4e374030e6a707c10dd937da630ab837c3c9625f2e833e8e8e82e0e3f5cdf0000000048473044022016bce2c94d65d3e8ae58f4f6af263ca58338ac6a0ccf0d22975be1518fe1507902207e39071e0e0c9c51d867ab82d08caabedf9c341ca5f0cb9c8d7aefca3033940701feffffff02441010240100000017a914791205b2a871d2d7bf8a931eb68f347a20ab8d868700e1f5050000000017a9140b5886633426dacc8329990dd1eb8af5fdcf60978700000000 6489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2
warning: falling back to mempool relay fee policy
Redeem fee: 0.0000033 BTC (0.00001019 BTC/kB)

Redeem transaction (54eedd43904ea50d75784f9b91d4988730406f4942c1d60c7dcdfb31887ec07a):
0200000001ff5f8f8c4321629ffb89fcad374fc8d950ceefc93aa852c37ab973a6e9a07a1301000000ef47304402200404a7ff289f36b06fddea6618f737d4f503b5a8a14f20ffdd920c88274994b3022034d658d1d2d8b05f46cc4a109d31685087eed180381ad4dd8b3abf068116789901210297d4ab57d52521e955f8974403872cf69974c48cd87ab9df4b82549237758d9f206489db3b887677a789b6244f4b03c2957b0d3adc13abe502486fa57aceb102d2514c616382012088a820053c93e0ce63d6e132cf334381a38482ac8981f6bdd178ee493d92b8340997af8876a9146f9d80594e7e0359bec5058824cd487a3f0c9ba9670431b8385bb17576a914ae53dc0df8d89b2726600cc90dfb9419fe2affea6888acffffffff01b6dff505000000001976a914967bade1c2d67e3613ef73e3fe1e63305303278d88ac31b8385b

Publish redeem transaction? [y/N] y
Published redeem transaction (54eedd43904ea50d75784f9b91d4988730406f4942c1d60c7dcdfb31887ec07a)
```

The cross-chain atomic swap is now completed and successful.  This example was
performed on the public Bitcoin regtest and Stakenet testnet blockchains.  For reference,
here are the four transactions involved:

| Description | Transaction |
| - | - |
| Bitcoin contract created by A | [346f4901dff1d69197850289b481f4331913126a8886861e7d5f27e837e0fe88](https://www.blocktrail.com/tBTC/tx/346f4901dff1d69197850289b481f4331913126a8886861e7d5f27e837e0fe88) |
| Decred contract created by B | [a51a7ebc178731016f897684e8e6fbbd65798a84d0a0bd78fe2b53b8384fd918](https://testnet.decred.org/tx/a51a7ebc178731016f897684e8e6fbbd65798a84d0a0bd78fe2b53b8384fd918) |
| A's Decred redemption | [53c2e8bafb8fe36d54bbb1884141a39ea4da83db30bdf3c98ef420cdb332b0e7](https://testnet.decred.org/tx/53c2e8bafb8fe36d54bbb1884141a39ea4da83db30bdf3c98ef420cdb332b0e7) |
| B's Bitcoin redemption | [c49e6fd0057b601dbb8856ad7b3fcb45df626696772f6901482b08df0333e5a0](https://www.blocktrail.com/tBTC/tx/c49e6fd0057b601dbb8856ad7b3fcb45df626696772f6901482b08df0333e5a0) |

If at any point either party attempts to fraud (e.g. creating an invalid
contract, not revealing the secret and refunding, etc.) both parties have the
ability to issue the refund transaction created in the initiate/participate step
and refund the contract.

## Discovering raw transactions

Several steps require working with a raw transaction published by the other
party.  While the transactions can sometimes be looked up from a local node
using the `getrawtransaction` JSON-RPC, this method can be unreliable since the
set of queryable transactions depends on the current UTXO set or may require a
transaction index to be enabled.

Another method of discovering these transactions is to use a public blockchain
explorer.  Not all explorers expose this info through the main user interface so
the API endpoints may need to be used instead.

For Insight-based block explorers, such as the Bitcoin block explorer on
[test-]insight.bitpay.com, the Litecoin block explorer on
{insight,testnet}.litecore.io, and the Decred block explorer on
{mainnet,testnet}.decred.org, the API endpoint `/api/rawtx/<txhash>` can be used
to return a JSON object containing the raw transaction.  For example, here are
links to the four raw transactions published in the example:

| Description | Link to raw transaction |
| - | - |
| Bitcoin contract created by A | https://test-insight.bitpay.com/api/rawtx/346f4901dff1d69197850289b481f4331913126a8886861e7d5f27e837e0fe88 |
| Decred contract created by B | https://testnet.decred.org/api/rawtx/a51a7ebc178731016f897684e8e6fbbd65798a84d0a0bd78fe2b53b8384fd918 |
| A's Decred redemption | https://testnet.decred.org/api/rawtx/53c2e8bafb8fe36d54bbb1884141a39ea4da83db30bdf3c98ef420cdb332b0e7 |
| B's Bitcoin redemption | https://test-insight.bitpay.com/api/rawtx/c49e6fd0057b601dbb8856ad7b3fcb45df626696772f6901482b08df0333e5a0 |

## First mainnet DCR-LTC atomic swap

| Description | Link to raw transaction |
| - | - |
| Decred contract created by A | [fdd72f5841414a9c8b4a188a98a4d484df98f84e1c120e1ed59a66e51e8ae90c](https://mainnet.decred.org/tx/fdd72f5841414a9c8b4a188a98a4d484df98f84e1c120e1ed59a66e51e8ae90c) |
| Litecoin contract created by B | [550d1b2851f6f104e380aa3c2810ac272f8b6918140547c9717a78b1f4ff3469](https://insight.litecore.io/tx/550d1b2851f6f104e380aa3c2810ac272f8b6918140547c9717a78b1f4ff3469) |
| A's Litecoin redemption | [6c27cffab8a86f1b3be1ebe7acfbbbdcb82542c5cfe7880fcca60eab36747037](https://insight.litecore.io/tx/6c27cffab8a86f1b3be1ebe7acfbbbdcb82542c5cfe7880fcca60eab36747037) |
| B's Decred redemption | [49245425967b7e39c1eb27d261c7fe972675cccacff19ae9cc21f434ccddd986](https://mainnet.decred.org/tx/49245425967b7e39c1eb27d261c7fe972675cccacff19ae9cc21f434ccddd986) |
## License

These tools are licensed under the [copyfree](http://copyfree.org) ISC License.
