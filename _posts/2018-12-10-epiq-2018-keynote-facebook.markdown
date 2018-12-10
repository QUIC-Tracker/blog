---
layout: post
title:  "EPIQ workshop at CoNEXT 2018 - Keynote"
subtitle: "Opening keynote"
date:   2018-12-10 08:55:47 +0100
categories: EPIQ-2018 
---

On the first week of December, the first edition of the [*Workshop on the Evolution, Performance, and Interoperability of QUIC*][epiq] was held at [CoNEXT 2018][conext] in Heraklion, Crete.
We presented our paper [*Observing the Evolution of QUIC Implementations*][paper] and discussed with the researchers and engineers attending the workshop. 
In [this series of blog posts][posts], we report a summary of each session of the workshop as well as some notes we took.

[epiq]: https://conferences2.sigcomm.org/co-next/2018/#!/workshop-epiq
[conext]: https://conferences2.sigcomm.org/co-next/2018/
[paper]: https://dl.acm.org/citation.cfm?id=3284852
[posts]: {{site.baseurl}}/categories/#epiq2018

## Keynote: Moving fast at scale
### Experience deploying IETF QUIC at Facebook (Subodh Iyengar -- *Facebook*)

After a short introduction by the two chairs Jörg Ott (*TU Munich*), Lars Eggert (*NetApp*), the first presenter of the day was Subodh Iyengar from Facebook.
Given that an intriguing title was announced, but no abstract, this keynote was set to be a good starting point for the workshop.
During about an hour, the presenter detailed how Facebook is experimenting with IETF-QUIC inside its infrastructure but also on mobile clients through the Facebook app on Android and iOS devices.

The presenter outlined several requirements regarding the use of QUIC in place of TCP.

* Being able to update their proxies without disrupting connections.
* Being able to route consistently connections to application servers.
* Being able to connect using QUIC, but fallback to TCP without impacting latency.
* Being able to instrument QUIC and obtain logs to analyse performance issues.

### Zero downtime restarts in QUIC

Facebook proxies, based on [*Proxygen*][proxygen], are constantly restarted to update their software.
Ideally, existing connections should not be shutdown during the update.
No downtime should occur when establishing new connections during a proxy update either.
To achieve this, a server ID is encoded into the server-chosen `ConnectionID`.
A single bit is sufficient to distinguish between an old and a new server.
This bit is used by the new proxy instances to distinguish new incoming connections from existing connections.
These existing connections are UDP-encapsulated and routed to the old instances.

![Zero downtime restarts in QUIC](/blog/assets/fb_restarts.jpg){:style="max-width: 75%; display: block; margin: auto;"}

Once all old connections terminate, the proxy update completes.
This operation takes about 15 minutes.

[proxygen]: https://code.fb.com/production-engineering/introducing-proxygen-facebook-s-c-http-framework/

### Stable routing of QUIC packets

In the initial phases of deployment, the Facebook engineers noticed a high percentage of connection timeout using QUIC.
They first suspected dead connections to be the cause, but since they did not implement the stateless reset mechanism, they chose to do so in order to guarantee that each peer's view of the connection state is synchronised.
They found that they were unable to send these resets to the correct connections, indicating a packet routing problem.
To debug this issue, they again took advantage of the server-chosen `ConnectionID` and reserved a second part to indicate the server host ID.
Each server is now capable of logging packets that arrives at the wrong destination.

![Stable routing of QUIC packets](/blog/assets/fb_routing.jpg){:style="max-width: 75%; display: block; margin: auto;"}

