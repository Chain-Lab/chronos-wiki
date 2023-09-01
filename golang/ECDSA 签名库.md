# ECDSA 椭圆曲线加密库

ecdsa 库用于地址的生成、交易的签名和交易的校验

地址有多种生成方式，通过对公钥进行不同的哈希组合输出即可

私钥的生成需要传入 golang 中的随机数发生器

```go
private_key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
```

这里的实际上是一个公私钥对，它的结构体如下：

```go
type PrivateKey struct {
	PublicKey
	D *big.Int
}
```

其中 D 是私钥，是一个大整数，PublicKey 是公钥，是椭圆曲线上的一个点，公钥的结构体：

```go
type PublicKey struct {
	elliptic.Curve
	X, Y *big.Int
}
```

包含了代表椭圆曲线上一点的 (X, Y)，以及对应的曲线类型，通过调用  `private_key.PublicKey` 可以得到公钥

### 公私钥的编码

在实际应用场景下需要对公私钥进行编码，私钥的编码/解码过程：

```go
hexPrivateKey := hex.EncodeToString(private.D.Bytes())           // 私钥的编码, 得到私钥 16 进制字符串
privateBytes, err := hex.DecodeString(hexPrivateKey)             // 私钥的解码，得到私钥的字节信息
```

在 golang 中没有给出通过私钥的 D 值来生成 PrivateKey 结构体的函数，**需要手动实现**，结合公钥的恢复过程得到完整的公私钥对，理论上来说只要填充了 D 值到结构体中都可以签名

公钥的编码/解码过程需要使用椭圆曲线库中的 `MarshalCompressed` 函数来将公钥的两个点压缩/编码为一串 bytes，解码过程需要使用 `UnmarshalCompressed`  函数将 bytes 再转换为 (X, Y)

```go
// 将公钥转换为 byte 数组再转换为 16 进制字符串
hexPublicKey := hex.EncodeToString(elliptic.MarshalCompressed(elliptic.P256(), pub.X, pub.Y))

// 先将公钥转换为 byte 数组
bytesPublicKey, err := hex.DecodeString(hexPublicKey)
// 然后转换 bytes 数组到两个点
x, y := elliptic.UnmarshalCompressed(elliptic.P256(), bytesPublicKey)
// 实例化
publicKey := ecdsa.PublicKey{
    elliptic.P256(),
    x,
    y,
}
```

### 签名的编码

ecdsa 签名输出的是整数对 (r, s)，属于 `big.Int` 类型，使用常规的方法对它进行编码得到16进制字符串即可

通过 ecdsa 库进行签名的校验，4000笔交易耗时在 300ms ~ 400ms，平均单笔交易耗时 0.075ms ~ 0.1ms