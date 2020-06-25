## cfxstore-cli

Conflux密钥管理。它以Parity Ethereum对应部分为基础建立。

### Usage

```
Conflux key management tool.
  Copyright 2020 Conflux Foundation

Usage:
    cfxstore insert <secret> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore change-pwd <address> <old-pwd> <new-pwd> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore list [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore import [--src DIR] [--dir DIR]
    cfxstore import-wallet <path> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore find-wallet-pass <path> <password>
    cfxstore remove <address> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore sign <address> <password> <message> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore public <address> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore list-vaults [--dir DIR]
    cfxstore create-vault <vault> <password> [--dir DIR]
    cfxstore change-vault-pwd <vault> <old-pwd> <new-pwd> [--dir DIR]
    cfxstore move-to-vault <address> <vault> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]
    cfxstore move-from-vault <address> <vault> <password> [--dir DIR]
    cfxstore [-h | --help]

Options:
    -h, --help               Display this message and exit.
    --dir DIR                Specify the secret store directory. It may be either
                             parity, parity-(chain), geth, geth-test
                             or a path [default: parity].
    --vault VAULT            Specify vault to use in this operation.
    --vault-pwd VAULTPWD     Specify vault password to use in this operation. Please note
                             that this option is required when vault option is set.
                             Otherwise it is ignored.
    --src DIR                Specify import source. It may be either
                             parity, parity-(chain), geth, geth-test
                             or a path [default: geth].

Commands:
    insert             Save account with password.
    change-pwd         Change password.
    list               List accounts.
    import             Import accounts from src.
    import-wallet      Import presale wallet.
    find-wallet-pass   Tries to open a wallet with list of passwords given.
    remove             Remove account.
    sign               Sign message.
    public             Displays public key for an address.
    list-vaults        List vaults.
    create-vault       Create new vault.
    change-vault-pwd   Change vault password.
    move-to-vault      Move account to vault from another vault/root directory.
    move-from-vault    Move account to root directory from given vault.
```

### 样例

#### `insert <secret> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*使用口令<password>加密私钥，并将其保存在私钥存储器中*

- `<secret>` - conflux私钥，32字节长
- `<password>` - 账户口令，文件路径
- `[--dir DIR]` - 私钥的存储路径可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore insert 7d29fab185a33e2cd955812397354c472d2b84615b645aa135ff539f6b0d70d5 password.txt
```

```
a8fa5dd30a87bb9e3288d604eb74949c515ab66e
```

--

```
cfxstore insert `cfxkey generate random -s` "this is sparta"
```

```
24edfff680d536a5f6fe862d36df6f8f6f40f115
```

--

#### `change-pwd <address> <old-pwd> <new-pwd> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*修改账户口令*

- `<address>` - conflux地址，20字节长
- `<old-pwd>` - 原有账户口令，文件路径
- `<new-pwd>` - 新账户口令，文件路径
- `[--dir DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore change-pwd a8fa5dd30a87bb9e3288d604eb74949c515ab66e old_pwd.txt new_pwd.txt
```

```
true
```

--

#### `list [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*列出私钥存储账户*

- `[--dir DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore list
```

```
 0: 24edfff680d536a5f6fe862d36df6f8f6f40f115
 1: 6edddfc6349aff20bc6467ccf276c5b52487f7a8
 2: e6a3d25a7cb7cd21cb720df5b5e8afd154af1bbb
```

--

#### `import [--src DIR] [--dir DIR]`
*从src导入私钥*

