#  条件语句

条件语句体应该总是被大括号包围。尽管有时候你可以不使用大括号（比如，条件语句体只有一行内容），但是这样做会带来问题隐患。比如，增加一行代码时，你可能会误以为它是 if 语句体里面的。此外，更危险的是，如果把 if 后面的那行代码注释掉，之后的一行代码会成为 if 语句里的代码。

**推荐:**

```obj-c
if (!error) {
    return success;
}
```

**不推荐:**

```obj-c
if (!error)
    return success;
```

和

```obj-c
if (!error) return success;
```


在 2014年2月 苹果的 SSL/TLS 实现里面发现了知名的 [goto fail](https://gotofail.com/) 错误。

代码在这里：


```obj-c
static OSStatus
SSLVerifySignedServerKeyExchange(SSLContext *ctx, bool isRsa, SSLBuffer signedParams,
                                 uint8_t *signature, UInt16 signatureLen)
{
  OSStatus        err;
  ...

  if ((err = SSLHashSHA1.update(&hashCtx, &serverRandom)) != 0)
    goto fail;
  if ((err = SSLHashSHA1.update(&hashCtx, &signedParams)) != 0)
    goto fail;
    goto fail;
  if ((err = SSLHashSHA1.final(&hashCtx, &hashOut)) != 0)
    goto fail;
  ...

fail:
  SSLFreeBuffer(&signedHashes);
  SSLFreeBuffer(&hashCtx);
  return err;
}
```


显而易见，这里有没有括号包围的2行连续的 `goto fail;` 。我们当然不希望写出上面的代码导致错误。

此外，在其他条件语句里面也应该按照这种风格统一，这样更便于检查。