After implementing the support for server host ID in [Facebook's L3 load balancer, *katran*][katran], they observed the number of misrouted packets to go down to 0.
The request latency was reduced by 15%, indicating that this problem affected a considerable amount of connections.
The presenter noted that they intend to use this ID when implementing futures features such as multi-path and anycast QUIC.

[katran]: https://github.com/facebookincubator/katran

### Pooling connections

They first started by analysing the amount of networks that allowed UDP connections.
Out of 25000 carriers, about 4000 of them reported no QUIC usage.
The presenter indicated that these carriers are not statistically significant, but he did not provide more accurate numbers.
Their team contacted some of them to enquire about this blockage. They report that some carriers intentionally block UDP outside of well-known usages, e.g. DNS.
Some others blocked it by accident.

Given this observation, they chose to race QUIC and TCP so that in the event of a fallback to TCP the latency is not impacted.
They used TCP with TLS 1.3 0-RTT to compare against QUIC, so that both require the same amount of RTTs to establish a connection.
They first went through a naive approach, i.e. start booth at the same time and cancel the other once one wins. Using this method they reported a 70% usage rate of QUIC.
The presenter noted probabilistic losses and TCP middleboxes speed up as possible causes for this result.

A second version of the racing algorithm was experimented with.
An arbitrary delay of 100ms was added to the start of the TCP connection, but no improvement in the QUIC use rate was observed.
The clients establishing the connections being mobile devices, they suspected the radio wakeup delay to impact their results.
Higher random losses at the beginning of QUIC connections were observed once more, they can probably be attributed to middleboxes.

![QUIC and TCP connection pooling](/blog/assets/fb_pooling.jpg){:style="max-width: 75%; display: block; margin: auto;"}

They third version of this algorithm is more complex but improved the QUIC utilisation rate to 93%.
First, the TCP delay is removed if QUIC loses the race, and added back on subsequent connections if QUIC wins.
Second, QUIC is not cancelled any more when TCP wins, and new requests are sent over QUIC if a connection is established.

The presenter concluded this section by presenting their use of 0-RTT data when pooling TCP and QUIC connections.
They chose a conservative approach, i.e. they cancel requests sent over QUIC if TCP + TLS 1.3 0-RTT succeeds.
Requests that fails over QUIC are replayed over TCP.

### Debugging QUIC in production

QUIC is a complex protocol in terms of the number of its possible actions. Answering the question of *Why was frame X sent at moment Y?* is thus critical when observing its behaviour.
To tackle this problem, the presenter introduced Facebook's trace format for QUIC, which logs the internal state of their implementation. It allows them to diagnose several events occurring in the life of a connection, such as congestion window blockages and loss recoveries duration.

They found that the ACK threshold for recovery, e.g. triggering Fast Retransmit, is not enough most of the time in their use-cases, because it may take several RTTs before enough packets have been received to trigger it.
They also observed HTTP connections to be idle most of the time.


![Debugging QUIC in production](/blog/assets/fb_debugging.jpg){:style="max-width: 75%; display: block; margin: auto;"}

### Results

The final section of the keynote was dedicated to performance results. The presenter first introduced their experimental setup.  
Facebook's QUIC implementation, *mvfst*, was deployed on mobile clients through the Facebook app and on their *proxygen* servers.
They implemented *draft-09*, which the presenter notes as quite stable although dating from January 2018.
The Cubic congestion controller was used when conducting their experiments, I assume it was in both TCP and QUIC but my notes are lacking on that matter.
They focused on API-style requests and response, not full HTML pages and assets. Requests sizes are comprised between 247 bytes and 13 KB while responses sizes are comprised 64 bytes and 500 KB.

![Results deploying QUIC](/blog/assets/fb_results.jpg){:style="max-width: 75%; display: block; margin: auto;"}

The results presented indicate reductions in overall latency of 6% for the 75th percentile, 10% for the 90th percentile and 23% for the 99th percentile.
When isolating performance results for later requests, i.e. requests sent in later stages of the connection  life, the observed latency reduction was of 1%, 5% and 15% for these percentiles.
The presenter noted similar latency reduction when isolating connections which experienced an RTT smaller than 500ms.

Only relative numbers were presented and no absolute numbers are given. The presenter explained he was not allowed to share those numbers, but assured they were significant in practice.
He concluded the keynote by explaining that while these initial results using an earlier version of QUIC and HTTP/1.1 were promising, a lot of experimentations remain to be conducted.
Major changes to the infrastructure are also required to use QUIC at large scale, as presented in this keynote. 

Finally, the presenter concludes by reviewing some future works, such as updating *mvfst* to the latest QUIC specification draft, adding HTTP/3 and 0-RTT support. Their team will also explore new use-cases such as partial reliability for live videos and low latency video delivery.

### Q&A session

The first question from the audience related to the fairness of *mvfst* with regard to TCP and how responsible it is to deploy QUIC at scale.
The presenter explained that they have not looked at it in detail.
Overall they did not observe problems related to congestion and fairness, because the data exchanged over QUIC is relatively small.
The presenter noted that a lot of connections do not exit slow start, as they are very short in time.

A follow-up question regarding CPU performance is brought by the audience.
He explained that they did not investigate the impact on battery life of mobile clients, but they observed an increase in CPU usage overall, some of it is due to the `sendmsg` call.

The next question related to the use of QUIC inside their backbone in addition to client-proxy communications, and the differences they observed between the two.
The presenter explained that they did not observe differences in backbone CPU usage. He noted that the loss rate is close to 0 and connections are frequently reused in the backbone.

A short question about *mvfst* availability is brought, the presenter announces that both client and server will eventually be open-source.

One person in the audience was concerned about their use of the `ConnectionID` field, which he notes as abusive and encouraging layer violations. The presenter commented that some work is being done in the IP layer to cope with these issues. He noted that the `ConnectionID` can also be used for other purposes than packet routing.

The final question was about how QUIC inside and outside the backbone coexist. The presenter explains that the edge proxies terminate the QUIC connections with the clients and establish new ones in the backbone.
#### Acknowledgements

I would like to thank François Michel for the pictures found in this article, Quentin De Coninck for his notes during the workshop, as well as [Robin Marx][rmarx] for [his notes][rmarx-notes] that completed mines.

[rmarx-notes]: https://docs.google.com/document/d/16SZDhfR2IspQLQ8s_-FiKBZRgp2WJ02gtDZsWYsNVJ8/edit
[rmarx]: https://twitter.com/programmingart