- `[--src DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为geth
- `[--dir DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity

```
cfxstore import
```

```
 0: e6a3d25a7cb7cd21cb720df5b5e8afd154af1bbb
 1: 6edddfc6349aff20bc6467ccf276c5b52487f7a8
```

--

#### `import-wallet <path> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*从presale钱包导入账户*

- `<path>` - presale钱包路径
- `<password>` - 账户口令，文件路径
- `[--dir DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore import-wallet ethwallet.json password.txt
```

```
e6a3d25a7cb7cd21cb720df5b5e8afd154af1bbb
```


--

#### `find-wallet-pass <path> <password>`
尝试从文件中打开给定口令列表的presale钱包。口令列表可以通过以下方法生成[Phildo/brutedist](https://github.com/Phildo/brutedist).

- `<path>` - presale钱包路径
- `<password>` - 可能的口令，文件路径

```
cfxstore find-wallet-pass ethwallet.json passwords.txt
```

```
Found password: test
```


--

#### `remove <address> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*从私钥存储区内移除账户*

- `<address>` - conflux地址，20字节长
- `<password>` - 账户口令，文件路径
- `[--dir DIR]` - 私钥存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore remove a8fa5dd30a87bb9e3288d604eb74949c515ab66e password.txt
```

```
true
```

--

#### `sign <address> <password> <message> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*使用账户的私钥对消息进行签名。*

- `<address>` - conflux地址，20字节长
- `<password>` - account口令，文件路径
- `<message>` - 需要签名的消息，32字节长
- `[--dir DIR]` - 私钥的存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 在此操作中需要使用的Vault口令，文件路径

```
cfxstore sign 24edfff680d536a5f6fe862d36df6f8f6f40f115 password.txt 7d29fab185a33e2cd955812397354c472d2b84615b645aa135ff539f6b0d70d5
```

```
c6649f9555232d90ff716d7e552a744c5af771574425a74860e12f763479eb1b708c1f3a7dc0a0a7f7a81e0a0ca88c6deacf469222bb3d9c5bf0847f98bae54901
```

--

#### `public <address> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*显示地址的公钥。*

- `<address>` - conflux地址，20字节长
- `<password>` - account口令，文件路径
- `[--dir DIR]` - 私钥的存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity
- `[--vault VAULT]` - 该操作需要使用的Vault
- `[--vault-pwd VAULTPWD]` - 此操作中需要使用的Vault口令，文件路径

```
cfxstore public 00e63fdb87ceb815ec96ae185b8f7381a0b4a5ea account_password.txt --vault vault_name --vault-pwd vault_password.txt
```

```
0x84161d8c05a996a534efbec50f24485cfcc07458efaef749a1b22156d7836c903eeb39bf2df74676e702eacc4cfdde069e5fd86692b5ef6ef81ba906e9e77d82
```

--

#### `list-vaults [--dir DIR]`
*列出所有Vault*

- `[--dir DIR]` - 私钥的存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity

```
cfxstore list-vaults
```

```
vault1
vault2
vault3
```

--

#### `create-vault <vault> <password> [--dir DIR]`
*创建新的Vault*

- `<vault>` - 新vault的名字。只能包含字母、数字、空格、破折号和下划线
- `<password>` - vault口令，文件路径
- `[--dir DIR]` - 私钥的存放路径，可以是parity、 parity-test、geth、geth-test或是一个路径。默认为parity

```
cfxstore create-vault vault3 vault3_password.txt
```

```
OK
```

--

#### `change-vault-pwd <vault> <old-pwd> <new-pwd> [--dir DIR]`
*修改Vault对应的口令*

- `<vault>` - name of existing vault
- `<old-pwd>` - old vault password, file path
- `<new-pwd>` - new vault password, file path
- `[--dir DIR]` - secret store directory, It may be either parity, parity-test, geth, geth-test or a path. default: parity

```
cfxstore change-vault-pwd vault3 vault3_password.txt new_vault3_password.txt
```

```
OK
```

--

#### `move-to-vault <address> <vault> <password> [--dir DIR] [--vault VAULT] [--vault-pwd VAULTPWD]`
*Move account to vault from another vault/root directory.*

- `<address>` - conflux address, 20 bytes long
- `<vault>` - name of existing vault to move account to
- `<password>` - password of existing `<vault>` to move account to, file path
- `[--dir DIR]` - secret store directory, It may be either parity, parity-test, geth, geth-test or a path. default: parity
- `[--vault VAULT]` - current vault of the `<address>` argument, if set
- `[--vault-pwd VAULTPWD]` - password for the current vault of the `<address>` argument, if any. file path


```
cfxstore move-to-vault 00e63fdb87ceb815ec96ae185b8f7381a0b4a5ea vault3 vault3_password.txt
cfxstore move-to-vault 00e63fdb87ceb815ec96ae185b8f7381a0b4a5ea vault1 vault1_password.txt --vault vault3 --vault-pwd vault3_password.txt
```

```
OK
OK
```

--

#### `move-from-vault <address> <vault> <password> [--dir DIR]`
*Move account to root directory from given vault.*

- `<address>` - conflux address, 20 bytes long
- `<vault>` - name of existing vault to move account to
- `<password>` - password of existing `<vault>` to move account to, file path
- `[--dir DIR]` - secret store directory, It may be either parity, parity-test, geth, geth-test or a path. default: parity


```
cfxstore move-from-vault 00e63fdb87ceb815ec96ae185b8f7381a0b4a5ea vault1 vault1_password.txt
```

```
OK
```
