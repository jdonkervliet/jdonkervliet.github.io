+++
title = 'Research'
menus = 'main'
weight = 60
+++

In my PhD, I focused on increasing the scalability of modifiable virtual environment (MVE) instances.
To this end, I designed several novel systems and approaches:

I designed **[Servo](https://atlarge-research.com/pdfs/2023-donkervliet-icdcs-servo.pdf)**, the first system to leverage serverless (and in particular FaaS) computing to improve the scalability and performance of game instances.
By identifying and isolating predictable parts of the game's simulation, Servo can use computational offloading to allow fine-grained horizontal scaling within a single game instance,
increasing the number of supported players by up to 15×.

Another research highlight is my work on **[Dyconits](https://atlarge-research.com/pdfs/icdcs21-dyconit-paper.pdf)**, a novel consistency model based on [conits](https://dl.acm.org/doi/10.1145/566340.566342) that enables using tunable consistency for online games and virtual worlds.
Dyconits is unique in that it simultaneously supports quantifying inconsistency, optimistically bounding inconsistency, and automatic policy-based consistency tuning.
Specifically, Dyconits can temporarily allow increased inconsistency to accommodate workload peaks and avoid performance degradation,
and increases the number of supported players by up to 40%.

To allow quantitative research on games such as Minecraft, I created **Yardstick** [[1](https://atlarge-research.com/publications.html#:~:text=Yardstick), [2](https://atlarge-research.com/pdfs/2023-jeickhoff-Meterstick-ICPE2023.pdf)], a distributed benchmark for Minecraft-like games.
Yardstick is the first benchmark of its kind,
and allows comparing the performance of alternative implementations of games that implement the Minecraft network protocol.
To evaluate system scalability, Yardstick uses scalable workloads of emulated players.
Using Yardstick, we showed that Minecraft-like games scale poorly (a few hundred players under good conditions) and that player modifications to the virtual world can cause significant performance degradations and system crashes (because the world is effectively programmable).

I am happy to have collaborated with many students during these research projects.
Most of these collaborations led to master's and bachelor's theses, and some helped students get their first scientific publication.  
[More about supervision highlights](/teaching#highlights)

Jump to: [Publications](#publications), [Service](#service)

# Publications

{{< publications >}}

# Service

- Reviewer for [ICPE 2026](https://icpe2026.spec.org/)
- Reviewer for [IEEE Transactions on Services Computing](https://www.computer.org/csdl/journal/sc), June 2025
- Reviewer for [IEEE Internet Computing](https://www.computer.org/csdl/magazine/ic), April 2025
- Editor for the [SPEC RG](https://research.spec.org/) Newsletter 2024
- Web chair for [CCGRID 2024](https://2024.ccgrid-conference.org/)
- Artifact reviewer for [CCGRID 2024](https://2024.ccgrid-conference.org/)
- Subreviewer for [CCGRID 2023](https://ccgrid2023.iisc.ac.in/)
- ⭐️ Created end-of-chapter exercises for **Computer Networks, 6th edition**, Tanenbaum, Feamster, Wetherall. 2018. ([link](https://computernetworksbook.com/))
