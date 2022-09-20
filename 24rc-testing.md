# Testing Guide: Bitcoin Core 24.0 Release Candidate

_For feedback on this guide, please visit [#26092](https://github.com/bitcoin/bitcoin/issues/26092)_

This document outlines some of the upcoming Bitcoin Core 24.0 release changes and provides steps to help test them. This guide is meant to be the starting point for experimentation and further testing, but is in no way comprehensive! After running through the steps in this guide, you are encouraged to do your own testing.

This can be as simple as testing the same features in this guide but trying it a different way. Even better, think of features you use regularly and test that they still work as expected in the release candidate. You can also read the [release notes][release notes] to find something not covered in this guide. **This is a great way to be involved with Bitcoin's development and helps keep Bitcoin running smoothly and bug-free! Your help in this endeavor is greatly appreciated.**


## Overview

Changes covered in this testing guide include:

- Observe the new headers pre-synchronization phase during IBD ([#25717](https://github.com/bitcoin/bitcoin/pull/25717))
- Using the GUI to restore a wallet from a backup file ([#471](https://github.com/bitcoin-core/gui/pull/471))
- Peristent settings are now unified between bitcoind and the GUI. ([#15936](https://github.com/bitcoin/bitcoin/pull/15936),[#602](https://github.com/bitcoin-core/gui/pull/602))
- Using the new `migratewallet` RPC to migrate legacy wallets to descriptor wallets. ([#19602](https://github.com/bitcoin/bitcoin/pull/19602))
- Wallet support for watchonly Miniscript descriptors. ([#24148](https://github.com/bitcoin/bitcoin/pull/24148))


## Preparation

### 1. Grab Latest Release Candidate

**Current Release Candidate:** [NA] ([release-notes][release notes])

There are two ways to grab the latest release candidate: pre-compiled binary or source code. The source code for the latest release can be grabbed from here: [NA]

If you want to use a binary, make sure to grab the correct one for your system.

### 2. Compile Release Candidate

**If you grabbed a binary, skip this step**

Before compiling, make sure that your system has all the right dependencies installed. As this guide utilizes the Bitcoin Core GUI, you must compile support for the GUI and have the `qt5` dependency already installed. Since descriptor wallets are now the default, it is recommended that you compile with sqlite, so make sure you have installed the `sqlite3` dependency.

To ensure we build in sqlite and gui support, pass the following options when configuring:
```shell
./autogen.sh
./configure --with-gui=yes --with-sqlite=yes --enable-suppress-external-warnings
```

For more information on compiling from source, here are some guides to compile Bitcoin Core for [UNIX/Linux](https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md), [macOS](https://github.com/bitcoin/bitcoin/blob/master/doc/build-osx.md), [Windows](https://github.com/bitcoin/bitcoin/blob/master/doc/build-windows.md), [FreeBSD](https://github.com/bitcoin/bitcoin/blob/master/doc/build-freebsd.md), [NetBSD](https://github.com/bitcoin/bitcoin/blob/master/doc/build-netbsd.md), and [OpenBSD](https://github.com/bitcoin/bitcoin/blob/master/doc/build-openbsd.md).

### 3. Setting up command line environment

If you plan to use the command line, below are a few environment variable to set.

First, create a temporary data directory
```shell
export DATA_DIR=/tmp/24-rc-test
mkdir $DATA_DIR
```

Next, specify the following paths. For source compiled, start from the root of your release candidate directory and run:

```shell
export BINARY_PATH=$(pwd)/src
export QT_PATH=$(pwd)/src/qt
```

For the downloaded binary, start from the root of the downloaded release candidate (cd ~/bitcoin-24.0rc1, for example) and run:
```shell
export BINARY_PATH=$(pwd)/bin
export QT_PATH=$BINARY_PATH
```

To avoid specifying the data directory (`-datadir=$DATA_DIR`) on each command, bellow are a few extra variables to set.
> :information_source: **Note:** If you are testing on signet, you can also include the `-signet` flag
```shell
alias bitcoind="$(echo $BINARY_PATH)/bitcoind -datadir=$DATA_DIR"
alias cli="$(echo $BINARY_PATH)/bitcoin-cli -datadir=$DATA_DIR"
alias qt="$(echo $QT_PATH)/bitcoin-qt -datadir=$DATA_DIR"
```

The commands throughout the rest of the guide will look like:

```shell
cli [cli args]

# for starting bitcoin-qt
qt [cli args]
```

### 4. Reset testing environment

Between sections in this guide, it's recommended to stop your node and wipe the data directory. You can use the commands provided below.

**Stop node**
```shell
cli stop
```

**Wipe and recreate the directory**
```shell
rm -r $DATA_DIR
mkdir $DATA_DIR
```

**Start node**
```shell
bitcoind -daemon
```

### 5. Testing with a signet faucet

If you are testing on signet, you can use test coins for the tests in this guide. After IBD has completed (this process is relatively quick on signet), you can get some free corn from one of the signet faucets available, e.g. https://signetfaucet.com/. Generate a fresh address when needed, pop it into the faucet, and [watch the signet coins flowing in](https://mempool.space/signet).

When you're finished, please [return your coins to a faucet in need](https://signet.bublina.eu.org/about.html) before stopping your node and cleaning up your datadir.

### 6. Testing using QT

If you're not comfortable with the command line, you can still test all of these changes in the GUI. Althought for some steps you might need to use the integrated RPC console (Window->Console).

To run bitcoin-qt, use:
```shell
qt [cli args]
```
> :bulb: You can use `bitcoin-cli` to talk to `bitcoin-qt` by starting `bitcoin-qt` with the `-server` option.

## Observe the new headers pre-synchronization phase during IBD

Checkpoints protect the disk of a new node from being filled with low-difficulty headers during synchronization — once a checkpoint is reached, no headers branching off before that point are allowed anymore. **However, before a checkpoint is reached, a DoS attack is still possible**.

**The logic for downloading headers from peers has been reworked to address this potential denial-of-service**. This is particularly relevant for nodes starting up for the first time (or for nodes which are starting up after being offline for a long time).

With this new logic, the node downloads headers from a given peer twice:
1. Download (Pre-synchronizing) blockheaders: to verify (using `nMinimumChainWork`) that the header is part of a chain that has sufficiently high work, prior to storing those headers permanently.
2. Redownload (Synchronizing) blockheaders: to fully validate and permanently store the headers.

> :book: ***What is `nMinimumChainWork`?*** A number designed to protect new clients from accepting fake blockchains during initial sync. It's [updated on every release](https://github.com/bitcoin/bitcoin/pull/25946) and defines the minimum amount of total work a chain must have before the client considers it valid.

For this test, we will start by observing the pre-syncing phase during IBD and then verify that the new node is actually able to sync. This can be tested with or without GUI.

### Starting IBD

You can observe the pre-synchronization phase in detail by using the log file. With or without GUI, the logs are located at `$DATA_DIR/debug.log`. For the relevant logs to appear in your `debug.log` file, you'll need to use the `-debug` flag with the `net` category. If you are using bitcoind you'll also be able to observe real-time logging from the console.

For the first part of this test, there is no need to sync the full blockchain. Therefore, you can use the `-stopatheight` flag, for the process to terminate almost as soon as the headers synchronization phase (download&redownload) is over.
```shell
bitcoind -debug=net -stopatheight=100

# for starting bitcoin-qt
qt -debug=net -stopatheight=100
```

> :information_source: **Note**: To be able to observe this pre-synchronizing phase make sure that you are testing a new node (empty datadir), or one that is months behind.


### Observing the logs

Either by looking after the fact, or during IBD, the logs will look like:

> Pre-syncing headers...
```shell
2022-09-18T08:44:41Z [net] Initial headers sync started with peer=3: height=0, max_commitments=4443404, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
2022-09-18T08:44:41Z [net] sending getheaders (101 bytes) peer=3
2022-09-18T08:44:41Z [net] more getheaders (from 00000000dfd5d65c9d8561b4b8f60a63018fe3933ecb131fb37f905f87da951a) to peer=3
2022-09-18T08:44:41Z Pre-synchronizing blockheaders, height: 2000 (~0.28%)
[...]
2022-09-18T08:46:33Z Pre-synchronizing blockheaders, height: 748000 (~99.15%)
[..]
2022-09-18T08:46:34Z [net] Initial headers sync transition with peer=3: reached sufficient work at height=752000, redownloading from height=0
```

> Syncing headers...
```shell
2022-09-18T08:46:34Z [net] Initial headers sync transition with peer=3: reached sufficient work at height=752000, redownloading from height=0
2022-09-18T08:46:34Z [net] sending getheaders (101 bytes) peer=3
2022-09-18T08:46:34Z [net] more getheaders (from 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f) to peer=3
2022-09-18T08:46:35Z [net] received: headers (162003 bytes) peer=3
[...]
2022-09-18T08:46:36Z Synchronizing blockheaders, height: 4041 (~0.56%)
[...]
2022-09-18T08:47:38Z [net] received: headers (162003 bytes) peer=3
2022-09-18T08:47:38Z [net] Initial headers sync complete with peer=3: releasing all at height=752000 (redownload phase)
2022-09-18T08:47:38Z Synchronizing blockheaders, height: 752000 (~99.66%)
```

During the pre-sync phase, you can use the `getpeerinfo` RPC to observe the progress in the presync phase indicated by the new `presynced_headers` field.
```shell
cli getpeerinfo
```
The peer that is currently involved in the presync phase will have the current presynced height set, instead of -1.

> :bulb: Use `grep` to easily filter the response of the RPC e.g. `cli getpeerinfo | grep "presynce_headers"`

**What you could observe during IBD depends on a number of different factors, some beyond our control**. Check ["Observe Further"](#observe-further) for some different scenarios.

### Using the GUI

The GUI should inform you about each step and the progress made on each step.

> Pre-syncing headers...
<img height="400" alt="pre-syncing headers" src="https://user-images.githubusercontent.com/32963518/186409992-2a465617-4c82-4a03-ba77-2e74059decf2.png">

> Syncing headers...
<img height="400" alt="syncing headers" src="https://user-images.githubusercontent.com/32963518/186410190-0a0dc8b8-1543-45cb-b123-0868fc05e464.png">

> Synchronizing with network...
<img height="400" alt="synchronizing with network" src="https://user-images.githubusercontent.com/32963518/186410096-6aee8461-60eb-4cc0-b239-571aa06efab6.png">

### Observe Further

Block headers syncing using only your initial peer is just one of the possible scenarios. If a peer disconnects or if a new block is found during the blockheaders synchronization, what you might see during ["Observing the logs"](#observing-the-logs) might be different.

> :information_source: The log snippets for the examples of this section are part of [this debug.log file](https://gist.github.com/kouloumos/f0507b18dce2acf5a9a9a580e27880eb).

The node starts IBD with only one initially chosen headers-sync peer, **if the peer disconnects it will start again with a different peer**. Below, we start with peer 0, which disconnects after 1400 blocks, hence restarting pre-sync with peer 2.
```shell
2022-09-18T10:08:37Z [net] Initial headers sync started with peer=0: height=0, max_commitments=4443456, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
2022-09-18T10:08:37Z [net] sending getheaders (101 bytes) peer=0
2022-09-18T10:08:37Z [net] more getheaders (from 00000000dfd5d65c9d8561b4b8f60a63018fe3933ecb131fb37f905f87da951a) to peer=0
2022-09-18T10:08:37Z Pre-synchronizing blockheaders, height: 2000 (~0.28%)
[...]
2022-09-18T10:08:38Z [net] more getheaders (from 000000002d9050318ec8112057423e30b9570b39998aacd00ca648216525fce3) to peer=0
2022-09-18T10:08:38Z Pre-synchronizing blockheaders, height: 14000 (~1.95%)
[...]
2022-09-18T10:08:38Z [net] sending getheaders (101 bytes) peer=0
2022-09-18T10:08:38Z [net] more getheaders (from 00000000f914f0d0692e56bd06565ac4de668251b6a29fe0535d1e0031cfd0de) to peer=0
2022-09-18T10:08:38Z [net] socket closed for peer=0
2022-09-18T10:08:38Z [net] disconnecting peer=0
[...]
2022-09-18T10:08:45Z [net] Initial headers sync started with peer=2: height=0, max_commitments=4443456, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
2022-09-18T10:08:45Z [net] sending getheaders (101 bytes) peer=2
2022-09-18T10:08:45Z [net] more getheaders (from 00000000dfd5d65c9d8561b4b8f60a63018fe3933ecb131fb37f905f87da951a) to peer=2
2022-09-18T10:08:45Z Pre-synchronizing blockheaders, height: 2000 (~0.28%)
```

Being in IBD doesn't stop other peers from announcing new blocks (every ~10 minutes) to us. When a node receives a block announcement, **it used to add all announcing peers as additional headers sync peers** and initiate headers sync. The addition of the pre-sync phase with [PR#25717](https://github.com/bitcoin/bitcoin/pull/25717) made this behavior more bandwidth-wasteful.

In an effort to strike a better balance between robustness in the face of stalling peers and bandwidth usage, [PR#25720](https://github.com/bitcoin/bitcoin/pull/25720) changed this logic such that **now only one of the announcing peers is added for headers sync** (one sync for each new block announcement).

**You can observe this change in behavior if a new block is found during the blockheaders synchronization**. Bellow, during header sync with peer 2, we receive a block announcement (via an `inv` message) from peer 3. Observe that from that point onwards we request the same headers from both peers.

```shell
2022-09-18T10:08:46Z [net] sending getheaders (101 bytes) peer=2
2022-09-18T10:08:46Z [net] more getheaders (from 00000000922e2aa9e84a474350a3555f49f06061fd49df50a9352f156692a842) to peer=2
2022-09-18T10:08:46Z Pre-synchronizing blockheaders, height: 4000 (~0.56%)
2022-09-18T10:08:47Z [net] received: inv (37 bytes) peer=3
2022-09-18T10:08:47Z [net] got inv: block 000000000000000000062ad19351f3c4430ee7da2deed4a79157b26e4e60a1de  new peer=3
2022-09-18T10:08:47Z [net] sending getheaders (69 bytes) peer=3
2022-09-18T10:08:47Z [net] getheaders (0) 000000000000000000062ad19351f3c4430ee7da2deed4a79157b26e4e60a1de to peer=3
[...]
2022-09-18T10:08:47Z [net] received: headers (162003 bytes) peer=3
2022-09-18T10:08:47Z [net] Initial headers sync started with peer=3: height=0, max_commitments=4443456, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
[...]
2022-09-18T10:08:49Z [net] sending getheaders (101 bytes) peer=3
2022-09-18T10:08:49Z [net] more getheaders (from 0000000011d1d9f1af3e1d038cebba251f933102dbe181d46a7966191b3299ee) to peer=3
2022-09-18T10:08:49Z Pre-synchronizing blockheaders, height: 12000 (~1.67%)
2022-09-18T10:08:49Z [net] received: headers (162003 bytes) peer=2
2022-09-18T10:08:49Z [net] sending getheaders (101 bytes) peer=2
2022-09-18T10:08:49Z [net] more getheaders (from 0000000011d1d9f1af3e1d038cebba251f933102dbe181d46a7966191b3299ee) to peer=2
```

What if another new block is found? An additional header sync will be initiated with the announcing peer, in this case peer 5.
```shell
2022-09-18T10:13:29Z [net] received: inv (37 bytes) peer=5
2022-09-18T10:13:29Z [net] got inv: block 000000000000000000069020bba8a05880ff4145fe8c67b6eaf829c374c96b03  new peer=5
[...]
2022-09-18T10:13:32Z [net] Initial headers sync started with peer=5: height=0, max_commitments=4443459, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
2022-09-18T10:13:32Z [net] sending getheaders (101 bytes) peer=5
2022-09-18T10:13:32Z [net] more getheaders (from 00000000dfd5d65c9d8561b4b8f60a63018fe3933ecb131fb37f905f87da951a) to peer=5
```

All the previous additional header syncs started from height=0 because we were in the pre-sync phase. If syncing with a peer has moved to the next phase and now a new block is announced, that additional header sync will be initiated from the current height of that phase.
```shell
2022-09-18T10:19:58Z [net] more getheaders (from 0000000000000000000b127f679ccb72c759e843c18792e0bc30e2fddcea4fb2) to peer=2
2022-09-18T10:19:58Z Synchronizing blockheaders, height: 538041 (~71.53%)
2022-09-18T10:19:58Z [net] received: inv (37 bytes) peer=16
2022-09-18T10:19:58Z [net] got inv: block 000000000000000000036232410961670a229292f0d10dba42460d5d0afcac07  new peer=16
[...]
2022-09-18T10:19:58Z [net] more getheaders (from 0000000000000000000d5bb523c5a0075071a35e4dae28ea60cd569e150ff6f8) to peer=2
2022-09-18T10:19:59Z Synchronizing blockheaders, height: 540041 (~71.79%)
2022-09-18T10:19:59Z [net] received: headers (162003 bytes) peer=16
2022-09-18T10:19:59Z [net] sending getheaders (1029 bytes) peer=16
2022-09-18T10:19:59Z [net] more getheaders (540041) to end to peer=16 (startheight:754640)
2022-09-18T10:19:59Z [net] Protecting outbound peer=16 from eviction
2022-09-18T10:19:59Z [net] received: headers (162003 bytes) peer=16
2022-09-18T10:19:59Z [net] Initial headers sync started with peer=16: height=540041, max_commitments=1308493, min_work=00000000000000000000000000000000000000003404ba0801921119f903495e
[...]
# The first syncing peer isn't always the fastest.
2022-09-18T10:20:24Z [net] Initial headers sync complete with peer=16: releasing all at height=752041 (redownload phase)
```

**After you finish testing the pre-sync phase, it is encouraged to test further and verify that a new node is able to sync the full blockchain.**

Once the test is successful, don't forget to [stop your node and clean the datadir](#4-reset-testing-environment).

## Testing the GUI

Since you're starting from a fresh datadir, the first thing you should see is the progress screen of Initial Block Download (IBD). You can simply hide it at the bottom right and proceed with testing.

> :information_source: If you are running on signet, the window title should be "Bitcoin Core - [signet]". See [this guide](https://github.com/bitcoin/bitcoin/issues/24501#issuecomment-1081463097) on how to switch to signet from within the GUI.


### Testing the Restore Wallet option

Restoring a wallet from a backup file in the GUI is now possible! Up until now, you could create a backup but the only option was to restore using the RPC interface. This change makes it easier for non-technical users to restore backups.

You can test this by trying to restore a backup you already have. If no backup is handy, you can always do some "cross testing" by using the resulting backup at the [end of the Testing Migrating Legacy Wallets to Descriptor Wallets](#migrating-a-legacy-wallet) section.

Otherwise, create a new wallet as per your preferred settings, generate addresses, [receive](#5-testing-with-a-signet-faucet)/send transactions. Feel free to experiment. In the end, make sure to create a backup and then restore from it.

> :bulb: A slightly different testing scenario is to [stop your node and clean the datadir](#4-reset-testing-environment) between backup and restore. Or maybe move the backup file and restore to a different machine.

<img height="400" alt="restore wallet" src="https://user-images.githubusercontent.com/18506343/190161082-38a7d963-e686-413c-ac7b-61ec69f6cd59.gif">

After restoring, make sure that your addresses, labels and balances are as expected.

Once the test is successful, don't forget to [stop your node and clean the datadir](#4-reset-testing-environment).

### Testing the unification of settings between bitcoind and the GUI

*Note: For this test you will also need bitcoind.*

Previously, configuration changes made in the bitcoin GUI (such as the pruning setting, proxy settings, UPNP preferences) were ignored by bitcoind. Now, changes made in the GUI take precedence over `bitcoin.conf` values and therefore persist to bitcoind.

> :information_source: You can find the shared persistent settings at `$DATA_DIR/<chain>/settings.json`. That's also where we account for wallets that load on startup.

For this test, we will select a configuration option that is available in the GUI, change that option in the GUI and verify that the change persists to bitcoind. We cannot cover all the options in this guide, it's encouraged to test further using other options.

#### Pruning settings

Pruning is disabled by default. You can confirm that either from the GUI's options or by running the `getblockchaininfo` RPC and looking at the `pruned` and `size_on_disk` fields.

In order to test the persistent settings:
- Enable pruning in the GUI
- Open bitcoind and use `getblockchaininfo` to verify that pruning is now enabled.

A different test is to enforce pruning by adding `prune=x` in `bitcoin.conf`, then change it in the GUI and verify that the GUI option takes precedence in bitcoind over what's set in `bitcoin.conf`.

> :information_source: The `-prune` setting is specified in MiB (2^20 bytes) while the GUI setting is specified in GB (10^9 bytes). So, in various cases rounded values will be displayed.

#### Spending of unconfirmed change

The spending of unconfirmed change is enabled by default. Therefore, creating a transaction and immediately using its unconfirmed change as input for another transaction should be allowed. Disable that from the Options and restart the client to activate changes.

<img width="450" alt="coin selection" src="https://user-images.githubusercontent.com/18506343/190131975-f4f8f8a0-1755-424e-a945-c3f8a99051b3.gif">

**Enable coin control before proceeding** (Options->Wallet->Enable coin control features), otherwise inputs for transactions are automatically selected.

Use the "Send" tab and by clicking the "Inputs..." button select one of your available UTXOs.
<img width="550" alt="coin selection" src="https://user-images.githubusercontent.com/18506343/190129819-05964683-4622-4d40-a5cd-dc3783e59c87.png">

Fill in the rest of the form. Make sure that you will create change by choosing an amount smaller than the value of the input. If you want to send to yourself, use the Receive tab, create a new receiving address and copy it at the "Pay To" field.
<img width="500" alt="send transaction form" src="https://user-images.githubusercontent.com/18506343/190130087-050cab2e-fc82-4718-9fb7-10bde64206d6.png">

> :information_source: At this point you could use a low enough fee to make sure that the transaction will remain unconfirmed until you complete the next step. This is not applicable for signet.

Send the transaction. Observe that the transaction is accounted as "Pending balance" and the unconfirmed output is not available as input for a new transaction.

<img width="500" alt="send transaction" src="https://user-images.githubusercontent.com/18506343/190140271-c8e8e641-2933-4eaa-b27b-2cbc6d2a743a.gif">

In order to test that prohibiting the spending of unconfirmed change persists to bitcoind:
- Open bitcoind and use the `getbalance` RPC. The available balance that returns as a result should match what's shown in the GUI.
- Using the GUI, re-enable the spending of unconfirmed change and observe that the available balance (that now includes the unconfirmed change) is the same between the GUI and bitcoind.

> :information_source: If you are comfortable with bitcoin-cli, you could also try to create a transaction that includes the unconfirmed change.

Once the test is successful, don't forget to [stop your node and clean the datadir](#4-reset-testing-environment).

## Testing Migrating Legacy Wallets to Descriptor Wallets

Descriptor wallets [were introduced with the Bitcoin Core 0.21 release](https://achow101.com/2020/10/0.21-wallets) and became the default wallet type with the Bitcoin Core 23.0 release. Descriptor wallets are ultimately a better design, that's why the [legacy wallet type and BDB format will soon be unsupported](https://github.com/bitcoin/bitcoin/issues/20160), which creates the need for a migration mechanism.

> :flashlight: Want to dive deeper into the changes? Check the [PR Review Club Meeting for PR#19602](https://bitcoincore.reviews/19602)

We are going to test this migration mechanism by using the new `migratewallet` RPC to migrate our legacy wallets. There is an extremely large number of possible configurations for the scripts that Legacy wallets can know about, be watching for, and be able to sign for. This guide cannot cover them all. Therefore **testing configurations other than the testing scenarios of this guide is encouraged and could provide valuable feedback to strengthen this migration mechanism**.


We will test the `migratewallet` RPC by creating a new legacy wallet and then migrating it. It is not required to have funds in it as the migration deals with keys and scripts, not really transactions. Even without funds, you can check that the keys and scripts were migrated correctly.

### What to expect after the migration

A new descriptor wallet creates 8 descriptors by default, 2 (external, internal) for each supported script type:
- P2PKH: `pkh([<fingerprint>/44'/1'/0']<xpub>/{0,1}/*)#<checksum>`
- P2SH(P2WPKH): `sh(wpkh([<fingerprint>/49'/1'/0']<xpub>/{0,1}/*))#<checksum>`
- P2WPKH: `wpkh([<fingerprint>/84'/1'/0']<xpub>/{0,1}/*)#<checksum>`
- P2TR: `tr([<fingerprint>/86'/1'/0']<xpub>/{0,1}/*)#<checksum>`

> :information_source: You can verify that with the `listdescriptors` RPC on a new descriptor wallet. For more information on output descriptors, see [doc/descriptors.md][descriptors.md].

A legacy wallet with an HD chain has an HD seed. This seed is hashed according to the BIP32 specification to become the BIP32 master key which everything else is then derived from. We need to associate those addresses with a descriptor. Because of that, in addition to the above, the new migrant wallet is expected to have:
- A range combo descriptor for the external addresses: `combo(<xpub>/0'/0'/*')#<checksum>`
- A range combo descriptor for the internal addresses: `combo(<xpub>/0'/1'/*')#<checksum>`
- A combo descriptor for the hd seed: `combo(<hdseed>)#<checksum>`

> :information_source: The HD seed is a valid key that could receive Bitcoin to it even though it's corresponding addresses would never be given out.

For wallets containing non-HD keys, each key will have its own combo descriptor.

After a successful migration, all the above descriptors will be part of a newly created Descriptor wallet named as the original wallet.

### Preparing a Legacy wallet

Create a legacy test-wallet. This is the wallet we will migrate [later](#migrating-a-legacy-wallet) in the guide.
```shell
cli -named createwallet wallet_name=legacy descriptors=false
alias wallet="$(echo $BINARY_PATH)/bitcoin-cli -datadir=$DATA_DIR -rpcwallet=legacy"
```
Create a second helper-wallet. This one is quite helpful in allowing more configurations to be tested by exporting scripts and importing them to our test-wallet.
```shell
cli -named createwallet wallet_name=legacy_helper descriptors=false
alias helper="$(echo $BINARY_PATH)/bitcoin-cli -datadir=$DATA_DIR -rpcwallet=legacy_helper"
```

Generate receiving addresses for different address types (P2PKH, P2SH(P2WPKH), P2WPKH) to verify that after migration the wallet still has them.
```shell
legacy=$(wallet getnewaddress "my-P2PKH" legacy)
p2sh_segwit=$(wallet getnewaddress "my-P2SH(P2WPKH)" p2sh-segwit)
bech32=$(wallet getnewaddress "my-P2WPKH" bech32)
```

#### non-HD keys

For wallets containing non-HD keys, each key will have its own combo descriptor.

To test for this behavior, we will use the helper-wallet.
```shell
non_HD_address=$(helper getnewaddress)
non_HD_key=$(helper dumpprivkey $non_HD_address)
wallet importprivkey $non_HD_key "non-HD"
```

#### Watch-only scripts

Because Descriptor wallets do not support having private keys and watch-only scripts, there may be up to two additional wallets created after migration. The one that contains all of the watchonly scripts will be named `<name>_watchonly`.

> :information_source: Legacy wallets allow for private keys and watch-only to mix together because it used to not be possible to have different wallet files for different purposes. [Multiwallet is relatively recent](https://github.com/bitcoin/bitcoin/blob/master/doc/release-notes/release-notes-0.15.0.md#multi-wallet-support).

To test for this behavior, we will use the helper-wallet.
```shell
watch_address=$(helper getnewaddress)
wallet importaddress $watch_address "watch-me"
```

#### Solvable scripts

Solvable is anything we have any (public) keys for and we know how to spend it (e.g. a multisig for which we have one public key, and we know the other two because we have the full redeemscript). Because the user is not watching the corresponding P2(W)SH scripts when the wallet is a legacy wallet, it does not make sense to have them be in the `<name>_watchonly` wallet. Therefore, they are part of a second additional wallet named `<name>_solvables`.

> :information_source: The solvables wallet can update a PSBT with the UTXO, scripts, and pubkeys, for inputs that spend any of the scripts it contains. It basically backups the `redeemscript`.

To test for this behavior, we will again use the helper-wallet to create a multisig with some keys that do not belong to this wallet.
```shell
address1=$(helper getnewaddress)
not_my_pubkey=$(helper getaddressinfo $address1 | jq -r ".pubkey")
my_address=$(wallet getnewaddress)

multisig_address=$(wallet addmultisigaddress 1 '''["'$my_address'", "'$not_my_pubkey'"]''' "multisig" | jq -r ".address")
```
> :bulb: Multisigs are special, `addmultisigaddress` adds a `redeemscript` so it's **__solvable__**, but it's **__not__ marked watch-only**. The net-effect of that is that we see the full script with `getaddressinfo`, __the wallet ignores transactions to it__, but we can sign them. This may be useful if you're a co-signer and you don't want those transactions to show up.

### Migrating a Legacy wallet

Before migrating, use the `getaddressinfo` RPC and observe how the wallet understands your newly created addresses.
```shell
wallet getaddressinfo $legacy
```
Do the same for `p2sh_segwit`, `bech32`, `non_HD_address`, `watch_address`, `multisig_address`. Pay attention to the `ismine`, `solvable`, `iswatchonly` and `labels` fields.

Run the migration and wait. If you are not running `bitcoind` as a daemon you can also see the related logging in your console.
```shell
wallet migratewallet
```

If you followed all the steps, at the end of the migration you should see something like:
```shell
{
  "wallet_name": "legacy",
  "watchonly_name": "legacy_watchonly",
  "solvables_name": "legacy_solvables",
  "backup_path": "/tmp/24-rc-test/signet/wallets/legacy/legacy-1663017338.legacy.bak"
}
```
This means that the migration was successful!

Things to observe:
- The new descriptor wallet has 12 descriptors, as explained in the "[What to expect after the migration](#what-to-expect-after-the-migration)" section.
- `legacy`, `p2sh_segwit`, `bech32` and `non_HD_address` belong to the `legacy` wallet.
- `watch_address` belongs to the `legacy_watchonly` wallet.
-  `multisig_address` belongs to the `legacy_solvables` wallet.

> :information_source: If you haven't already, why not use your legacy wallet backup for [testing the GUI's Restore Wallet option](#testing-the-restore-wallet-option)?

Once the test is successful, don't forget to [stop your node and clean the datadir](#4-reset-testing-environment).

### Further testing

By now, you have a better grasp of what goes into the migration process. Remember, this is meant to get you started on testing. Here are a few different ways to test the migration mechanism.

#### Use the GUI

The migration is not yet supported for the GUI but it is still possible (and encouraged) to repeat the above steps from within the GUI and then use the integrated RPC console to run the migration.

#### Migrate one of your own legacy wallets

If you are using legacy wallets, this is a good opportunity to test further by migrating one of your own legacy wallets. You don't have to worry about loss of funds as the migration process will create a backup of the wallet before migrating. In any case, feel free to create a backup beforehand using the `backupwallet` RPC.

## Testing watch-only support for Miniscript descriptors

The descriptor language has been extended to allow a Descriptor Wallet to handle tracking for a larger variety of scripts. You can now import Miniscript descriptors for P2WSH in a watchonly wallet to track coins, but you can't spend from them using the Bitcoin Core wallet [yet](https://github.com/bitcoin/bitcoin/pull/24149).

> :flashlight: Want to dive deeper into the changes? Check the [PR Review Club Meeting for PR#24148](https://bitcoincore.reviews/24148)

For this test, we will construct a Miniscript output descriptor, import it into our descriptor wallet, derive new addresses and detect funds sent to them.

### Miniscript

Miniscript is a language for writing (a subset of) Bitcoin Scripts in a structured way, enabling analysis, composition, generic signing and more, allowing us to go further than just [the few simple templates we have now in descriptors](https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md#features).  Miniscript excites a lot of people. That being said, don't expect to fully understand Miniscript as part of this guide. Find more about Miniscript on the [reference website][miniscript homepage].

There are four steps to go from our desired wallet locking conditions to the actual Miniscript output descriptor that we will import to our descriptor wallet:
1. We start from our desired locking conditions.
    > I want to unconditionally lock the UTXO for 21 blocks, and then require a 2-of-3 multisig from Alice, Bob, or Carol.
2. We then write them in a policy. Policy is intended to simplify designing Scripts for humans.
    ```shell
    and(thresh(2,pk(key_a),pk(key_b),pk(key_c)),after(21))
    ```
3. The policy language is then compiled into Miniscript. The resulting textual notation aka Miniscript exists to permit including Miniscript expressions inside output descriptors and easily communicate them.
    ```shell
    and_v(v:multi(2,key_a,key_b,key_c),older(21))
    ```
4. The `wsh()` output descriptor is the one extended with Miniscript support. That's why we can  only import Miniscript descriptors for P2WSH.
    ```shell
    wsh(and_v(v:multi(2,key_a,key_b,key_c),older(21)))
    ```

> :bulb: Miniscript doesn't "compile" to Script (maybe the word works but it can lead to confusion), each Miniscript fragment maps to a specific Script.

### Construct Miniscripts

In the previous section, `key_*` either refers to a fixed public key in hexadecimal notation, or to an xpub. Both can be used inside a descriptor. For more information on output descriptors, see [doc/descriptors.md][descriptors.md].

In this guide we will use [miniscript.fun][miniscript.fun], an easy-to-use graphical playground for Miniscript, to create the spending policy for the example in the previous section. You can also construct Miniscripts using the [Miniscript homepage][miniscript homepage] (does not create keys) and https://min.sc/ (more advanced playground).

<img width="707" alt="miniscript" src="https://user-images.githubusercontent.com/18506343/189630003-44d8d60f-3156-48f3-b701-1023ff1cdf7b.png">

> I want to unconditionally lock the UTXO for 21 blocks, and then require a 2-of-3 multisig from Alice, Bob, or Carol. ([miniscript.fun-src][miniscript-example])

Assign the resulting output descriptor to a variable.
```shell
descriptor="wsh(and_v(v:multi(2,[67e54752]tpubD6NzVbkrYhZ4YRJ9MTbmErYTvHdyph7n12fQvuBTozwGQC2LtT8aKbLGMs2jWC11Uj7dXsScu6bDyLdNPLFumAENDNDnaXA3p679HVimacv/0/*,[a9e03770]tpubD6NzVbkrYhZ4Ygoy6im7VLabzegPPSHVD4bY2q3jNkZumP48sK6EZoWuSwAEh4AsimdSXrrjxpuEWSD3k5P4WPcBVWJEVBuuCmMckhd5MbH/0/*),after(21)))#juwzr4jc"
```

> :information_source: **Note:** If you decide to manually construct a descriptor, you will need a checksum. You can get that using the `getdescriptorinfo` RPC.

> :bulb: For testing purposes, you can use https://iancoleman.io/bip39/ to generate BIP39 Mnemonics, and xpubs for the miniscript.fun playground

### Testing Miniscripts

Descriptor wallets **do not allow** both private keys and watch-only addresses in a single wallet, they can either always have private keys, or never have private keys. That's why we need to create a descriptor wallet with disabled private keys for this test.
```shell
cli -named createwallet wallet_name=miniscript_wo disable_private_keys=true
```

We are going to test by importing the Miniscript descriptor that we have previously constructed. **This guide tests against a specific Miniscript output descriptor but it's encouraged to follow the guide with your own spending policies.**
```
cli importdescriptors '''[{"desc": "'$descriptor'", "active": true, "timestamp": "now"}]'''
```

Derive a new address from the new watch-only descriptor.
```
watch_address=$(cli getnewaddress)
```

[Send coins](#5-testing-with-a-signet-faucet) to the `watch_address` and confirm that the wallet detects the funds.
```
cli listtransactions
```

**BONUS change covered**: Received coins are now tracked by parent descriptors ([#25504](https://github.com/bitcoin/bitcoin/pull/25504))

Did you notice the `parent_descs` field in the result of the RPC command you just executed? Bitcoin Core allows us to track coins for multiple descriptors in a single wallet. This new field makes it easy to link a coin with a descriptor. In this case, you can see that the `parent_desc` contains the imported Miniscript descriptor.

Once the test is successful, don't forget to [stop your node and clean the datadir](#4-reset-testing-environment).

## The Most Important Step

Thank you for your help in making Bitcoin as robust as it can be. Please remember to add a comment on v24.0 testing issue detailing:

1. Your hardware and operating system
2. Which release candidate you tested and whether you compiled from source or used a binary (e.g. 24.0rc1 binary or 24.0rc1 compiled from source)
3. What you tested
4. Any other relevant findings

Don't be shy about leaving a comment even if everything worked as expected! We want to hear from you and so it doesn't count unless you leave some feedback.

Thanks for your contribution and for taking the time to make Bitcoin awesome. For feedback on this guide, please visit [#26092](https://github.com/bitcoin/bitcoin/issues/26092)

[release notes]: https://github.com/bitcoin-core/bitcoin-devwiki/wiki/24.0-Release-Notes-draft
[descriptors.md]: https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
[miniscript homepage]: https://bitcoin.sipa.be/miniscript/
[miniscript.fun]: https://miniscript.fun/
[miniscript-example]: https://miniscript.fun/#/full/eyJpZCI6ImRlbW9AMC4xLjAiLCJub2RlcyI6eyIyIjp7ImlkIjoyLCJkYXRhIjp7Im1uZW1vbmljIjoiY3Jhc2ggZmF0YWwgaG9sbG93IHRoYW5rIHN3YWxsb3cgc3VibWl0IHRhdHRvbyBwb3J0aW9uIGNvZGUgZm9hbSBtYXRoIGZvcmNlIiwicGFzc3dvcmQiOiIiLCJkZXJpdmF0aW9uIjoiIn0sImlucHV0cyI6e30sIm91dHB1dHMiOnsia2V5Ijp7ImNvbm5lY3Rpb25zIjpbeyJub2RlIjozLCJpbnB1dCI6ImtleSIsImRhdGEiOnt9fV19fSwicG9zaXRpb24iOlstNjgyLjk3ODA3NzAxNzExNTUsLTk0LjM1NDg3NDg2ODg2NzM4XSwibmFtZSI6IkJJUDM5In0sIjMiOnsiaWQiOjMsImRhdGEiOnt9LCJpbnB1dHMiOnsia2V5Ijp7ImNvbm5lY3Rpb25zIjpbeyJub2RlIjoyLCJvdXRwdXQiOiJrZXkiLCJkYXRhIjp7fX1dfX0sIm91dHB1dHMiOnsia2V5Ijp7ImNvbm5lY3Rpb25zIjpbXX19LCJwb3NpdGlvbiI6Wy0zNTQuMzg1MjYxNjAxMzIyNiwtMjYuMTcwMDU4OTc5NTg0MzIyXSwibmFtZSI6IktleSJ9LCI2Ijp7ImlkIjo2LCJkYXRhIjp7fSwiaW5wdXRzIjp7InBvbCI6eyJjb25uZWN0aW9ucyI6W3sibm9kZSI6MTcsIm91dHB1dCI6InBvbCIsImRhdGEiOnt9fV19fSwib3V0cHV0cyI6eyJkZXNjIjp7ImNvbm5lY3Rpb25zIjpbXX19LCJwb3NpdGlvbiI6WzU1Ni45NTE5NTcxNzE2NDE0LDEyNC42MTE0ODQ0NTUzODg2Ml0sIm5hbWUiOiJEZXNjcmlwdG9yIn0sIjgiOnsiaWQiOjgsImRhdGEiOnsibnVtIjoyMX0sImlucHV0cyI6e30sIm91dHB1dHMiOnsicG9sIjp7ImNvbm5lY3Rpb25zIjpbeyJub2RlIjoxNywiaW5wdXQiOiJwb2wxIiwiZGF0YSI6e319XX19LCJwb3NpdGlvbiI6Wy03OC41NTk4NTE4NjQzMzg0OCwtNjUuNTQ3MDM1NjYxMDU4NDZdLCJuYW1lIjoiQWZ0ZXIifSwiMTAiOnsiaWQiOjEwLCJkYXRhIjp7fSwiaW5wdXRzIjp7ImtleSI6eyJjb25uZWN0aW9ucyI6W3sibm9kZSI6MTIsIm91dHB1dCI6ImtleSIsImRhdGEiOnt9fV19fSwib3V0cHV0cyI6eyJrZXkiOnsiY29ubmVjdGlvbnMiOlt7Im5vZGUiOjE2LCJpbnB1dCI6InBvbGljaWVzIiwiZGF0YSI6e319XX19LCJwb3NpdGlvbiI6Wy0zNDEuMjAzMDYwNTIzMDI2Myw0OTYuNTM3NjQ5NTU1NjA3NF0sIm5hbWUiOiJLZXkifSwiMTIiOnsiaWQiOjEyLCJkYXRhIjp7Im1uZW1vbmljIjoic3BlbGwgYm9yZGVyIGV4aGF1c3QgeWFyZCBlY29ub215IGZvc3NpbCBhcm15IGd1ZXNzIHR5cGljYWwgYm9tYiBhY2hpZXZlIGFtb3VudCJ9LCJpbnB1dHMiOnt9LCJvdXRwdXRzIjp7ImtleSI6eyJjb25uZWN0aW9ucyI6W3sibm9kZSI6MTAsImlucHV0Ijoia2V5IiwiZGF0YSI6e319XX19LCJwb3NpdGlvbiI6Wy02NzYuNzgwMTk0NTc5Mzg1NCw0MzMuMTY2OTMxMDkzMDg4MjRdLCJuYW1lIjoiQklQMzkifSwiMTQiOnsiaWQiOjE0LCJkYXRhIjp7Im1uZW1vbmljIjoiY2FtZXJhIHVuYWJsZSB0cmljayBpbmNvbWUgd2lzZG9tIG1peCByZW1pbmQgb3N0cmljaCBvZmZpY2UgbWFzdGVyIHN3YW1wIGxvdHRlcnkifSwiaW5wdXRzIjp7fSwib3V0cHV0cyI6eyJrZXkiOnsiY29ubmVjdGlvbnMiOlt7Im5vZGUiOjE1LCJpbnB1dCI6ImtleSIsImRhdGEiOnt9fV19fSwicG9zaXRpb24iOlstMTAyNi4wNzIwNjE4NDk4NzY3LDI2NS41NzE0OTg4ODQxOTIzXSwibmFtZSI6IkJJUDM5In0sIjE1Ijp7ImlkIjoxNSwiZGF0YSI6e30sImlucHV0cyI6eyJrZXkiOnsiY29ubmVjdGlvbnMiOlt7Im5vZGUiOjE0LCJvdXRwdXQiOiJrZXkiLCJkYXRhIjp7fX1dfX0sIm91dHB1dHMiOnsia2V5Ijp7ImNvbm5lY3Rpb25zIjpbeyJub2RlIjoxNiwiaW5wdXQiOiJwb2xpY2llcyIsImRhdGEiOnt9fV19fSwicG9zaXRpb24iOlstMzYyLjQzMzExNjAxNTIxNDIsMjM3LjE3NDY1OTM5MzAyOTRdLCJuYW1lIjoiS2V5In0sIjE2Ijp7ImlkIjoxNiwiZGF0YSI6eyJ0aHJlc2giOjJ9LCJpbnB1dHMiOnsicG9saWNpZXMiOnsiY29ubmVjdGlvbnMiOlt7Im5vZGUiOjEwLCJvdXRwdXQiOiJrZXkiLCJkYXRhIjp7fX0seyJub2RlIjoxNSwib3V0cHV0Ijoia2V5IiwiZGF0YSI6e319XX19LCJvdXRwdXRzIjp7InBvbCI6eyJjb25uZWN0aW9ucyI6W3sibm9kZSI6MTcsImlucHV0IjoicG9sMiIsImRhdGEiOnt9fV19fSwicG9zaXRpb24iOlstNDIuNzg0Nzc3NTQ4NTc2NzQsMjUyLjIyMjU2NzM5MjgyMjAxXSwibmFtZSI6IlRocmVzaG9sZCJ9LCIxNyI6eyJpZCI6MTcsImRhdGEiOnt9LCJpbnB1dHMiOnsicG9sMSI6eyJjb25uZWN0aW9ucyI6W3sibm9kZSI6OCwib3V0cHV0IjoicG9sIiwiZGF0YSI6e319XX0sInBvbDIiOnsiY29ubmVjdGlvbnMiOlt7Im5vZGUiOjE2LCJvdXRwdXQiOiJwb2wiLCJkYXRhIjp7fX1dfX0sIm91dHB1dHMiOnsicG9sIjp7ImNvbm5lY3Rpb25zIjpbeyJub2RlIjo2LCJpbnB1dCI6InBvbCIsImRhdGEiOnt9fV19fSwicG9zaXRpb24iOlsyNTEuNjQ5NzA2MzcyNTA5MiwxMTEuMzMyODE0NzQ5MjcxNTVdLCJuYW1lIjoiQW5kIn19LCJjb21tZW50cyI6W3sidGV4dCI6IkFsaWNlIiwicG9zaXRpb24iOlstMTA4OS4zODA5NDUzOTI1OTM3LDIwMC4yMjM3Njc5NzM4MjYxN10sImxpbmtzIjpbMTRdLCJ0eXBlIjoiZnJhbWUiLCJ3aWR0aCI6MjgwLCJoZWlnaHQiOjQxNS45OTk5OTk5OTk5OTk5NH0seyJ0ZXh0IjoiQm9iIiwicG9zaXRpb24iOlstNzQzLjgyNTA0MTQzMzI4MjQsLTE1OS41NDIwOTc0MjI1MzU2OF0sImxpbmtzIjpbMl0sInR5cGUiOiJmcmFtZSIsIndpZHRoIjoyODAsImhlaWdodCI6NDE2fSx7InRleHQiOiJDYXJvbCIsInBvc2l0aW9uIjpbLTczOS4yMDk4NjkwODk1MTg0LDM2OS4yMDIxMjg0MjE0MTQxXSwibGlua3MiOlsxMl0sInR5cGUiOiJmcmFtZSIsIndpZHRoIjoyODAsImhlaWdodCI6NDE2fV0sIm5ldHdvcmsiOiJzaWduZXQifQ==
