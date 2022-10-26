---
title: 一次前后端API交互数据加密
date: 2022-10-26 09:53:44
categories: ['开发']
tags: ['加密', 'AES', 'RSA']
---

一次合规要求，需要对关键接口进行加密传输。
处理流程如下：
![](/images/2022/10/1666750206503.png)

流程简化过，公私钥提前分配到前后端。
<!--more-->
### 前端传递密钥
前端生成随机32位（数字+字谜）AES密钥
```js
//随机字符串，生成密钥
function randomString(e) {
    e = e || 32;
    let t = "ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz1234567890",
        a = t.length,
        n = "";
    for (i = 0; i < e; i++) n += t.charAt(Math.floor(Math.random() * a));
    return n
}
//公钥加密密钥
function rsaEncrypt(secretKey) {
    const publicKey = "-----BEGIN PUBLIC KEY-----MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC+FMib8idOCv4WDYuc+7ROGXriloJGYLZiLDBmDz/Jrj5vraFPg6XEU1dr6Mx9XpuPPbnyaaN2R9IYWcuXixCt9YK9BQoVyzH2+XqmmIpYOh1nWV5Hba6bPC+aCgGN/w782lCGspElST3Q2wsI8f2E19IWH9Y56GnJVGziGxeD6wIDAQAB-----END PUBLIC KEY-----";
    //使用 jsencrypt
    let encryptor = new JSEncrypt()
    encryptor.setPublicKey(publicKey)
    return encryptor.encrypt(secretKey)
}
//BODY sha-256
//需要优化，不能实际使用，一般是通过对json key进行排序（按字典顺序）
function genSha256(data, ...keys) {
    let data2urparams = ""
    for (let i = 0; i < keys.length; i++) {
        let key = keys[i]
        data2urparams += key + '=' + data[key] + '&'
    }
    if (data2urparams.endsWith('&')) {
        data2urparams = data2urparams.substring(0, data2urparams.length - 1)
    }
    //nodejs使用 crypto-js
    return CryptoJS.SHA256(data2urparams).toString()
}
function demo() {
    let secretKey = randomString()
    console.log('AES密钥', secretKey)

    let requestBody = {
        'key': secretKey,
        'demo': '中文'
    }
    let hash256 = await genSha256(requestBody, 'key', 'demo')
    console.log(hash256)
    let encryptBody = rsaEncrypt(JSON.stringify(requestBody))
    console.log('加密后内容', encryptBody)
}
//例子
/*
未加密数据库
{
    'key': '2fC4Q4z6WeD8iQ4TnSP9eBjFR02Ejzw4',
    'demo': '中文'
}
sha-256
04327dc37a0ff5c627279c1687698222cac8205642b6af4abf36632db79f3c63
加密base64后数据
KLU7x0K4EEFE6vFVBgFu02hjLGq4HBZwX+W/SYatpzY6+yaEYwYdKX+1coyotVsZtXWL/BMEBCNJ3/8MBwcyOIEUZ8g7No4zWHIQzUNrWDlxC0+tGw05JuYSD1EKEyMPMzDgvxK7wTpTbtZfIrlDfvthE7bhmRns3v44zKNn9jE=
*/
```
### 后端解密数据
```java
//使用hutool解密
String privateKey = "MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAL4UyJvyJ04K/hYN" +
                "i5z7tE4ZeuKWgkZgtmIsMGYPP8muPm+toU+DpcRTV2vozH1em489ufJpo3ZH0hhZ" +
                "y5eLEK31gr0FChXLMfb5eqaYilg6HWdZXkdtrps8L5oKAY3/DvzaUIaykSVJPdDb" +
                "Cwjx/YTX0hYf1jnoaclUbOIbF4PrAgMBAAECgYB/+oRbIv49uH78oCAZEQuD7fnj" +
                "54xNED6b+L6ZaLj89FlLXe8XFz8b4TUiDXrpCjLYjanNwjxxnceh54uBO/t8yweI" +
                "7jskVoaWStotK5sgpTUjiF5hdLsR4RCsnH/zZ+pTf2+9PN4V6j4CIdFQjn5/rIFb" +
                "/HbzqfES88rz6dFCIQJBAOrhUwUikYRwNr9PSC1UuM/VRhEUAXGHLGRKmmLpCMNn" +
                "uqGYBMBPwIQOQPLmaCTpt5FC51GbBCwMwgWuyFebCrsCQQDPLDzD365dJWJwsOmH" +
                "A+mMHleE/+cELLtxQOHNTXGwGFNa+PvZHWO5W0gVMNC5Z0mPQp1teIzvmwO3lHC6" +
                "BFCRAkEAqcvEKXUo/zXjzf8xbVvO0qgaI+Rzeq++Xq4z14chR6moGIN+A8xjntNz" +
                "DmWUKgMvKfrUoIDQzktWw6bru7EgWwJBAIaRPHMacr6sDtIWB8ocP3I1LzIDqsHq" +
                "cGJy+3iISkVQt6wKuEPhtCns4dhp2dnj/kLgyTMXL6xfKz3uXH5nWRECQFCraEHB" +
                "Opw0MnLay6yKawaVbvgknHS4LjYSY5rzwKlo16Ab7EEtm2G8mnOOBB8r7U/FXgBJ" +
                "rhucymdoEn747T8=";
RSA rsa = new RSA(privateKey, null);
String body = "KLU7x0K4EEFE6vFVBgFu02hjLGq4HBZwX+W/SYatpzY6+yaEYwYdKX+1coyotVsZtXWL/BMEBCNJ3/8MBwcyOIEUZ8g7No4zWHIQzUNrWDlxC0+tGw05JuYSD1EKEyMPMzDgvxK7wTpTbtZfIrlDfvthE7bhmRns3v44zKNn9jE=";
byte[] decrypt = rsa.decrypt(body, KeyType.PrivateKey);
System.out.println(StrUtil.str(decrypt, CharsetUtil.CHARSET_UTF_8));
```

