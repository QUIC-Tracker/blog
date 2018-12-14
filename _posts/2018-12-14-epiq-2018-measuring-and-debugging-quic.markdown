---
layout: post
title:  "EPIQ workshop at CoNEXT 2018 - Session 3 - Measuring and debugging QUIC"
subtitle: "Session 3 - Measuring and debugging QUIC"
date:   2018-12-14 09:55:23 +0100
categories: EPIQ-2018 
---

On the first week of December, the first edition of the [*Workshop on the Evolution, Performance, and Interoperability of QUIC*][epiq] was held at [CoNEXT 2018][conext] in Heraklion, Crete.
We presented our paper [*Observing the Evolution of QUIC Implementations*][paper] and discussed with the researchers and engineers attending the workshop. 
In [this series of blog posts][posts], we report a summary of each session of the workshop as well as some notes we took.

[epiq]: https://conferences2.sigcomm.org/co-next/2018/#!/workshop-epiq
[conext]: https://conferences2.sigcomm.org/co-next/2018/
[paper]: https://dl.acm.org/citation.cfm?id=3284852
[posts]: {{site.baseurl}}/categories/#epiq2018

## Session 3 - Measuring and debugging QUIC
### Towards QUIC debuggability -- presented by Robin Marx (*Hasselt University*)

The presenter started the talk by explaining the launch of HTTP/2 in 2015. 
He noted that a similar sense of excitement regarding HTTP/2 was present as it now for QUIC, but the former quicly revealed itself as slow to deploy.
One reason is that in addition of being a more complex protocol than its predecessors, HTTP/2 lacked specialised tool to debug the new issues arising from its use.
HTTP/2 prioritisation is complex, but a good visualisation tool can make the problem more easy to understand.
Thanks to a few of such tools, bugs were discovered in HTTP/2 implementations.

The presenter ended its introduction by leveraging the experience with HTTP/2 when envisioning the future of QUIC. 
QUIC combines a transport layer protocol and a mapping for HTTP, called HTTP/3. It is thus even more complex, and everything has to be reimplemented.
Then the presenter introduced the tools proposed in their paper.
The first is a timeline that plots several events over time, such as the exchanged packets with different colours for the frames they contain (ACK, flow control or data).
The second is a sequence diagram that of exchanged packets that helps to visualise the flow of packets, the reordering or the losses that can occur.
It is based on the two hosts clocks to match sending and receiving time. The clocks difference can be manually adjusted.

To achieve this, the presenter advocates for a common logging format. Trying to parse the existing logs of QUIC implementations is a complex task.
Its form is usually not stable and can differ heavily from one to the other.
To overcome this problem, the presenter introduced *qlog*, a JSON-based log format from which all their visualisation tools are based on.
Because it uses JSON, it can easily be integrated into the web. The tools presented are actually run in the browser and the presenter also presented a way to access the logs using `.well-known` URLs. 
Being an open-format, the presenter hopes that several implementations can use it and that all the collected data can be stored in an open system.
The data collected should be freely accessible so that the QUIC community can analyse it and benefit from it.

The presenter will bring his work to the IETF, in the hope that it will stimulate discussions on debuggability within the working group.
The tools, slides and paper presented can be found at [https://quic.edm.uhasselt.be/](https://quic.edm.uhasselt.be/).

### Observing the Evolution of QUIC Implementations -- presented by Maxime Piraux (*UCLouvain*)

This is my own talk, you will have to wait for the recorded video to be published but you can already read [our paper][our-paper].

[our-paper]: https://arxiv.org/pdf/1810.09134

### Interoperability-Guided Testing of QUIC Implementations using Symbolic Execution -- presented by Felix Rath (*RWTH Aachen University*)

In their work, the authors investigated how state-of-the-art software testing techniques can be used in the context of interoperability testing between QUIC implementations. 
Their method does not require a formal specification of the protocol, solving the chicken-and-egg problem, by substituting interoperability using symbolic execution.
The authors consider two kinds of failure when testing two implementations against each other, an interoperability failure and a robustness failure.
The first arise when one of peer's view of the connection is not synchronised with the other, e.g. one thinks it is closed instead of opened.
The second usually manifests in the form of out-of-bound memory access or use-after-free violations.

Then the presenter introduced their methodology. They focused on testing the packet handling code of an implementation.
To do so, they use symbolic execution to follow all execution paths in this code.
They had to adapt the QUIC implementations to add an entry point and replace external libraries, such as TLS, by mocks.
They implemented several scenarios in which they modify parts of the packets.
OpenSSL was replaced by a mock that implements a null cipher.

The presenter reported the results of their case study focusing on the picoquic and quant implementations.
Both being written in C, they were an easy fit for the symbolic execution tool used, KLEE.
Using 6 scenarios and a relatively low coverage of about 40%, the authors were able to report several bugs as explained in detail in the paper.
Some bugs found are hard to explain, e.g. dropping packet 4, 5 and 7 resulted in a null pointer dereference when processing packet 9.
The presenter concluded by stating that a standardised way of sharing an implementation's belief state would ease and improve their work.

More details on their work can be found in [their paper][frath-paper]

[frath-paper]: https://dl.acm.org/citation.cfm?id=3284853

#### Acknowledgements

I would like to thank Quentin De Coninck for his notes during the workshop, as well as [Robin Marx][rmarx] for [his notes][rmarx-notes] that completed mines.

[rmarx-notes]: https://docs.google.com/document/d/16SZDhfR2IspQLQ8s_-FiKBZRgp2WJ02gtDZsWYsNVJ8/edit
[rmarx]: https://twitter.com/programmingart


