### On Security 

### Prologue 

### I. Redis Horror Story #1
All seasoned security professionals have their fair share of security horror stories. Some of those stories involve Redis. To show you just what's possible in the real world, I'm going to share a different Redis security horror story at the beginning of each week. This week's story is [Redis Wannamine](https://www.imperva.com/blog/archive/new-research-shows-75-of-open-redis-servers-infected/).

Redis Wannamine is the name for a Redis exploit that used a combination of vulnerabilities to run crypto mining software on Redis servers. Attackers first scanned the internet, looking for applications using [Apache Struts](https://struts.apache.org/). The attack then exploited a known vulnerability in Apache Struts to run commands on the server to target Redis. The attack then used Redis to create a cron job and install a *crypto miner* in a Redis data file known as an RDB file.

Note: A crypto miner, also known as a cryptocurrency miner or crypto-mining software, is a program or software application that utilizes computational power to solve complex mathematical problems, validate transactions, and secure a blockchain network in exchange for rewards in the form of cryptocurrencies.

Imagine the poor administrator who set this up. They went through a ton of trouble to ensure that their Redis was closed to the internet, and they may have even ensured Redis was only accessible on the local host. Still, the attackers were able to hijack the Redis server because of other vulnerabilities in the system. Talk about feeling like you don't control your own destiny. But what could have prevented this attack? Well, outside of securing external systems, setting a strong password on Redis or changing the default port could have prevented this exploit entirely.

This administrator probably thought they'd done everything right, so they decided they could treat themselves and forget the password to Redis. Unfortunately, that's not how it works in the world of security. As a result, that administrator ended up enriching the attackers. At the time, one [Monero](https://en.wikipedia.org/wiki/Monero) coin was worth between $80 and $100. 

Bottom line-- your CPU is worth something to attackers. It's not always the data they're after. And it's not enough, in all cases, to secure Redis from the outside. It's usually best to secure it from the inside as well.


### II. The CIA Triad
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


### III. Risk Assessment
**Objective**: By the end of the text exercise, the learner will understand basic security concepts that will help them establish a foundation for making informed risk management decisions.

Risks are everywhere. But it's rarely practical to secure against every conceivable risk. What this means is that security is not always about securing "all the things."

Rather, security is about making informed risk management decisions. A good security plan makes clear what risks it's prioritizing.

Accepting risk is okay; which risks you choose to accept are entirely dependent on your unique situation. This is the art of risk assessment. The risk assessment process follows the steps outlined in the diagram below.

![alt risk assessment process](img/risk_assessment_process.png)

#### Establish Context
The first and most critical part of the risk assessment process is establishing context. What do we mean by context? Let's take an example.

Suppose you're a hospital processing health records and making them available online. Your security decisions will be completely different from those of a website that generates memes.

So you consider the context of a security risk by measuring the impact of an identified security issue and the likelihood of a risk being exploited.

We can all probably agree that the impact of a hospital data breach is much more serious that a meme generating website breach.

#### Assess Risk
Once you establish context, you perform a risk assessment. To cite the Wikipedia definition, a risk assessment is "an assessment of the possible mishaps, their likelihood and consequences and your tolerances for such events."

First you must identify the mishaps.

For this example, let's suppose our hospital has discovered that unauthorized users can access other users' health records using a public API. This may occur if the API doesn’t have access controls built into it. would allow unauthorized users to access other users' data.

Our meme website may have found that all employees have full database access, allowing anyone to drop all the tables. This might occur if the database is protected by a corporate network. In this case, an administrator might not bother to enable authentication.

So, here we've established a couple of risks for our hospital and meme generator website.

#### Analyze the Situation
Once we understand the possible risks, we need to analyze them. Analysis is an assessment of the impact the identified risk would have on a business if the risk were exploited.

In our healthcare example, a data breach may be reportable to the government. Such a breach might also make the news, damaging the hospital's reputation and opening it to litigation by the patients whose data has been exposed.

In our meme generator example, a dropped database might prevent users from visiting the site, causing anything from minor annoyance to an exodus of users to a competing website.

#### Evaluate the Risk
Once we've analyzed the potential impact of each risk, the next step is to evaluate the cost of the impact if a risk materializes.

Suppose we're storing 5000 records in our healthcare application. According to a research study by IBM ("The 2019 Cost of a Data Breach"), the average cost of a breached healthcare record is $429. This means that a compromise of all records could cost us over 2 million dollars.

Now, we believe that the probability of someone exploiting our application is 25%. We can use the technique of mathematical expectation to estimate that the cost of failing to mitigate this risk is about half a million dollars. This does not include potential damage to our reputation, which is harder to define.

Now let's apply the same thinking to our meme generator.

The cost of a dropped database for our meme website is significantly less than half a million dollars. The meme website makes a few cents in advertising revenue every time a user visits the site.

What we don’t know with 100% certainty is how many times a user will come back and at what point issues will cause our user to abandon our site altogether. Sometimes risks are hard to quantify, so we have to take our best guess.

#### Risk Decision
Once we’ve evaluated each risk, it's time to make a decision about what we want to do about the risk. We need to base this decision on an analysis of cost to mitigate it compared to the potential cost of the risk.

Risks can either be accepted, mitigated, transfered or avoided.

1. *Accepting* means the organization will do nothing.
2. *Mitigating* means that the organization will take action to reduce the likelihood of the risk.
3. *Transferring* means moving the cost of the risk to a third party, such as an insurance provider.
4. *Avoiding* means disbanding the project or resolving the issue entirely.

The hospital provider will likely choose to mitigate or avoid the risk due to the high cost. The cost of the risk is probably greater than the cost of writing the code to fix the application.

Mitigating the risk may involve putting the public API behind a firewall and adding authentication and access controls.

The meme provider will likely accept or mitigate their issue. The risk to the meme provider is lower because the cost is not high; however, adding authentication to the database is relatively simple. They may choose to mitigate the risk by adding authentication.

No matter what we decide, we'll always want to continue monitoring the decision to see if the risk materializes and decide whether a different action needs to be taken.

#### In Summary
For those of you who aren’t security engineers, know that what we've described above is what occurs in the course of any standard risk assessment.

It's important to remember that no risk management strategy is appropriate for every situation. While the term "secure all the things" is catchy, many have learned the hard way that it's not always the most expedient or most cost-effective approach.

The bottom line is that risk assessments and decisions are business-driven. When deploying and configuring Redis, like any database, you should keep this in mind to ensure that your approach to deploying Redis aligns with your organization's risk management policies.


### IV. Defense in Depth


### Biblipgraphy 

### Epilogue 

### EOF (2024/06/28)
