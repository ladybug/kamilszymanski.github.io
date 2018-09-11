---
title: Interpreting jstat's number of Full GC events
date: 2016-02-04T23:58:26+01:00
excerpt: "How results of your measurements can drift from reality when you don't validate your assumptions"
tags:
  - JVM
---

"You can't control what you can't measure"[^1] and you want to control things, you don't want to just bet on luck.
Nonetheless measuring is not enough, you also need to know how to interpret results of such measurements.

It's not hard to find examples of misinterpreted measurement results with the `free`[^2] command output being the most commonly misinterpreted one.
Often lack of understanding of used terminology or it's ambiguity can be accounted for it.
For example let's look at `jstat` output for a Java process:

```sh
$ jstat -gc -t 4648 1s
Timestamp        S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
           19.4 8704.0 8704.0  0.0   8704.0 69952.0  12416.7   240928.0   173395.8  154352.0 149283.9 22180.0 20722.4     27    0.594   8      0.231    0.825
```

Anyone knowing JVM Garbage Collection basics should have no trouble telling what those values mean, shouldn't he?
Well let's try: EC stands for Eden space Capacity, EU denotes Eden space Usage, S0C - Survivor space 0 Capacity, YGC - number of Young generation Garbage Collections, YGCT - Young generation Garbage Collection Time, and so on.

Pretty obvious, isn't it?
So let's gather statistics for the old generation (and metaspace) and answer the question how many Full GCs were performed?

```sh
$ jstat -gcold -t 5007 1s
Timestamp          MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
           11.7 113064.0 109246.8  16156.0  14883.7    174784.0    115285.4     13     4    0.079    0.368
```

If your answer is 4 you might be correct or far from being correct.
It all depends on which garbage collector monitored Java process is using.
For Parallel GC 4 is the correct answer, but for CMS the correct[^3] answer is 2.
If you don't believe me check the GC logs from the same Java process run:

```sh
0.408: [GC (Allocation Failure) 0.408: [ParNew: 69952K->8704K(78656K), 0.0396199 secs] 69952K->21786K(253440K), 0.0397117 secs] [Times: user=0.12 sys=0.02, real=0.04 secs] 
1.213: [GC (Allocation Failure) 1.213: [ParNew: 78656K->8704K(78656K), 0.0115476 secs] 91738K->29967K(253440K), 0.0116237 secs] [Times: user=0.06 sys=0.00, real=0.00 secs] 
2.152: [GC (Allocation Failure) 2.152: [ParNew: 78656K->8704K(78656K), 0.0176088 secs] 99919K->38548K(253440K), 0.0176831 secs] [Times: user=0.11 sys=0.00, real=0.02 secs] 
2.170: [GC (CMS Initial Mark) [1 CMS-initial-mark: 29844K(174784K)] 39075K(253440K), 0.0021170 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
2.172: [CMS-concurrent-mark-start]
2.185: [CMS-concurrent-mark: 0.013/0.013 secs] [Times: user=0.08 sys=0.00, real=0.02 secs] 
2.185: [CMS-concurrent-preclean-start]
2.186: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.186: [CMS-concurrent-abortable-preclean-start]
2.637: [CMS-concurrent-abortable-preclean: 0.169/0.451 secs] [Times: user=2.13 sys=0.05, real=0.45 secs] 
2.637: [GC (CMS Final Remark) [YG occupancy: 54140 K (78656 K)]2.637: [Rescan (parallel) , 0.0284352 secs]2.666: [weak refs processing, 0.0001802 secs]2.666: [class unloading, 0.0051609 secs]2.671: [scrub symbol table, 0.0035550 secs]2.675: [scrub string table, 0.0008166 secs][1 CMS-remark: 29844K(174784K)] 83984K(253440K), 0.0391194 secs] [Times: user=0.21 sys=0.00, real=0.04 secs] 
2.676: [CMS-concurrent-sweep-start]
2.688: [CMS-concurrent-sweep: 0.011/0.012 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
2.688: [CMS-concurrent-reset-start]
2.696: [CMS-concurrent-reset: 0.008/0.008 secs] [Times: user=0.04 sys=0.01, real=0.01 secs] 
3.044: [GC (Allocation Failure) 3.044: [ParNew: 78656K->8704K(78656K), 0.0251656 secs] 97836K->40288K(253440K), 0.0252453 secs] [Times: user=0.14 sys=0.00, real=0.03 secs] 
3.677: [GC (Allocation Failure) 3.677: [ParNew: 78656K->8704K(78656K), 0.0159650 secs] 110240K->50554K(253440K), 0.0160374 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
4.851: [GC (Allocation Failure) 4.851: [ParNew: 78656K->8704K(78656K), 0.0172068 secs] 120506K->61047K(253440K), 0.0172778 secs] [Times: user=0.10 sys=0.00, real=0.02 secs] 
6.191: [GC (Allocation Failure) 6.192: [ParNew: 78656K->8704K(78656K), 0.0271375 secs] 130999K->77488K(253440K), 0.0272281 secs] [Times: user=0.15 sys=0.00, real=0.03 secs] 
6.219: [GC (CMS Initial Mark) [1 CMS-initial-mark: 68784K(174784K)] 78713K(253440K), 0.0030824 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
6.222: [CMS-concurrent-mark-start]
6.288: [CMS-concurrent-mark: 0.057/0.066 secs] [Times: user=0.39 sys=0.01, real=0.07 secs] 
6.288: [CMS-concurrent-preclean-start]
6.291: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
6.291: [CMS-concurrent-abortable-preclean-start]
7.113: [CMS-concurrent-abortable-preclean: 0.775/0.822 secs] [Times: user=3.79 sys=0.09, real=0.83 secs] 
7.113: [GC (CMS Final Remark) [YG occupancy: 45871 K (78656 K)]7.113: [Rescan (parallel) , 0.0072989 secs]7.121: [weak refs processing, 0.0005665 secs]7.121: [class unloading, 0.0092666 secs]7.131: [scrub symbol table, 0.0150502 secs]7.146: [scrub string table, 0.0012746 secs][1 CMS-remark: 68784K(174784K)] 114655K(253440K), 0.0346254 secs] [Times: user=0.12 sys=0.01, real=0.03 secs] 
7.148: [CMS-concurrent-sweep-start]
7.185: [CMS-concurrent-sweep: 0.035/0.037 secs] [Times: user=0.24 sys=0.00, real=0.04 secs] 
7.185: [CMS-concurrent-reset-start]
7.193: [CMS-concurrent-reset: 0.008/0.008 secs] [Times: user=0.05 sys=0.00, real=0.01 secs] 
8.242: [GC (Allocation Failure) 8.242: [ParNew: 78656K->8704K(78656K), 0.0260379 secs] 136080K->77426K(253440K), 0.0261191 secs] [Times: user=0.17 sys=0.00, real=0.03 secs] 
8.893: [GC (Allocation Failure) 8.893: [ParNew: 78656K->8703K(78656K), 0.0166601 secs] 147378K->85555K(253440K), 0.0167357 secs] [Times: user=0.10 sys=0.00, real=0.01 secs] 
9.279: [GC (Allocation Failure) 9.280: [ParNew: 78655K->5650K(78656K), 0.0039284 secs] 155507K->82501K(253440K), 0.0040061 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
9.905: [GC (Allocation Failure) 9.906: [ParNew: 75602K->8704K(78656K), 0.0143583 secs] 152453K->89562K(253440K), 0.0144558 secs] [Times: user=0.08 sys=0.00, real=0.01 secs] 
10.602: [GC (Allocation Failure) 10.602: [ParNew: 78656K->8704K(78656K), 0.0364558 secs] 159514K->104422K(253440K), 0.0365486 secs] [Times: user=0.14 sys=0.00, real=0.04 secs] 
11.523: [GC (Allocation Failure) 11.523: [ParNew: 78656K->8704K(78656K), 0.0380868 secs] 174374K->123989K(253440K), 0.0381740 secs] [Times: user=0.23 sys=0.00, real=0.04 secs] 
12.274: [GC (Allocation Failure) 12.274: [ParNew: 78656K->8704K(78656K), 0.0313268 secs] 193941K->136954K(253440K), 0.0314313 secs] [Times: user=0.17 sys=0.00, real=0.03 secs] 
```

