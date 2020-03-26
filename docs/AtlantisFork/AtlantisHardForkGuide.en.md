### "Atlantis" Hard Fork Guide

#### 0. Overview

CoinEx Chain will have a hard fork on March 30, 2020, with the code name "Atlantis". This hard fork will use a genesis.json file for data migration. The data directory of the old chain, which stores historical blocks and latest state, is no longer needed by the new chain. The new chain will use a new data directory and have a new Chain-ID.

The overall flow of this hard fork is:

1. When the old chain reaches a pre-defined height (or when it exceeds this height by several blocks), we stop the cetd process of the old chain.
2. Use the cetd of the old chain to export the latest state at the pre-defined height, which will be stored in a file named genesis.exported.json 
3. Use the cetd of the new chain to process this genesis.exported.json file, and we'll get the genesis.json for the new chain.
4. Use this genesis.json file to start cetd of the new chain.

This article introduces the whole hard fork flow. You can find some linux commands here, which can be used for "copy&paste". But, please note this article may be updated any time before the final hard fork. So, before you copy&paste, please make sure this article is up-to-date, and please reload your browser page if necessary.



#### 1. Stop the old chain and backup its data directory

When the old chain reaches a pre-defined height (or when it exceeds this height by several blocks), we stop the cetd process of the old chain. The following command can show the latest excuted block:

```
/path/to/old/cetcli status --indent
```

Please you should use the cetcli of the old chain.

If you used systemctl or supervisor to start cetd as a daemon, please use it to stop cetd. Do not just kill cetd.

After the hard fork, the data directory of the old chain is no longer needed. BUT, to be 100% safe, **Please archive the data directory of the old chain and download them to a local computer or mobile hard disk. Also, please DO NOT delete the data directory of the old chain in short term.**

If you did not change it, then the directory of the old chain would be: $HOME/.cetd/

The ED25519 private key file used by validators for voting, is VERY VERY IMPORTANT, so please backup it up with multiple copy. If you did not change it, then it would be: $HOME/.cetd/config/priv_validator_key.json 



#### 2. Generate the genesis.json file used by the new chain (optional)

This step is not mandatory. Because you can also use the genesis file generated by other validator. But to make sure the genesis file sent from other validator is correct, you'd better to generate the genesis file by ourself, or at least check other's file against your file.

Please use cetd of the old chain to export the chain's state, which will be stored in genesis.exported.json file:

```bash
/path/to/old/cetd export --height=<predefined-height> --for-zero-height > genesis.exported.json
```

Then use the cetd of the new chain to process genesis.exported.json file, and you can get a genesis.json file:

```bash
/path/to/new/cetd migrate genesis.exported.json --genesis-block-height=<predefined-height> --output genesis.json 

```

You can use the following commands to compare the genesis.json file from other validator and yourself:

```bash
jq -S . genesis.json.myself > 1
jq -S . genesis.json.others > 2
diff 1 2
```

Here jq is used to sort json files to make them readable.



#### 3. Use genesis.json to start a new chain

##### 3.1. Download the binary file of the new chain

First define some environmenal variables:

```bash
export ARTIFACTS_BASE_URL=https://raw.githubusercontent.com/coinexchain/testnets/master/coinexdex-test-upgrade
export CETD_URL=${ARTIFACTS_BASE_URL}/linux_x86_64/cetd
export CETCLI_URL=${ARTIFACTS_BASE_URL}/linux_x86_64/cetcli
export CHECK_SH=${ARTIFACTS_BASE_URL}/dex2_check.sh
export GENESIS_URL=${ARTIFACTS_BASE_URL}/genesis.json
export SHA256_CHECKSUM_URL=${ARTIFACTS_BASE_URL}/sha256.sum
export PUBLIC_IP=123.36.28.137
export RUN_DIR=/home/ubuntu/data-cetd-dex2
```
Please not the above `PUBLIC_IP` and `RUN_DIR` have example values, which may not be suitable for your case, so you must change them according to your needs.

Then you can download the files:
```bash
mkdir ${RUN_DIR}
cd ${RUN_DIR}
curl ${CETD_URL} >  cetd
curl ${CETCLI_URL} > cetcli
curl ${CHECK_SH} > dex2_check.sh
curl ${GENESIS_URL} > genesis.json
chmod a+x cetd cetcli
```
The above genesis.json is the genesis file generated and uploaded by some other validator.



#### 3.2. Make new data directory

1. `${RUN_DIR}/cetd init moniker --chain-id=coinexdex2 --home=${RUN_DIR}/.cetd`
2. Copy the downloaded genesis.json file to the data directory: `cp genesis.json ${RUN_DIR}/.cetd/config`
3. If you are a validator

    *   Copy the ED25519 private key file `priv_validator_key.json` of the old chain to the data direction of the new chain. It should be at `${RUN_DIR}/.cetd/config`

4. Configure the external IP of this node

   *	`ansible localhost -m ini_file -a "path=${RUN_DIR}/.cetd/config/config.toml section=p2p option=external_address value='\"tcp://${PUBLIC_IP}:26656\"' backup=true"`
   *   `ansible` is used here. If you did not install it, Please intall it according to: [ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)

5. Verify the execuatbles and genesis.json file, etc:
   *  `bash dex2_check.sh`

    
#### 3.3. Use nohup to start a new node

First define a list of seed nodes. Seed nodes are the several nodes who start firstly in the hard fork, which are not known until the hard fork happens.


```bash
export CHAIN_SEEDS=<the-chain-seeds-which-would-be-known-then>
```

Then use nohup to start cetd:

```bash
nohup ${RUN_DIR}/cetd start --home=${RUN_DIR}/.cetd --minimum-gas-prices=20.0cet --p2p.seeds=${CHAIN_SEEDS} &> cetd.log &
```



#### 3.4. Configure cetd to start automatically using systemctl or supervisor

When cetd has been running correctly for about 1~2 hours, you can kill the cetd process and switch to systemctl or supervisor, which can be configure cetd to start automatically.

First add the seed list into config.toml:

```bash
ansible localhost -m ini_file -a "path=${RUN_DIR}/.cetd/config/config.toml section=p2p option=seeds value='\"${CHAIN_SEEDS}\"' backup=true"
```

Then you can use your favorite tool such as systemctl or supervisor, to make cetd as an automatically-starting deamon.