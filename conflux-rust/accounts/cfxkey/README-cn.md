## cfxkey-cli

Conflux密钥生成器。 建立在对应的Parity Ethereum之上。值得注意的是，Conflux的地址方案与与Ethereum不同。您不能直接将Ethereum密钥文件导入Conflux。

### 使用方法

```
Conflux Keys Generator.
  Copyright 2020 Conflux Foundation

Usage:
    cfxkey info <secret-or-phrase> [options]
    cfxkey generate random [options]
    cfxkey generate prefix <prefix> [options]
    cfxkey sign <secret> <message>
    cfxkey verify public <public> <signature> <message>
    cfxkey verify address <address> <signature> <message>
    cfxkey recover <address> <known-phrase>
    cfxkey [-h | --help]

Options:
    -h, --help         Display this message and exit.
    -s, --secret       Display only the secret key.
    -p, --public       Display only the public key.
    -a, --address      Display only the address.
    -b, --brain        Use parity brain wallet algorithm. Not recommended.

Commands:
    info               Display public key and address of the secret.
    generate random    Generates new random Ethereum key.
    generate prefix    Random generation, but address must start with a prefix ("vanity address").
    sign               Sign message using a secret key.
    verify             Verify signer of the signature by public key or address.
    recover            Try to find brain phrase matching given address from partial phrase.
```

### 样例

#### `info <secret>`
*显示私钥信息*

- `<secret>` - conflux私钥信息, 32字节长

```
cfxkey info 17d08f5fe8c77af811caa0c9a187e668ce3b74a99acc3f6d976f075fa8e0be55
```

```
secret:  17d08f5fe8c77af811caa0c9a187e668ce3b74a99acc3f6d976f075fa8e0be55
public:  689268c0ff57a20cd299fa60d3fb374862aff565b20b5f1767906a99e6e09f3ff04ca2b2a5cd22f62941db103c0356df1a8ed20ce322cab2483db67685afd124
address: 26d1ec50b4e62c1d1a40d16e7cacc6a6580757d5
```

--


#### `info --brain <phrase>`
*显示从brain钱包恢复短语生成私钥的信息。*

- `<phrase>` - 助记词恢复短语，共12个单词

```
cfxkey info --brain "this is sparta"
```

```
恢复短语不是由Parity生成的：单词'this'也不是来源于字典。

secret:  aa22b54c0cb43ee30a014afe5ef3664b1cde299feabca46cd3167a85a57c39f2
public:  c4c5398da6843632c123f543d714d2d2277716c11ff612b2a2f23c6bda4d6f0327c31cd58c55a9572c3cc141dade0c32747a13b7ef34c241b26c84adbb28fcf4
address: 006e27b6a72e1f34c626762f3c4761547aff1421
```

--

#### `generate random`
*随机生成密钥对。*

```
cfxkey generate random
```

```
secret:  7d29fab185a33e2cd955812397354c472d2b84615b645aa135ff539f6b0d70d5
public:  35f222d88b80151857a2877826d940104887376a94c1cbd2c8c7c192eb701df88a18a4ecb8b05b1466c5b3706042027b5e079fe3a3683e66d822b0e047aa3418
address: a8fa5dd30a87bb9e3288d604eb74949c515ab66e
```

--

#### `generate random --brain`
*随机生成带有恢复短语的新密钥对。*

```
cfxkey generate random --brain
```

```
recovery phrase: thwarting scandal creamer nuzzle asparagus blast crouch trusting anytime elixir frenzied octagon
secret:  001ce488d50d2f7579dc190c4655f32918d505cee3de63bddc7101bc91c0c2f0
public:  4e19a5fdae82596e1485c69b687c9cc52b5078e5b0668ef3ce8543cd90e712cb00df822489bc1f1dcb3623538a54476c7b3def44e1a51dc174e86448b63f42d0
address: 00cf3711cbd3a1512570639280758118ba0b2bcb
```


--

#### `generate prefix <prefix>`
*用<prefix>对应前缀开始的地址随机生成新的密钥对。*

- `<prefix>` - 所需要的地址前缀，0-32字节长。

```
cfxkey generate prefix ff
```

```
secret:  2075b1d9c124ea673de7273758ed6de14802a9da8a73ceb74533d7c312ff6acd
public:  48dbce4508566a05509980a5dd1335599fcdac6f9858ba67018cecb9f09b8c4066dc4c18ae2722112fd4d9ac36d626793fffffb26071dfeb0c2300df994bd173
address: fff7e25dff2aa60f61f9d98130c8646a01f31649
```

