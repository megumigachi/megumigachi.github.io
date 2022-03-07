---
layout: post
title: "Why is Paxos Hard?"
categories:
  - CS
tags:
  - consensus
  - system
toc: true
toc_sticky: true
toc_label: Contents
---

[Chinese Version](https://xxchan.github.io/cs/2022/02/09/paxos-hard-zh.html)


In this blog, I will **omit practical aspects like performance issues**. I will focus on general ideas like basic notions, expressiveness, or "**how to make it possible**" instead of "how to make it super-fast". Also, I will try to convey my ideas to readers without any knowledge of distributed systems or consensus. Still, it may be better if you already know one or more consensus algorithms, like Raft or Paxos.

## My Way of Learning Consensus

[This zhihu answer: Is there a moment when you feel the malice of the author of a paper?](https://www.zhihu.com/question/65038346/answer/227674428) (Chinese) was the first time I knew Paxos and consensus. I thought it was really fun.

Exactly one year ago, I was learning MIT 6.824 and implementing Raft. I thought it organizes things into modules and explains each module clearly. I also giggled when reading the "What's wrong with Paxos?" part. Although I experienced some difficulty debugging Raft, the cognitive process was smooth.

Last semester, I needed to implement Paxos in one of my homework. I happened to have bookmarked [this blog](https://blog.openacid.com/algo/paxos/) earlier. I found it explains Paxos in a simple way and I implemented (basic) Paxos correspondingly, without reading papers or other explanation articles. The homework also required us to implement multiple decisions using a given technique, so I didn't read things like multi-Paxos.

I also had Concurrent Algorithms and Distributed Algorithms courses. They are theoretical, and both talked about consensus, and I thought it was not very hard to understand. The consensus algorithm given in the DA course is very like Paxos, and it's simple enough to fit in one slide. Besides, the Universal Construction in the CA course and the Total Order Broadcast algorithm in the DA course, which prove you can use consensus to do almost anything, are also not hard to understand. And then, I began to wonder why people think Paxos is hard to understand, and that's why I wanted to write this blog.

Finally, I read the two Paxos papers and confirmed that they are both understandable. But Lamport's language is so interesting. *The Part-Time Parliament* is very detailed and straightforward. It kind of looks like it uses a mapping to transform the terms. *Paxos Made Simple* instead talks about intuitions in plain words. It is kind of like a blog.

Anyway, let's go to the main topic.

## Abstractions: Consensus, Replicated Log, and Replicated State Machine

First, let's put aside the consensus algorithms but focus on the abstraction (or the interface).

In distributed systems, we generally want to *share* something. Then shared objects or shared data structures are super helpful. It is like a usual data structure (e.g., queue, map) in a single machine but can be accessed by different computers (or threads). In other words, the computers work together to maintain a global object and have its state replicated on every machine.

Then replicated state machine is the ultimate abstraction, a generalization of all shared data structures. (Or we can say all shared data structures can be implemented using replicated state machine). You may quickly come up with that maintaining a replicated log is a natural implementation of the replicated state machine. And that's true.

By the way, the replicated log can also be viewed as another abstraction, Total-Order Broadcast: Every process broadcasts messages, but they deliver messages in the same global order. (Very similar, not very important.)

Then what's consensus? Here it is, the most original consensus interface. First, it's just one shared object. Its *sequential specification* (How it works if it is accessed by different processes *sequentially*) is:

```
maintaining a variable: prop
initial value: ⊥ (uninitialized)

propose(v):
    if prop == ⊥:
        prop = v
    return prop
```

Fairly simple, right? Some notes here:
- It is one-shot. A single decision. It can be viewed as a write-once variable.
- Here synchronous style interface is shown. Using asynchronous event-based interfaces is also totally fine.
- It is usually assumed that every process has something to propose, and proposing is the only way to get the decided value. I think this convention is just for simplicity. You can easily modify the interface to get notified about the decision without proposing.

Equivalently, we can also describe consensus by giving its properties:
- Validity: A decided value is proposed.
- Agreement: No processes decide differently.
- Termination: Every process eventually decides.

## Implement Replicated State Machine Using Consensus

Consensus is such a simple abstraction, but it is powerful enough to build replicated state machine, i.e., it can be used to implement any shared object. There's a loud name for this conclusion: **Universality of Consensus**.

Before we go further, we can think about a naïve solution: Consensus is already a single replicated log entry, so why not use a list of consensus as the whole replicated log? When a process have a new entry to append to the log, it just keeps moving forward and proposing the entry in the consensus list until its proposal is decided in one consensus instance.

This idea almost works, except for one critical problem: a process may never get its proposal decided. It can starve. The solution is also not very complicated. Now processes not only propose their own value but also **help** others. Specifically, when a new entry comes, the process will first broadcast it, and it keeps collecting others' entries. Then each of its proposals will be **the set of all pending entries** (instead of one entry). That's it!

Of course, this is not the only way and is a rather unoptimized way, but it works perfectly.

Without a concrete algorithm in mind, you may already have thought that it was feasible to go from the consensus to the replicated log. I just showed that there's a simple way to do so to make us confident in the possibility. Going from the consensus to the replicated log is like shifting from manual gear to automatic gear.

## Why is Paxos hard?

If you know how Paxos work, you may already find out that its core indeed implements consensus. Many complexities come from turning it into the replicated log. It seems that this relationship is not clearly stated in the paper (although I didn't read this part carefully). But having the replicated log algorithm and other meta-knowledge about consensus above in mind and omitting implementation details and performance issues (although this might be hard in an engineer's mindset), you can convince yourself it works, in a simple way. All you need is (single-decision) consensus! In other words, if you already know the consensus abstraction and are looking for an implementation, you will quickly get what Paxos is doing. So it's not the algorithm itself that is complex, but there are many concepts mixed together.

For those who don't know Paxos yet, you can go back to read the paper and focus on the single decision part first without being bothered by multiple decisions now! But I will still give some simple explanations here.

[The previously mentioned blog by xp](https://blog.openacid.com/algo/paxos/) (Chinese) is clear and understandable because it also focuses on simple things without going too far to confuse the reader. But I think the ideas in this blog can be further summarized:

First, Single decision consensus interface is introduced (slide 1-5). And then many failed attempts (but still important techniques) are discussed:
- slide 6-9: Single writer *all-ack*. It doesn't tolerate any crashes.
- slide 10-12: Single writer majority-ack (or *majority-write*). *Timestamp* + *majority-read* are also needed for it to work. This algorithm doesn't tolerate the writer's crash.
- slide 14-17: Majority-write can neither be generalized to **multiple writers**.
- slide 18-19: (majority) Read before (majority) write.
- slide 20: Request (majority) promise before (majority) write.

Yes, I think Paxos's idea can be summarized into one sentence: **request promise before write**! It is indeed so simple and can be implemented with fewer than 200 LoC.

xp tells you *how* Paxos works in a simple way, but you may still wonder *why* we go this way and whether this really works. Now I should say in his paper, Lamport **derives** this idea (from *properties* he wants to achieve). So if you re-read the paper after you already know Paxos, you may find it rather understandable and has brilliant reasoning and intuitions. But I'm afraid that maybe this is also why many people failed to understand it at the beginning, because they are not accustomed to this way of thinking and want to know how exactly it works before knowing where the idea can be derived.

Btw, xp's follow-up blog: [Implement paxos-based kv storage with 200 line of code](https://drmingdrmer.github.io/algo/2020/10/28/paxoskv.html) implements (single-decision) Paxos, and manages consensus instances by hand. (There's a more complicated follow-up blog [here](https://blog.openacid.com/algo/mmp3/)) So you can also learn (single-decision) Paxos by reading the implementation.

## Is Raft Easier?

Let's go back and look at Raft again. First, the "What's wrong with Paxos?" part.

> We hypothesize that Paxos' opaqueness derives from its choice of the single-decree subset as its foundation. Single-decree Paxos is dense and subtle: it is divided into two stages that do not have simple intuitive explanations and cannot be understood independently.

I should now say this evaluation is a little bit **unfair**. Single-decision consensus is a well-established abstraction in distributed computing theory. Single-decree Paxos also has good intuitions.

---

Another evaluation that Paxos does not provide a good foundation for building *practical* implementations in *real-world* systems may be true. However, I think this is just a **non-goal** for Paxos. It talks about an elegant way to solve a theoretical problem. It just **doesn't care** much about how you implement it in the real-world... I think Lamport is also a more theoretical researcher. (He established the foundation of the distributed computing theory.) Real-world stuff may not be that attractive to theory researchers, if the problem is not theoretically interesting.

---

Now let's look at Raft itself. I will also try to describe it in a few words.

First, it talks about the replicated state machine and **tells you explicitly we are building the replicated log**. Engineers may be unfamiliar with consensus, but must know logs very well.

Second, Raft chooses the **leader-based** approach: only the leader can interact with the outside world and append entries to the log. Paxos is instead completely **peer-to-peer**. Having a leader will usually make the main process or the "happy path" easier to understand. Raft chooses majority-write naturally. Then the only problem is how to handle the leader's crash, and Raft's approach is to restrict nodes from voting for *outdated* nodes. The trick is simple, but it also takes some effort to understand why it works because many edge cases are covered. The property to maintain is that the new leader must *at least* contain all majority-written entries, but whether it contains more pending entries is insignificant. 

Finally, inconsistent states exist in the replicate log, and the leader will fix it by overwriting.

So is Raft easy? Its core idea is also not hard. It's just single-leader majority-write, … with some additional but simple tricks (to ensure majority-write work).

To summarize, Raft
- Implements replicated log directly.
- Talks **concrete notions** like heartbeat, timeout, RPC and crash recovery, instead of talking about abstractions like a theory researcher.
- Talks **all details in all paths** (state transformations), and **exposes complexities** in a manageable way. (Paxos doesn't care these *trivial* implementation complexities.)

So after reading Raft, you can almost directly **translate it into code**. Instead, Paxos is theoretically simple. After reading it, you may think for a while and then exclaim: **Oh, it works! It's so simple!** (Just like a Mathematician.) Then no wonder most people, especially engineers, will feel more comfortable when reading Raft.

## More about Consensus

There are more interesting theoretical problems about consensus. Besides universality, the most important one may be the **impossibility** of consensus! To be specific, here's the FLP impossibility: Consensus is impossible in an asynchronous distributed system if one process can crash.

Hey, then what are Paxos and Raft doing???

*Don't panic.* Here **asynchronous** system means: each process can be delayed for *arbitrarily long*, and we *know nothing* about whether they are alive. But in practice, we do have some assumptions (about **time**). Real-world is more like an **eventually synchronous** system, which means we can have an *eventually perfect failure detector*: It can wrongly suspect a slow process as dead, but it will make corrections if time lasts long enough. Basically, the heartbeat and timeout mechanism implements an eventually perfect failure detector.

Another side note is that actually, we assume communication by message-passing above. If we switch to shared memory, we will still have FLP impossibility. To be precise, it is impossible to implement consensus using only **shared registers**.

I should mention that the proof of FLP impossibility is also very brilliant! The core idea is: any consensus algorithm's execution must have a **critical timing**. From God's perspective, currently, the system doesn't reach a decision but will immediately (after a single step of any process) reach a decision. All later steps are just going through a process to get everybody notified. (If you know *Steins;Gate*, it means the world line will convergent into an attractor field.) Then we argue that all the processes' next steps are accessing the same register at the critical timing!

To continue the discussion, let's assume we have two processes `p1` and `p2` . If they are not accessing the same object, the following two possible histories are equivalent:
1. `p1` goes one step and then `p2` goes one step.
2. `p2` goes one step and then `p1` goes one step.

This contradicts the critical timing assumption!

Now assume they are both about to access register `x` .
- Case 1: If the first step is reading `x` , we stop the process forever after this step. Then the other process will think everything is the same as before, so it cannot decide! This means the critical step must **make some observable change** to the whole system. So
- Case 2: Both processes' first step is writing `x` . Then the following two possible histories are equivalent:
  
  1. `p1` writes `v1` to `x`, and stops forever. Then `p2` writes `v2` to `x`.
  2. `p1` stops forever immediately. `p2` writes `v2` to `x`. 
  
  Again, this contradicts the critical timing assumption!


The proof is now finished. (I just showed the world line can never convergent.) Not complicated, right? And now you may also have some idea why the impossibility holds. Asynchrony is a too weak assumption. In other words, it gives too strong power to the *scheduler* (or us the prover).

More excitingly, the above proof framework applies to more situations. Actually, we proved above consensus *among 2 processes* is impossible using only registers. We can similarly prove consensus *among 3 processes* is impossible using only registers and queue/fetch-and-add! But we can implement consensus among any number of processes using compare-and-swap!

These results proposed a way to **compare the expressiveness of shared objects**: consensus number. Register has consensus number 1, queue and FAA have 2, and CAS has ∞. There are still unresolved research problems about consensus numbers, but I think we'd better stop here now.

As a concluding note, I think distributed computing theory is interesting and worth learning a bit. 😄
