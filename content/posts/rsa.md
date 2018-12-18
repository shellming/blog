---
title: RSA 公钥私钥生成及使用
date: 2018-12-11
categories: [development]
tags: [rsa,非对称加密,安全]
language: zh
---
有些接口交互需要用 RSA 算法对参数进行签名以保证安全性，记录一下 RSA 密钥对的生成以及使用

#### 秘钥生成

为方便 Java 使用，最终生成的私钥需要转化成 PKCS8 格式
``` bash
# 生成 SSLeay 格式的 rsa 私钥
openssl genrsa -out rsaprivatekey.pem 1024

# 生成对应的公钥
openssl rsa -in rsaprivatekey.pem -pubout -out rsapublickey.pem

# 将 RSA 私钥转换成 PKCS8 格式
openssl pkcs8 -topk8 -inform PEM -in rsaprivatekey.pem -outform PEM -nocrypt -out rsaprivatepkcs8.pem
```

由于生成的私钥中已经包含了公钥信息，所以服务器端只要保存私钥就可以了，需要的时候可以从私钥中导出公钥

```
# 将私钥从 PKCS8 格式转回 SSLeay 格式的
openssl rsa -in rsaprivatepkcs8.pem -out ssleay.pem
# 提取公钥
openssl rsa -in ssleay.pem -pubout -out rsapublickey.pem
```

#### 秘钥使用

``` java
public class RSAUtils {
private static final Logger LOGGER = LoggerFactory.getLogger(RSAUtils.class);
private static final String KEY_ALGORITHM = "RSA";
private static final String ENCODING = "utf-8";
private static final int MAX_ENCRYPT_BLOCK = 117;
private static final int MAX_DECRYPT_BLOCK = 128;
private static final String SIGNATURE_ALGORITHM = "SHA1WithRSA";

public static String encryptByPublicKey(String content, String publicKey) throws Exception {
byte[] data = content.getBytes(ENCODING);
byte[] keyBytes = Base64.decodeBase64(publicKey.getBytes(ENCODING));
X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(keyBytes);
KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
PublicKey publicK = keyFactory.generatePublic(x509KeySpec);
// 对数据加密
Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
cipher.init(Cipher.ENCRYPT_MODE, publicK);
int inputLen = data.length;
ByteArrayOutputStream out = new ByteArrayOutputStream();
try {
int offSet = 0;
byte[] cache;
int i = 0;
// 对数据分段加密
while (inputLen - offSet > 0) {
if (inputLen - offSet > MAX_ENCRYPT_BLOCK) {
cache = cipher.doFinal(data, offSet, MAX_ENCRYPT_BLOCK);
} else {
cache = cipher.doFinal(data, offSet, inputLen - offSet);
}
out.write(cache, 0, cache.length);
i++;
offSet = i * MAX_ENCRYPT_BLOCK;
}
byte[] encryptedData = out.toByteArray();
return Base64.encodeBase64String(encryptedData);
} finally {
IOUtils.closeQuietly(out);
}
}

public static String decryptByPrivateKey(String content, String privateKey) throws Exception {
byte[] encryptedData = Base64.decodeBase64(content);
byte[] keyBytes = Base64.decodeBase64(privateKey.getBytes(ENCODING));
PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(keyBytes);
KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
Key privateK = keyFactory.generatePrivate(pkcs8KeySpec);
Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
cipher.init(Cipher.DECRYPT_MODE, privateK);
int inputLen = encryptedData.length;
ByteArrayOutputStream out = new ByteArrayOutputStream();
try {
int offSet = 0;
byte[] cache;
int i = 0;
// 对数据分段解密
while (inputLen - offSet > 0) {
if (inputLen - offSet > MAX_DECRYPT_BLOCK) {
cache = cipher.doFinal(encryptedData, offSet, MAX_DECRYPT_BLOCK);
} else {
cache = cipher.doFinal(encryptedData, offSet, inputLen - offSet);
}
out.write(cache, 0, cache.length);
i++;
offSet = i * MAX_DECRYPT_BLOCK;
}
byte[] decryptedData = out.toByteArray();
return new String(decryptedData, ENCODING);
} finally {
IOUtils.closeQuietly(out);
}
}

private static PrivateKey getPrivateKey(String key)
throws InvalidKeySpecException, InternalException, UnsupportedEncodingException,
NoSuchAlgorithmException {
byte[] keyBytes = Base64.decodeBase64(key.getBytes(ENCODING));
PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
return keyFactory.generatePrivate(keySpec);
}

public static String signByPrivateKey(String privateKey, String content)
throws Exception {
PrivateKey priKey = getPrivateKey(privateKey);
Signature signature = Signature.getInstance(SIGNATURE_ALGORITHM);
signature.initSign(priKey);
signature.update(content.getBytes(ENCODING));
byte[] signed = signature.sign();
return new String(UrlBase64.encode(signed), ENCODING);
}

private static PublicKey getPublicKey(String publicKey)
throws InvalidKeySpecException, InternalException, NoSuchAlgorithmException,
UnsupportedEncodingException {
KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
byte[] encodedKey = Base64.decodeBase64(publicKey.getBytes(ENCODING));
return keyFactory.generatePublic(new X509EncodedKeySpec(encodedKey));
}

public static boolean verifySignature(String content, String signature, String publicKey)
throws SignatureException,
InvalidKeyException,
InternalException, InvalidKeySpecException, NoSuchAlgorithmException,
UnsupportedEncodingException {
Signature sign = Signature.getInstance(SIGNATURE_ALGORITHM);
sign.initVerify(getPublicKey(publicKey));
sign.update(content.getBytes(ENCODING));
return sign.verify(UrlBase64.decode(signature.getBytes(ENCODING)));
}
}
```
