---
title: HTTPS and OpenSSL
date: 2014-06-09 21:11:29
tags:
  - HTTPS
  - TLS/SSL
  - OpenSSL
---

一、HTTPS
---------

HTTPS协议使用SSL协议建立了安全的HTTP通道。SSL协议使用对称加密算法对数据进行加密。使用对称加密算法的原因主要是强度高、速度快。但是对称加密使用到的密钥必须使用非对称加密算法才能在网络上安全传输，该过程即SSL握手。SSL会话总是以SSL握手开始。

SSL握手过程大致如下：

<!-- more -->

1. 客户端建立与服务端的连接，然后向服务端发送客户端的SSL版本号、加密设置、随机数据，以及其它会用到的一些信息。

2. 服务端接收到客户端的数据后，向客户端发送服务端的SSL版本号、加密设置、随机数据，以及其它会用到的一些信息。服务端也会同时把自己的数字证书（服务端公钥）发送给客户端，该数字证书应该由第三方受信任的CA机构颁发，即该数据证书拥有第三方受信任的CA机构的数字签名（CA私钥加密）。

3. 客户端使用从服务端收到的信息来对服务端进行验证。首先，客户端本身维护着所有第三方受信任的CA机构的证书（CA公钥），该证书用于验证服务端发送过来的证书是否是由第三方受信任的CA机构颁发的。其次，客户端也会验证服务端证书的认证信息，核对该证书是否是颁发给该服务端的。如果验证失败则向用户发出警告，如果验证成功则继续进行。

4. 客户端生成用于对后续会话进行对称加密的密钥，然后使用从服务端收到的证书来对该密钥进行加密（服务端公钥加密）。加密完成后，将加密的结果发送给服务端。

5. 服务端收到客户端发送过来的经过加密的密钥后，使用自己的私钥进行解密，得到对称加密的密钥。

6. 现在，对称加密的密钥已经安全地从客户端传送到了服务端，后续的会话则使用该密钥进行加密。

SSL握手的一个主要目的就是将客户端生成的对称加密的密钥安全地传送到服务端，而这个过程就是依靠服务端的受信任的证书来达成的。首先，由于数据经过服务端给出的证书（公钥）加密，中间人拦截到数据后是无法解密的；其次，服务端证书是由第三方受信任的CA颁发的，中间人无法伪造成服务端给出自己的证书。

有些软件，类似GoAgent这样的HTTPS代理，需要用户手动导入一个受信任的CA证书。这样是比较危险的。因为一旦该CA证书是公开的，那么就可能有人利用该CA证书去签发一个自己的证书，然后对信任它的浏览器进行中间人攻击。一个办法是自己再生成一个私密的证书来替换掉公开的证书，然后再导入为信任证书。

OpenSSL是一个用于SSL相关加密、解密、生成证书、验证证书等工具的一个库/命令。其用法在[\[3\]][3]和[\[4\]][4]中可以找到。

二、OpenSSL工具
---------------

1、用openssl进行信息摘要。

```bash
openssl dgst -md5 < input.file > output.file
openssl md5 < input.file > output.file
# md5 可以用 sha 等其他方法代替，详见 openssl dgst -h 
```

2、用openssl进行对称加密解密。

```bash
# 加密
openssl enc -e -des3 -in input.file -out output.file

# 解密
openssl enc -d -des3 -in output.file  # 注意解密算法必须与加密时一致

# 可以使用 -a / -base64 选项来产生 base64 编码的加密结果
openssl enc -e -a -des3 -in input.file

# 那么解密时也需要加上该选项
openssl enc -d -a -des3

# 当然也可以不使用 -a / -base64 选项而是使用 openssl base64 / base64 命令
# 可以省略 -in 和 -out 而使用 stdin 和 stdout
# 除了 des3 还可以使用 aes 等加密算法 ，详见 openssl enc -h
```

3、用openssl进行非对称加密解密。

```bash
# 先生成 RSA 的私钥 rsa_key。注意该私钥必须严格保密且大于等于2048位
openssl genrsa -out rsa_key 2048

# 然后通过该私钥生成公钥 rsa_key.pub。该公钥可以通过网络传输给他人。
openssl rsa -in rsa_key -pubout -out rsa_key.pub

# 他人接收到该公钥后，即可用该公钥对数据 secret.txt 进行加密，结果在 result.bin。
openssl rsautl  -encrypt -in secret.txt -inkey rsa_key.pub -pubin -out result.bin

# result.bin 可安全传输于网络。收到后即可用私钥解密。 
openssl rsautl -decrypt -in result.bin -inkey rsa_key -out origin.txt

# 若不使用-in或-out，则输入输出对应到stdin或stdout。 
# 若使用私钥加密公钥解密，即数字签名，则 -encrypt 应换成 -sign，-decrypt 应换成 -verify 
```

4、用openssl进行CA根证书的制作，以及用CA根证书签发SSL证书。

```bash
# 由于 openssl ca 命令的使用依赖于特定的目录结构，所以建议使用专门的目录来完成后续的操作
mkdir -p nicky/CA/{newcerts,private} && cd nicky

# 该目录结构的定义存在于 /etc/ssl/openssl.cnf 中，cp过来之后根据需要和实际条件修改一些条目，比如CA证书、私钥的路径和文件名;
cp /etc/ssl/openssl.cnf .

# 生成用于制作CA证书的RSA私钥
openssl genrsa -out CA/ca.key 4096

# 制作CA证书
openssl req -new -x509 -key CA/private/ca.key -out CA/ca.crt -days 3650
```

**在生成SSL证书之前，需要修改openssl.cnf文件，主要为了加上Alternative DNS Names：**

```bash
...
[ req ]
...
req_extensions = v3_req
...
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
...
[ alt_names ]
`DNS.1 = xxx.my-site.com`
`DNS.2 = yyy.my-site.com`
```

接下来可以开始生成SSL证书：

```bash
# 生成SSL证书私钥
openssl genrsa -out my-site.key 4096

# 生成SSL证书申请文件
openssl req -new -key my-site.key -out my-site.csr -config ./openssl.cnf -extensions v3_req

# 签发SSL证书
openssl ca -config ./openssl.cnf -in my-site.csr -out my-site.crt -extensions v3_req -days 365
```
<br>

##### 参考资料：

[1] [http://www.pierobon.org/ssl/ch2/detail.htm][1]

[2] [http://www.tldp.org/HOWTO/SSL-Certificates-HOWTO/x64.html][2]

[3] [http://users.dcc.uchile.cl/~pcamacho/tutorial/crypto/openssl/openssl_intro.html][3]

[4] [http://rhythm-zju.blog.163.com/blog/static/310042008015115718637/][4]


[1]: http://www.pierobon.org/ssl/ch2/detail.htm

[2]: http://www.tldp.org/HOWTO/SSL-Certificates-HOWTO/x64.html

[3]: http://users.dcc.uchile.cl/~pcamacho/tutorial/crypto/openssl/openssl_intro.html

[4]: http://rhythm-zju.blog.163.com/blog/static/310042008015115718637/

