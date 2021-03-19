+++
tags = ["php"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "php对后台返回数据进行加密"
date = "2020-04-29T22:32:49+08:00"
+++


# php对后台返回数据进行加密，前端进行解密

## 生成加密密钥

生成私钥：

```sh
openssl genrsa -out rsa_1024_priv.pem 1024

```

生成对应的公钥：

```sh
openssl rsa -pubout -in rsa_1024_priv.pem -out rsa_1024_pub.pem

```

## php加密解密代码

**注意:**
​明文长度最大为公钥长度-11，假如我的公钥长度是128，那明文最长也就117，所以下面加密我使用了循环，分段加密

```php
if (!function_exists('privateDecrypt')) {
    /**
     * //解密
     * @param string $encryptString
     * @return string
     */
    function privateDecrypt($encryptString = '')
    {
        $privateKey = storage_path('key/exam.key');
        $decrypted = '';
        foreach (explode("__&_&__", base64_decode($encryptString)) as $chunk) {
            openssl_private_decrypt($chunk, $decryptData, file_get_contents($privateKey));
            $decrypted .= $decryptData;
        }
        return $decrypted;
    }
}

if (!function_exists('publicEncrypt')) {
    /**
     * //加密
     * @param string $data
     * @return string
     */
    function publicEncrypt($data = '')
    {
        $publicKey = storage_path('key/exam_pub.key');
        $encrypt_data = '';
        foreach (str_split(json_encode($data), 117) as $chunk) {
            openssl_public_encrypt($chunk, $encryptData, file_get_contents($publicKey));
            $encrypt_data .= base64_encode($encryptData) . "__&_&__";
        }
        return $encrypt_data;
    }
}

```

## 前端js的解密：

encrypted为加密后的数据

```js
<script src="http://code.jquery.com/jquery-1.8.3.min.js"></script>
<script src="bin/jsencrypt.min.js"></script>
<script type="text/javascript">
$(function () {

    //公钥
    var pub_key = '-----BEGIN PUBLIC KEY-----\n' +
        'MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDlOJu6TyygqxfWT7eLtGDwajtN\n' +
        'FOb9I5XRb6khyfD1Yt3YiCgQWMNW649887VGJiGr/L5i2osbl8C9+WJTeucF+S76\n' +
        'xFxdU6jE0NQ+Z+zEdhUTooNRaY5nZiu5PgDB0ED/ZKBUSLKL7eibMxZtMlUDHjm4\n' +
        'gwQco1KRMDSmXSMkDwIDAQAB\n' +
        '-----END PUBLIC KEY-----';

    //new JSEncrypt
    var js_encrypt = new JSEncrypt();
    //初始化公钥
    js_encrypt.setPublicKey(pub_key);

    //通过 公钥 解密
    var uncrypted = js_encrypt.decrypt(encrypted);
    console.log(uncrypted);
});
</script>

```