`jstat` man-pages say that FGC stands for the number of Full GC events, so let's take another look at the GC logs, this time limiting them to single CMS cycle:

```sh
2.170: [GC (CMS Initial Mark) [1 CMS-initial-mark: 29844K(174784K)] 39075K(253440K), 0.0021170 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
2.172: [CMS-concurrent-mark-start]
2.185: [CMS-concurrent-mark: 0.013/0.013 secs] [Times: user=0.08 sys=0.00, real=0.02 secs] 
2.185: [CMS-concurrent-preclean-start]
2.186: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.186: [CMS-concurrent-abortable-preclean-start]
2.637: [CMS-concurrent-abortable-preclean: 0.169/0.451 secs] [Times: user=2.13 sys=0.05, real=0.45 secs] 
2.637: [GC (CMS Final Remark) [YG occupancy: 54140 K (78656 K)]2.637: [Rescan (parallel) , 0.0284352 secs]2.666: [weak refs processing, 0.0001802 secs]2.666: [class unloading, 0.0051609 secs]2.671: [scrub symbol table, 0.0035550 secs]2.675: [scrub string table, 0.0008166 secs][1 CMS-remark: 29844K(174784K)] 83984K(253440K), 0.0391194 secs] [Times: user=0.21 sys=0.00, real=0.04 secs] 
2.676: [CMS-concurrent-sweep-start]
2.688: [CMS-concurrent-sweep: 0.011/0.012 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
2.688: [CMS-concurrent-reset-start]
2.696: [CMS-concurrent-reset: 0.008/0.008 secs] [Times: user=0.04 sys=0.01, real=0.01 secs] 
```

You can clearly see that CMS performs its work in several phases but just 2 of them (Initial Mark and Final Remark) count as GC events, only those that are stop-the-world phases.

Should you measure something be sure what you really measure otherwise the results can keep you far from reality.
Always validate your assumptions and [RTFM](https://xkcd.com/293/)!

[^1]: Tom DeMarco
[^2]: [Linux is borrowing unused memory for disk caching what makes it looks like you are low on memory while you are not](http://www.linuxatemyram.com/)
[^3]: assuming there were no failures
