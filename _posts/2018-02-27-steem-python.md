---
layout: post
title: steem-python 源码分析
categories: [python, steem, blockchain]
description: python steem-python steem blockchain
keywords: python steem-python steem blockchain
---

[steem-python](https://github.com/steemit/steem-python)是[steem](https://github.com/steemit/steem)项目的
官方python库(client)。需要python3.5以上的运行环境。`steem`作为我的入门区块链的第一项目，我会逐步分析该项
目，并不断更新该文章。

`steem-python`该项目总的来说，比较粗糙，在运行的过程中，代码随时会原地爆炸，基本上用的过程就是一个debug、
熟悉代码的过程。通常需要分析代码才能确定是不是命令输入的参数有问题。

## steem
WHAT/WHY/HOW`steem`的相关内容先略过，随后有时间再补充，基本就是其白皮书中讲的那些。

## 安装
其编译安装比较简单，按照[官方文档](https://github.com/steemit/steem-python#installation-with-pipenv-recommended)
操作即可：
```shell
> pip3 install --upgrade --user pipenv
> git clone https://github.com/steemit/steem-python.git
> cd steem-python
> pipenv install --three --dev
> pipenv install .
```

MacOS上在编译时需要注意设计[openssl的引用路径](https://github.com/steemit/steem-python#homebrew-build-prereqs)：
```shell
> brew install openssl
> export CFLAGS="-I$(brew --prefix openssl)/include $CFLAGS"
> export LDFLAGS="-L$(brew --prefix openssl)/lib $LDFLAGS"
```

在编译完成后，我们会得到`steempy`、`steemtail`两个可执行脚本，这两个脚本会出现在python运行环境的`bin`
目录下，如果找不到，应该是`PATH`设置的问题。

## steempy
`steempy`是`steem`的一个命令行客户端，其代码的入口为[legacyentry](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/cli.py#L32-L1314)。
其支持的子命令有：
```
    set                 Set configuration
    config              Show local configuration
    info                Show basic STEEM blockchain info
    changewalletpassphrase
                        Change wallet password
    addkey              Add a new key to the wallet
    parsewif            Parse a WIF private key without importing
    delkey              Delete keys from the wallet
    getkey              Dump the privatekey of a pubkey from the wallet
    listkeys            List available keys in your wallet
    listaccounts        List available accounts in your wallet
    upvote              Upvote a post
    downvote            Downvote a post
    transfer            Transfer STEEM
    powerup             Power up (vest STEEM as STEEM POWER)
    powerdown           Power down (start withdrawing STEEM from steem POWER)
    powerdownroute      Setup a powerdown route
    convert             Convert STEEMDollars to Steem (takes a week to settle)
    balance             Show the balance of one more more accounts
    interest            Get information about interest payment
    permissions         Show permissions of an account
    allow               Allow an account/key to interact with your account
    disallow            Remove allowance an account/key to interact with your
                        account
    newaccount          Create a new account
    importaccount       Import an account using a passphrase
    updatememokey       Update an account's memo key
    approvewitness      Approve a witnesses
    disapprovewitness   Disapprove a witnesses
    sign                Sign a provided transaction with available and
                        required keys
    broadcast           broadcast a signed transaction
    orderbook           Obtain orderbook of the internal market
    buy                 Buy STEEM or SBD from the internal market
    sell                Sell STEEM or SBD from the internal market
    cancel              Cancel order in the internal market
    resteem             Resteem an existing post
    follow              Follow another account
    unfollow            unfollow another account
    setprofile          Set a variable in an account's profile
    delprofile          Set a variable in an account's profile
    witnessupdate       Change witness properties
    witnesscreate       Create a witness
```
在后面的内容中可能会讲到一些命令。

先讲几个全局参数，上面所有命令都可以设置这些参数：
```
--node ${node url}  执行命令所连接的网络节点，指你执行的这条命令发送到哪个网络，
                    是公共网络还是测试网络，如果缺省，默认是https://steemd.steemit.com
--no-broadcast, -d  
  --no-wallet, -p       Do not load the wallet
  --unsigned, -x        Do not try to sign the transaction
  --expires EXPIRES, -e EXPIRES
                        Expiration time in seconds (defaults to 30)
  --verbose VERBOSE, -v VERBOSE
                        Verbosity
```

#### 数据存储
`steempy`每个命令的操作都是有上下文衔接的，其数据存储使用的是[`sqlite3`](https://docs.python.org/3/library/sqlite3.html)
该数据库存在本地的：
```
Mac OS X:               ~/Library/Application Support/steem/steem.sqlite
Unix:                   ~/.local/share/steem/steem.sqlite    # or in $XDG_DATA_HOME, if defined
Win XP (not roaming):   C:\Documents and Settings\<username>\Application Data\Steemit Inc\steem\steem.sqlite
Win XP (roaming):       C:\Documents and Settings\<username>\Local Settings\Application Data\Steemit Inc\steem\steem.sqlite
Win 7  (not roaming):   C:\Users\<username>\AppData\Local\Steemit Inc\steem\steem.sqlite
Win 7  (roaming):       C:\Users\<username>\AppData\Roaming\Steemit Inc\steem\steem.sqlite
```

`steempy`的每个命令都可能导致本地的数据库发生变更。

与之相关的两个类分别是[`Configuration`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steembase/storage.py#L194-L330)和[`Key`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steembase/storage.py#L92-L191)
实现都比较简单，就不展开讲了。

* `Configuration`主要用于存储本地配置，比如client接入公网的节点地址、博文格式、默认投票权重等等。位于`config`表
* `Key`主要用于存储用户的公钥、私钥。位于`keys`表。该类主要用于钱包等功能。

#### 钱包
用户的私钥通常通过[`BIP38`](https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki)加密后存储于
本地的数据库中。本地`steempy`首次运行时，会随机生成一个`MasterPassword`，然后用用户输入的密码作为加密种子
对`MasterPassword`进行加密，然后将`加密后的MasterPassword`存储于本地的`Configuration`中，在`Configuration`
中的key为`encrypted_master_password`。

其加密逻辑为，看起来很复杂的样子：
```
MasterPassword = 随机生成的字符串
AES_KEY = sha256(用户密码)
VI = 随机字符串(32byte)   # 了解下AES算法，就知道VI是干嘛的了
加密中间值 = UTF-8( base64( VI + AES_CBC( AES_KEY, VI) ) )
加密前缀 = hex( sha256(MasterPassword) )[:4]
加密后的MasterPassword = 加密前缀 + "$" + 加密中间值
```

待续...

#### 操作命令

##### set
设置本地配置，存储于`Configuration`中，目前只能修改：
* default\_account
* default\_vote\_weight
* nodes

##### config
列出本地配置，只能显示`set`命令能修改的那几项。

##### info
用于显示`steem`区块链上的信息，例如博文，货币总量，当前汇率等。

```
> steempy info  # 显示head区块链的信息和一些其他基本信息
+---------------------------------+------------------------------------------+
| Key                             | Value                                    |
+---------------------------------+------------------------------------------+
| average_block_size              | 17555                                    |
| confidential_sbd_supply         | 0.000 SBD                                |
| confidential_supply             | 0.000 STEEM                              |
| current_aslot                   | 20293015                                 |
| current_reserve_ratio           | 130516821                                |
| current_sbd_supply              | 9741713.725 SBD                          |
| current_supply                  | 264689185.354 STEEM                      |
| current_witness                 | gtg                                      |
| head_block_id                   | 0134ae176bac5d9f2b54a3d57abe4404fd7e9851 |
| head_block_number               | 20229655                                 |
| id                              | 0                                        |
| internal price                  | 3.325                                    |
| last_irreversible_block_num     | 20229640                                 |
| max_virtual_bandwidth           | 172439575682088960000                    |
| maximum_block_size              | 65536                                    |
| num_pow_witnesses               | 172                                      |
| participation_count             | 128                                      |
| pending_rewarded_vesting_shares | 323928167.875575 VESTS                   |
| pending_rewarded_vesting_steem  | 157592.040 STEEM                         |
| recent_slots_filled             | 340282366920938463463374607431768211455  |
| sbd_interest_rate               | 0                                        |
| sbd_print_rate                  | 10000                                    |
| steem per mvest                 | 489.42201392298205                       |
| time                            | 2018-02-27T06:50:45                      |
| total_pow                       | 514415                                   |
| total_reward_fund_steem         | 0.000 STEEM                              |
| total_reward_shares2            | 0                                        |
| total_vesting_fund_steem        | 186528960.475 STEEM                      |
| total_vesting_shares            | 381120904186.285994 VESTS                |
| virtual_supply                  | 267619024.068 STEEM                      |
| vote_power_reserve_rate         | 10                                       |
+---------------------------------+------------------------------------------+

# 查看一篇博文信息，这篇博文链接为: https://steemit.com/esteem/@htcptc/it-s-a-great-date-8f5a62660a36f
# 博文的识别符为@{用户名}/{博文permlink} 即下面的identifier字段
> steempy info @htcptc/it-s-a-great-date-8f5a62660a36f
+----------------------------+-------------------------------------------------+
| Key                        | Value                                           |
+----------------------------+-------------------------------------------------+
| abs_rshares                | 0                                               |
| active                     | 2018-02-27 06:58:18                             |
| active_votes               | []                                              |
| allow_curation_rewards     | True                                            |
| allow_replies              | True                                            |
| allow_votes                | True                                            |
| author                     | htcptc                                          |
| author_reputation          | 136926895748                                    |
| author_rewards             | 0                                               |
| beneficiaries              | [{'account': 'esteemapp', 'weight': 1000}]      |
| body                       | ![image](https://img.esteem.ws/hl4un9qewp.jpg)  |
| body_length                | 0                                               |
| cashout_time               | 2018-03-06 06:58:18                             |
| category                   | esteem                                          |
| children                   | 0                                               |
| children_abs_rshares       | 0                                               |
| community                  | esteem                                          |
| created                    | 2018-02-27 06:58:18                             |
| curator_payout_value       | 0.000 SBD                                       |
| depth                      | 0                                               |
| id                         | 35508214                                        |
| identifier                 | @htcptc/it-s-a-great-date-8f5a62660a36f         |
| json_metadata              | {                                               |
|                            |     "links": [],                                |
|                            |     "image": [                                  |
|                            |         "https://img.esteem.ws/hl4un9qewp.jpg"  |
|                            |     ],                                          |
|                            |     "tags": [                                   |
|                            |         "esteem",                               |
|                            |         "travel",                               |
|                            |         "photography",                          |
|                            |         "blog"                                  |
|                            |     ],                                          |
|                            |     "app": "esteem/1.5.1",                      |
|                            |     "format": "markdown+html",                  |
|                            |     "community": "esteem"                       |
|                            | }                                               |
| last_payout                | 1970-01-01 00:00:00                             |
| last_update                | 2018-02-27 06:58:18                             |
| max_accepted_payout        | 1000000.000 SBD                                 |
| max_cashout_time           | 1969-12-31 23:59:59                             |
| net_rshares                | 0                                               |
| net_votes                  | 0                                               |
| parent_author              |                                                 |
| parent_permlink            | esteem                                          |
| pending_payout_value       | 0.000 SBD                                       |
| percent_steem_dollars      | 10000                                           |
| permlink                   | it-s-a-great-date-8f5a62660a36f                 |
| promoted                   | 0.000 SBD                                       |
| reblogged_by               | []                                              |
| replies                    | []                                              |
| reward_weight              | 10000                                           |
| root_comment               | 35508214                                        |
| root_title                 | it's a great date                               |
| tags                       | [                                               |
|                            |     "esteem",                                   |
|                            |     "esteem",                                   |
|                            |     "travel",                                   |
|                            |     "photography",                              |
|                            |     "blog"                                      |
|                            | ]                                               |
| title                      | it's a great date                               |
| total_payout_value         | 0.000 SBD                                       |
| total_pending_payout_value | 0.000 STEEM                                     |
| total_vote_weight          | 0                                               |
| url                        | /esteem/@htcptc/it-s-a-great-date-8f5a62660a36f |
| vote_rshares               | 0                                               |
+----------------------------+-------------------------------------------------+

# 查看账户信息
> steempy info icycrystal4
+-----------------------------------+----------------------------------------------------------------------+
| Key                               | Value                                                                |
+-----------------------------------+----------------------------------------------------------------------+
| active                            | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| active_challenged                 | False                                                                |
| average_bandwidth                 | 117000000                                                            |
| average_market_bandwidth          | 0                                                                    |
| balance                           | 0.000 STEEM                                                          |
| can_vote                          | True                                                                 |
| comment_count                     | 0                                                                    |
| created                           | 2018-02-24T11:20:54                                                  |
| curation_rewards                  | 0                                                                    |
| delegated_vesting_shares          | 0.000000 VESTS                                                       |
| guest_bloggers                    | []                                                                   |
| id                                | 774870                                                               |
| json_metadata                     | {}                                                                   |
| last_account_recovery             | 1970-01-01T00:00:00                                                  |
| last_account_update               | 1970-01-01T00:00:00                                                  |
| last_active_proved                | 1970-01-01T00:00:00                                                  |
| last_bandwidth_update             | 2018-02-24T11:23:15                                                  |
| last_market_bandwidth_update      | 1970-01-01T00:00:00                                                  |
| last_owner_proved                 | 1970-01-01T00:00:00                                                  |
| last_owner_update                 | 1970-01-01T00:00:00                                                  |
| last_post                         | 1970-01-01T00:00:00                                                  |
| last_root_post                    | 1970-01-01T00:00:00                                                  |
| last_vote_time                    | 2018-02-24T11:23:15                                                  |
| lifetime_bandwidth                | 117000000                                                            |
| lifetime_market_bandwidth         | 0                                                                    |
| lifetime_vote_count               | 0                                                                    |
| market_history                    | []                                                                   |
| memo_key                          | STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2                |
| mined                             | False                                                                |
| name                              | icycrystal4                                                          |
| next_vesting_withdrawal           | 1969-12-31T23:59:59                                                  |
| other_history                     | []                                                                   |
| owner                             | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM713qFYhL2CHQ3HnD1CVxL2KznsTJRVAaCMWFCAgxCBiw4abzn7", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| owner_challenged                  | False                                                                |
| post_count                        | 0                                                                    |
| post_history                      | []                                                                   |
| posting                           | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| posting_rewards                   | 0                                                                    |
| proxied_vsf_votes                 | [0, 0, 0, 0]                                                         |
| proxy                             |                                                                      |
| received_vesting_shares           | 29700.000000 VESTS                                                   |
| recovery_account                  | steem                                                                |
| reputation                        | 0                                                                    |
| reset_account                     | null                                                                 |
| reward_sbd_balance                | 0.000 SBD                                                            |
| reward_steem_balance              | 0.000 STEEM                                                          |
| reward_vesting_balance            | 0.000000 VESTS                                                       |
| reward_vesting_steem              | 0.000 STEEM                                                          |
| savings_balance                   | 0.000 STEEM                                                          |
| savings_sbd_balance               | 0.000 SBD                                                            |
| savings_sbd_last_interest_payment | 1970-01-01T00:00:00                                                  |
| savings_sbd_seconds               | 0                                                                    |
| savings_sbd_seconds_last_update   | 1970-01-01T00:00:00                                                  |
| savings_withdraw_requests         | 0                                                                    |
| sbd_balance                       | 0.000 SBD                                                            |
| sbd_last_interest_payment         | 1970-01-01T00:00:00                                                  |
| sbd_seconds                       | 0                                                                    |
| sbd_seconds_last_update           | 1970-01-01T00:00:00                                                  |
| tags_usage                        | []                                                                   |
| to_withdraw                       | 0                                                                    |
| transfer_history                  | []                                                                   |
| vesting_balance                   | 0.000 STEEM                                                          |
| vesting_shares                    | 1021.764734 VESTS                                                    |
| vesting_withdraw_rate             | 0.000000 VESTS                                                       |
| vote_history                      | []                                                                   |
| voting_power                      | 9800                                                                 |
| withdraw_routes                   | 0                                                                    |
| withdrawn                         | 0                                                                    |
| witness_votes                     | []                                                                   |
| witnesses_voted_for               | 0                                                                    |
+-----------------------------------+----------------------------------------------------------------------+

# 查看区块信息，从1开始
# 查看创世块
> steempy info 1
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+
| Key                     | Value                                                                                                                              |
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+
| block_id                | 0000000109833ce528d5bbfb3f6225b39ee10086                                                                                           |
| extensions              | []                                                                                                                                 |
| previous                | 0000000000000000000000000000000000000000                                                                                           |
| signing_key             | STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX                                                                              |
| timestamp               | 2016-03-24T16:05:00                                                                                                                |
| transaction_ids         | []                                                                                                                                 |
| transaction_merkle_root | 0000000000000000000000000000000000000000                                                                                           |
| transactions            | []                                                                                                                                 |
| witness                 | initminer                                                                                                                          |
| witness_signature       | 204f8ad56a8f5cf722a02b035a61b500aa59b9519b2c33c77a80c0a714680a5a5a7a340d909d19996613c5e4ae92146b9add8a7a663eef37d837ef881477313043 |
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+

# 根据公钥查看账户信息
> steempy info STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH
+-------------+
| Account     |
+-------------+
| icycrystal4 |
+-------------+
```