### 前端加密请求数据
```js
//aes加密，返回为base64
function aesEncrypt(secretkey, body) {
    let key = CryptoJS.enc.Utf8.parse(secretkey);
    let iv = CryptoJS.enc.Utf8.parse("1234567890123456");
    //加密模式为CBC，补码方式为PKCS5Padding（也就是PKCS7）
    let encrypted = CryptoJS.AES.encrypt(CryptoJS.enc.Utf8.parse(body), key, {
        iv: iv,
        mode: CryptoJS.mode.CBC,
        padding: CryptoJS.pad.Pkcs7
    })
    
    return CryptoJS.enc.Base64.stringify(encrypted.ciphertext);
}
//aes解密，body为base64
function aesDecrypt(secretkey, body) {
    let key = CryptoJS.enc.Utf8.parse(secretkey);
    //前后端同意iv
    let iv = CryptoJS.enc.Utf8.parse("1234567890123456");
    let decrypted = CryptoJS.AES.decrypt(body, key, {
        iv: iv,
        mode: CryptoJS.mode.CBC,
        padding: CryptoJS.pad.Pkcs7
    });
    return decrypted.toString(CryptoJS.enc.Utf8);
}
//例子
/*
密钥：2fC4Q4z6WeD8iQ4TnSP9eBjFR02Ejzw4
请求数据
{
    "id":1,
    "desc": "中文",
    "timestamp": 1666767634606
}
JSON.stringify(requestBody)后：'{"id":1,"desc":"中文","timestamp":1666767634606}'

加密后数据：E2ezeNGZRom4HA0OULidnOrBHWBgzbxVf2LCvTrnvqQzmjhODMSEyvJ9q98kp9uu8p07UMzng28K7rScmjfgdA==
*/
在实际开发过程中需要对请求数据进行sha-256，同上，此处简化
```
### 后端解密请求

```java
//无第三方依赖，解密
String key = "2fC4Q4z6WeD8iQ4TnSP9eBjFR02Ejzw4";
String iv = "1234567890123456";
String body = "E2ezeNGZRom4HA0OULidnOrBHWBgzbxVf2LCvTrnvqQzmjhODMSEyvJ9q98kp9uu8p07UMzng28K7rScmjfgdA==";

byte[] raw = key.getBytes(StandardCharsets.US_ASCII);
SecretKeySpec skeySpec = new SecretKeySpec(raw, "AES");
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
IvParameterSpec ivParameterSpec = new IvParameterSpec(iv.getBytes());
cipher.init(2, skeySpec, ivParameterSpec);
byte[] encrypted1 = Base64.getDecoder().decode(body);
byte[] original = cipher.doFinal(encrypted1);
System.out.println(new String(original, StandardCharsets.UTF_8));
```

```java
//无第三方依赖，加密
String key = "2fC4Q4z6WeD8iQ4TnSP9eBjFR02Ejzw4";
String iv = "1234567890123456";
String body = "{\"id\":1,\"desc\":\"中文\",\"timestamp\":1666767634606}";


byte[] raw = key.getBytes(StandardCharsets.UTF_8);
SecretKeySpec skeySpec = new SecretKeySpec(raw, "AES");
//如果传空，则取默认值，平台aes加密时，iv取默认值
IvParameterSpec ivParameterSpec = new IvParameterSpec(iv.getBytes(StandardCharsets.UTF_8));
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, skeySpec, ivParameterSpec);
byte[] encrypted = cipher.doFinal(body.getBytes(StandardCharsets.UTF_8));
System.out.println(Base64.getEncoder().encodeToString(encrypted));
```

