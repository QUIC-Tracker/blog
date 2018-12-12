---
layout: post
title:  "EPIQ workshop at CoNEXT 2018 - Session 2 - QUIC and media"
subtitle: "Session 2 - QUIC and media"
date:   2018-12-11 08:55:47 +0100
categories: EPIQ-2018 
---

On the first week of December, the first edition of the [*Workshop on the Evolution, Performance, and Interoperability of QUIC*][epiq] was held at [CoNEXT 2018][conext] in Heraklion, Crete.
We presented our paper [*Observing the Evolution of QUIC Implementations*][paper] and discussed with the researchers and engineers attending the workshop. 
In [this series of blog posts][posts], we report a summary of each session of the workshop as well as some notes we took.

[epiq]: https://conferences2.sigcomm.org/co-next/2018/#!/workshop-epiq
[conext]: https://conferences2.sigcomm.org/co-next/2018/
[paper]: https://dl.acm.org/citation.cfm?id=3284852
[posts]: {{site.baseurl}}/categories/#epiq2018

## Session 2 - QUIC and media
### Real-time Audio-Visual Media Transport over QUIC -- presented by Colin Perkins (*University of Glasgow*)

The presenter started by emphasising how important is audio/video stream in the global Internet traffic volume.
However, the transport protocols involved in this traffic are TCP or other unreliable real-time protocol such as WebRTC over UDP.
These protocols suffer from several drawbacks: TCP suffers from the well-known head-of-line blocking problem which can induce a high latency, while WebRTC trades off a lower latency for a higher amount of losses.
Because WebRTC is not based on HTTP, it is less familiar to developers and benefits of less CDN support which in turns makes it harder to deploy.
QUIC in itself is not a perfect fit, it suffers too from HoL blocking on a per-stream basis.

While various approaches can work around the HoL, the best answer to this problem is adding partial reliability to QUIC explained the presenter.
In the rest of the presentation, the presenter introduced their design which maps real-time capabilities to QUIC.
Instead of adding a simple unreliable datagram capability, they chose to reimplement common RTP features, such as timing, sequencing and loss tolerance, that are required by real-time applications.

Their complete design can be found in [their paper][perkins-ott].

[perkins-ott]: https://dl.acm.org/authorize?N664180

### The QUIC Fix for Optimal Video Streaming -- presented by Mirko Palmer (*Max-Planck-Institut für Informatik*)

The presenter introduced the first part of their work as an investigation on the capabilities of reliable transport protocols, such as TCP and QUIC, to transport video streaming traffic.
TCP is often used because of its large support by CDNs for example.
Then the presenter introduced several figures, claiming that both TCP and QUIC are not well suited for video streaming. 
They measured the percentage of time spent rebuffering when streaming videos in several networks with increasing loss rate, showing for instance that TCP spent an equal amount of time rebuffering frames than playing them in a network with random losses at a rate of 0.64%.
QUIC achieved a 30% rebuffering, i.e. being able to play three frames played and then rebuffering one frame in average.

A first approach to overcome this problem could be to leverage the different types of images that are streamed in a video. I frames are critical, because they convey a complete frame and thus must be delivered reliably.
P and B frames on the contrary can be delivered in a best-effort manner. Their loss will be recovered after receiving an I frame. Then the presenter explained their approach, which consists of adding unreliable streams to QUIC.
These types of streams do not guarantee reliability. Using these streams, they transport the P and B frames of a video chunk while the I frames are still delivered in reliable manner.
Their unreliable streams also support forward erasure correction (FEC) and implements the Reed-Solomon scheme.

To evaluate their implementation, they review the same network environments as presented at the beginning of the talk. 
They showed that using their approach, the time spent rebuffering is below one percent in a network with random losses at a rate of 0.64%. 
They adapted the SSIM metric to account for stall events in the perceived frame quality and evaluated their design. 
By matching Mean Opinion Score (MOS) to the new SSIM metric, their approach is rated as *fair* when no FEC is used and *excellent* when FEC is used, while QUIC and TCP offer *poor* or *bad* experience.

During the Q&A session, a question is brought about whether their work targets live video streaming or any kind of video streaming.
The presenter answered that they considered video-on-demand and streaming of live events, but not video conferencing.
A followup question focused on the use of buffers during their experiments, as they are often used in VOD solutions.
The presenter explained that they used a single-frame buffer in their experiments, and that no buffers are needed as their solution experiences very few stalls in lossy networks.
A part of the audience is confused by this statement, as this choice degrades the baseline performance of reliable transport protocols and thus weaken their evaluation.
It also may not correspond to the solutions commonly deployed today, e.g. video streaming on YouTube is transported via reliable protocols.

#### Acknowledgements

I would like to thank Quentin De Coninck for his notes during the workshop, as well as [Robin Marx][rmarx] for [his notes][rmarx-notes] that completed mines.

[rmarx-notes]: https://docs.google.com/document/d/16SZDhfR2IspQLQ8s_-FiKBZRgp2WJ02gtDZsWYsNVJ8/edit
[rmarx]: https://twitter.com/programmingart