--

#### `generate prefix --brain <prefix>`
*随机生成带有恢复短语的新密钥对，地址以<prefix>指定的前缀开始。*

- `<prefix>` - 所需要的地址前缀，0-32字节长。

```
cfxkey generate prefix --brain 00cf
```

```
recovery phrase: thwarting scandal creamer nuzzle asparagus blast crouch trusting anytime elixir frenzied octagon
secret:  001ce488d50d2f7579dc190c4655f32918d505cee3de63bddc7101bc91c0c2f0
public:  4e19a5fdae82596e1485c69b687c9cc52b5078e5b0668ef3ce8543cd90e712cb00df822489bc1f1dcb3623538a54476c7b3def44e1a51dc174e86448b63f42d0
address: 00cf3711cbd3a1512570639280758118ba0b2bcb
```

--

#### `sign <secret> <message>`
*使用私钥对消息进行签名*

- `<secret>` - conflux私钥，32字节长
- `<message>` - 待签名消息,32字节长

```
cfxkey sign 17d08f5fe8c77af811caa0c9a187e668ce3b74a99acc3f6d976f075fa8e0be55 bd50b7370c3f96733b31744c6c45079e7ae6c8d299613246d28ebcef507ec987
```

```
c1878cf60417151c766a712653d26ef350c8c75393458b7a9be715f053215af63dfd3b02c2ae65a8677917a8efa3172acb71cb90196e42106953ea0363c5aaf200
```

--

#### `verify public <public> <signature> <message>`
*验证签名。*

- `<public>` - conflux公钥，64字节长
- `<signature>` - 消息签名，65字节长
- `<message>` - 消息，32字节长

```
cfxkey verify public 689268c0ff57a20cd299fa60d3fb374862aff565b20b5f1767906a99e6e09f3ff04ca2b2a5cd22f62941db103c0356df1a8ed20ce322cab2483db67685afd124 c1878cf60417151c766a712653d26ef350c8c75393458b7a9be715f053215af63dfd3b02c2ae65a8677917a8efa3172acb71cb90196e42106953ea0363c5aaf200 bd50b7370c3f96733b31744c6c45079e7ae6c8d299613246d28ebcef507ec987
```

```
true
```

--

#### `verify address <address> <signature> <message>`
*验证签名。*

- `<address>` - conflux address, 20 bytes long
- `<signature>` - message signature, 65 bytes long
- `<message>` - message, 32 bytes long

```
cfxkey verify address 689268c0ff57a20cd299fa60d3fb374862aff565b20b5f1767906a99e6e09f3ff04ca2b2a5cd22f62941db103c0356df1a8ed20ce322cab2483db67685afd124 c1878cf60417151c766a712653d26ef350c8c75393458b7a9be715f053215af63dfd3b02c2ae65a8677917a8efa3172acb71cb90196e42106953ea0363c5aaf200 bd50b7370c3f96733b31744c6c45079e7ae6c8d299613246d28ebcef507ec987
```

```
true
```

--

#### `recover <address> <known-phrase>`
*尝试恢复给定预期地址和部分恢复短语（太短或有无效词）的账户。*

- `<address>` - conflux地址，20字节长
- `<known-phrase>` - 已知的恢复短语，形式可以是 `thwarting * creamer`

```
RUST_LOG="info" cfxkey recover "00cf3711cbd3a1512570639280758118ba0b2bcb" "thwarting scandal creamer nuzzle asparagus blast crouch trusting anytime elixir frenzied octag"
```

```
INFO:cfxkey::brain_recover: Invalid word 'octag', looking for potential substitutions.
INFO:cfxkey::brain_recover: Closest words: ["ocean", "octagon", "octane", "outage", "tag", "acting", "acts", "aorta", "cage", "chug"]
INFO:cfxkey::brain_recover: Starting to test 7776 possible combinations.

thwarting scandal creamer nuzzle asparagus blast crouch trusting anytime elixir frenzied octagon
secret:  001ce488d50d2f7579dc190c4655f32918d505cee3de63bddc7101bc91c0c2f0
public:  4e19a5fdae82596e1485c69b687c9cc52b5078e5b0668ef3ce8543cd90e712cb00df822489bc1f1dcb3623538a54476c7b3def44e1a51dc174e86448b63f42d0
address: 00cf3711cbd3a1512570639280758118ba0b2bcb
```
