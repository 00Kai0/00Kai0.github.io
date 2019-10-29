---
layout: post
title:  "Python 解析支付宝公钥证书"
image: ''
date:   2019-07-22 00:30:00
tags:
- alipaySDK
- Python
- MD5
description: ''
categories:
- Python
---

由于工作需要，我们开发的 App 需要接入支付宝的支付功能。于是我们着手了解 Alipay 相关的 api 文档。   
经过斟酌后，我们选择使用了一个第三方 SDK (https://github.com/fzlee/alipay)。这里不得不吐槽一下官方的python SDK，
从包路径到使用，都带有强烈的 java 风格，没有了 python 的简约气息。   
尽管使用了 SDK，但是我们发现，最新的支付宝都使用了公钥证书来签名。而官方 SDK 也只有 java 才支持公钥证书方式，我们使用的第三方 SDK 也没有。
于是乎我们不得不自己来实现公钥证书的解析，但是网络上关于自行实现签名的内容比较少，仅有提供一些说明。   
官方提供了自行实现签名的过程 https://docs.open.alipay.com/291/106118。   
其中比较关键的是从证书提取`app_cert_sn` 和 `alipay_root_cert_sn`两个关键参数，这里给出 Java 中的实现:
```java
/**
 * 从公钥证书中提取公钥序列号
 *
 * @param certPath 公钥证书存放路径，例如:/home/admin/cert.crt
 * @return 公钥证书序列号
 * @throws AlipayApiException
 */
public static String getCertSN(String certPath) throws AlipayApiException {
    InputStream inputStream = null;
    try {
        inputStream = new FileInputStream(certPath);
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        X509Certificate cert = (X509Certificate) cf.generateCertificate(inputStream);
        MessageDigest md = MessageDigest.getInstance("MD5");
        md.update((cert.getIssuerX500Principal().getName() + cert.getSerialNumber()).getBytes());
        String certSN = new BigInteger(1, md.digest()).toString(16);
        //BigInteger会把0省略掉，需补全至32位
        certSN = fillMD5(certSN);
        return certSN;

    } catch (NoSuchAlgorithmException e) {
        throw new AlipayApiException(e);
    } catch (IOException e) {
        throw new AlipayApiException(e);
    } catch (CertificateException e) {
        throw new AlipayApiException(e);
    } finally {
        try {
            if (inputStream != null) {
                inputStream.close();
            }
        } catch (IOException e) {
            throw new AlipayApiException(e);
        }
    }
}

/**
 * 获取根证书序列号
 *
 * @param rootCertContent
 * @return
 */
public static String getRootCertSN(String rootCertContent) {
    String rootCertSN = null;
    try {
        X509Certificate[] x509Certificates = readPemCertChain(rootCertContent);
        MessageDigest md = MessageDigest.getInstance("MD5");
        for (X509Certificate c : x509Certificates) {
            if (c.getSigAlgOID().startsWith("1.2.840.113549.1.1")) {
                md.update((c.getIssuerX500Principal().getName() + c.getSerialNumber()).getBytes());
                String certSN = new BigInteger(1, md.digest()).toString(16);
                //BigInteger会把0省略掉，需补全至32位
                certSN = fillMD5(certSN);
                if (StringUtils.isEmpty(rootCertSN)) {
                    rootCertSN = certSN;
                } else {
                    rootCertSN = rootCertSN + "_" + certSN;
                }
            }

        }
    } catch (Exception e) {
        AlipayLogger.logBizError(("提取根证书失败"));
    }
    return rootCertSN;

}

private static String fillMD5(String md5) {
    return md5.length() == 32 ? md5 : fillMD5("0" + md5);
}

```

这里和官网所说的流程大概相同:
* 解析X.509证书文件，获取证书签发机构名称（name）以及证书内置序列号（serialNumber）。   
* 将name与serialNumber拼接成字符串，再对该字符串做MD5计算。

`第一步`中解析X.509证书比较容易，在 python 实现中我们使用了 openssl 来解析证书：
```python
cert = OpenSSL.crypto.load_certificate(OpenSSL.crypto.FILETYPE_PEM, cert)
```
但是在获取 name 和 serialNumber 碰到了障碍。
在 Java 中我们看到 `md.update((c.getIssuerX500Principal().getName() + c.getSerialNumber()).getBytes());` 这一行可以轻松提取 name 和 serialName。
可惜的是，openssl 里只有`get_serial_number`这样的 API 来提取序列号，没有像 java 里 `getIssuerX500Principal `来获取他想要的机构名称。
经过长时间的资料查询和研究，从 https://sbing.vip/archives/2019-new-alipay-php-docking.html 这里找到了线索：
```
需要拼接成：CN=Ant Financial Certification Authority Class 2 R1,OU=Certification Authority,O=Ant Financial,C=CN
```
于是找到了解决方法：`name = 'CN={},OU={},O={},C={}'.format(certIssue.CN, certIssue.OU, certIssue.O, certIssue.C)`。   
`第二步`中的拼接和 MD5 校验就比较简单，使用 python 自带的 hashlib 就可以完成，并且比 Java 更简洁。   
最后一个问题来自于根证书，源码显示根证书包含多个证书信息，读取文件的时候需要使用 `split('\n\n')` 来获取证书字符串列表，再遍历获取证书 SN 信息。
还有源码里做了筛选`if (c.getSigAlgOID().startsWith("1.2.840.113549.1.1"))`，`Openssl`里也没有这样的 API 可以调度。
我没有选择像它那样解析出算法的 OID。我猜想这个就是为了找到指定算法类型，于是我使用了别的方法代替：
```python
try:
    sigAlg = cert.get_signature_algorithm()
except ValueError:
    continue
if b'rsaEncryption' in sigAlg or b'RSAEncryption' in sigAlg:
```


以上是我对支付宝公钥证书验证的大致理解，最后的算法类型也是我的猜测，有问题可以告诉我哦。这么看起来不是特别困难的问题，但是在解决问题的过程中的确花了很多时间，网络上能提供的资料也只有支付宝官网的文档和上面
的一个php实现的博客。解决完让我豁然开朗，也希望还在炮坑的同学能从中受益。   
目前我是在[alipay](https://github.com/fzlee/alipay) ( 我觉得做的还可以 )基础上加入了证书签名。如果以后有时间我希望可以计划制作一个更好用的 SDK。   
完整 Demo 的地址：https://github.com/00Kai0/pyAliPay/blob/master/demo.py
