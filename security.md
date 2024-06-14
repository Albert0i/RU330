### On Security 

### Prologue 

### Redis Horror Story #1
All seasoned security professionals have their fair share of security horror stories. Some of those stories involve Redis. To show you just what's possible in the real world, I'm going to share a different Redis security horror story at the beginning of each week. This week's story is [Redis Wannamine](https://www.imperva.com/blog/archive/new-research-shows-75-of-open-redis-servers-infected/).

Redis Wannamine is the name for a Redis exploit that used a combination of vulnerabilities to run crypto mining software on Redis servers. Attackers first scanned the internet, looking for applications using [Apache Struts](https://struts.apache.org/). The attack then exploited a known vulnerability in Apache Struts to run commands on the server to target Redis. The attack then used Redis to create a cron job and install a <a title="A crypto miner, also known as a cryptocurrency miner or crypto-mining software, is a program or software application that utilizes computational power to solve complex mathematical problems, validate transactions, and secure a blockchain network in exchange for rewards in the form of cryptocurrencies.">crypto miner</a> in a Redis data file known as an RDB file.

Imagine the poor administrator who set this up. They went through a ton of trouble to ensure that their Redis was closed to the internet, and they may have even ensured Redis was only accessible on the local host. Still, the attackers were able to hijack the Redis server because of other vulnerabilities in the system. Talk about feeling like you don't control your own destiny. But what could have prevented this attack? Well, outside of securing external systems, setting a strong password on Redis or changing the default port could have prevented this exploit entirely.

This administrator probably thought they'd done everything right, so they decided they could treat themselves and forget the password to Redis. Unfortunately, that's not how it works in the world of security. As a result, that administrator ended up enriching the attackers. At the time, one [Monero](https://en.wikipedia.org/wiki/Monero) coin was worth between $80 and $100. 

Bottom line-- your CPU is worth something to attackers. It's not always the data they're after. And it's not enough, in all cases, to secure Redis from the outside. It's usually best to secure it from the inside as well.


### The CIA Triad
The CIA triad describes the three most basic goals of information security: Confidentiality, Integrity, and Availability.

![alt The CIA Triad](img/cia_triad.png)

#### Confidentiality
Confidentiality is the property, that information is not made available or disclosed to unauthorized individuals, entities, or processes.

Suppose you send a letter in the mail to your sibling on their birthday. You want to ensure that the letter is only accessible to your sibling. To help ensure the confidentiality of the letter, you seal it in an envelope.

The letter is confidential because it's sealed in an envelope so that no one can see its contents. This is certainly more secure than a postcard, where anyone who handles it can read it.

#### Integrity
Integrity is the assurance that a message is not modified in transit.

A common technique for ensuring the integrity of a letter is signing the sealed flap and then comparing the signature on the flap to a known signature. A known signature over the flap of an envelope tells us two things: first, that the contents of the letter haven't been modified, and second, that the letter was sealed by a known sender.

#### Availability
Availability, which is the final concept in the CIA triad, is the property that the services you need are available for use when you need them.

For example, when you send a letter, you're relying on an operational postal service. You're making the assumptions that:

1. A mail carrier will pick up your letter.
2. A sorting facility will reliably route your letter to another post office.
3. A mail carrier at the other post office will deliver your letter to its intended recipient.

While securing a letter and securing a Redis deployment differ wildly in practice, you can refer to the CIA triad in both cases to plan your security strategy.

As security engineers or administrators, it's our responsibility to ensure that we always take confidentiality, integrity, and availability into consideration when we're designing secure systems. We'll frequently revisit these concepts throughout the course, so make sure you understand them before moving on.


### Biblipgraphy 

### Epilogue 

### EOF (2024/06/28)
