---
layout: post
title: "E-voting, blockchain and zero-knowledge proof (1): Why are we still using paper to vote? What is an ideal e-voting system?"
description: "Exploring why paper-based voting still prevails, the security and privacy challenges of electronic voting systems, and how blockchain and zero-knowledge proof technologies can create an ideal e-voting solution."
date: 2025-05-05 00:00:00 +0800
categories: technology
tags: e-voting blockchain zero-knowledge
image: /assets/images/2025-05-05-evoting-blockchain-zk-introduction/cover.png
---

![E-voting, blockchain and zero-knowledge proof (1)](/assets/images/2025-05-05-evoting-blockchain-zk-introduction/cover.png)

## Background

In the year 2025, many of us are still using paper ballots and stamps to vote, still waiting for days for the tallying results, and even weeks for a final outcome if disputes trigger a recount. One would have thought technology would have solved this problem by now. However, e-voting remains a controversial topic and faces significant resistance. The reasons for opposition aren't merely rooted in tradition or politics; many concerns relate to legitimate issues of security, integrity and privacy. Are we destined to count paper ballots for important decisions indefinitely?

A few years ago, I worked on a [project](https://github.com/DASC7600-2022-E-Voting/E-Voting-App) implementing a secure e-voting system on blockchain. It was surprising to me that even in the web3 space—where trustless systems, public verifiability, zero knowledge and governance are all highly valued—e-voting remains a niche topic that has received limited attention and resources. In the end, I wasn't able to complete a fully functional MVP. Here I would like to share some concepts and ideas I've learned, hoping they might benefit others interested in this field.

In this series of articles, we will explore the reasons why e-voting is not widely adopted, and how blockchain and zero-knowledge proof technologies can help address some of these challenges.

## Why (not) e-voting?

E-voting systems come in many forms, but they can primarily be divided into two categories: online voting and electronic voting machines (EVMs). Online voting enables citizens to cast their ballots via the internet, while EVMs are physical devices used for on-site electronic voting.

In the following sections, we'll discuss the advantages and disadvantages of e-voting. Some points apply to both categories, while others are specific to one form.

### Advantages of e-voting

- **Speed & Efficiency**: E-voting can provide faster results, as the counting process is automated and can even be done in real-time.

- **Reduce human error**: Counting errors can be completely eliminated with e-voting, and recounting is never needed. Depending on the system design, errors in voter eligibility verification and even double voting can also be prevented.

- **Accessibility**: E-voting machines and online voting platforms can be designed with accessibility in mind, featuring audio transcription capabilities and adjustable fonts. In traditional voting, people with disabilities often must rely on poll workers for assistance.

- **Cost-effective**: E-voting reduces costs associated with printing, mailing, and labor for counting and verifying votes.

- **Convenience**: Remote voting allows citizens to vote from anywhere, which is especially beneficial for people overseas or with mobility challenges. Even for on-site voting, e-voting machines can reduce waiting times.

### Disadvantages of e-voting

- **Security**: E-voting systems must operate on servers connected to the internet, making them vulnerable to hacking and cyber attacks. For example, a hacker could alter vote counts or manipulate election results.

- **Integrity**: E-voting websites and hardware can be susceptible to supply chain attacks and tampering. For instance, a hacker could modify the user interface of a voting machine to mislead voters into selecting the wrong candidate or even change the vote count after the election.

- **Privacy**: Ensuring votes are cast in private and that voter identities remain confidential is crucial. However, in the context of e-voting, it is challenging to guarantee that votes leave no digital fingerprints and are truly anonymous, as digital authentication methods are required to verify voter identities. Data breaches could expose voter identities and even the content of their votes.

- **Technical issues**: Software bugs and hardware glitches can lead to errors in the voting process, such as miscounted votes or system crashes. Additionally, internet connectivity problems or hardware failures could prevent voters from casting their ballots.

- **Transparency**: While paper ballots can be physically counted, audited, and observed by third parties, e-voting systems are often closed-source and proprietary, making it difficult to verify the integrity of such black-box systems. Even open-source or audited systems face challenges in ensuring that the deployed system remains un-tampered and un-compromised.

## What would an ideal e-voting system look like?

An ideal e-voting system should preserve the advantages of e-voting while mitigating most of the disadvantages. However, it is often impossible to eliminate all drawbacks, and design trade-offs must be made based on specific use cases.

To design a practical e-voting system that would gain widespread acceptance, let's take a step back and analyze the characteristics of an ideal voting system.

- **Public verifiability**:
  The system should allow third parties to verify the following information:

  - Number of eligible voters
  - The start, end, and duration of the voting period
  - The number of votes casted
  - The number of votes deemed invalid, if any
  - The tallying results

  This ensures that the entire voting process and results can be audited and verified by any third party, not just by a trusted authority.

- **Individual verifiability**:
  The system should allow voters to verify the following information. Whether this information can also be verified by third parties depends on design and implementation constraints:

  - If one is eligible to vote
  - If one's vote is casted
  - If one's vote is counted

  This ensures that voters can confirm their votes are included in the tally.

- **Anonymity**:
  The system should ensure the following information cannot be proven or obtained by anyone, including the voting authority:

  - One cannot know the option for a just-cast vote until tallying
  - Any voter's vote cannot be linked to their identity
  - No voter can prove they voted for a specific candidate

  These concepts are often referred to as [ballot secrecy](https://en.wikipedia.org/wiki/Secret_ballot) and receipt freeness. This ensures voters cannot be coerced or bribed to vote for a specific candidate (vote buying) and that their votes remain truly private.

- **High availability**:
  The system should possess the following properties to ensure voters can cast their ballots during the voting period:

  - Capacity to handle the expected number of voters
  - Resistance to DDOS attacks
  - No single point of failure
  - Disaster recovery plan

## How can blockchain and zero-knowledge proof help?

Looking at the above requirements, designing a system that satisfies all of them is highly challenging. How can one make something publicly verifiable while ensuring parts of it remain secret? How can secrecy be maintained in such a system if the secret holder might be incentivized to reveal their voting choice? Many existing systems—e-voting or otherwise—rely on trusting some form of authority to resolve these conflicts and act as the primary entity ensuring the system's privacy and integrity. This reliance often becomes the source of fraud and disputes, as such authorities can be bribed, compromised, or even have their own incentives to commit fraud.

This is where blockchain and [zero-knowledge proof](https://en.wikipedia.org/wiki/Zero-knowledge_proof) come into play. Blockchain is designed as a publicly verifiable ledger that eliminates the need for trusted authorities. It is inherently highly available and tamper-proof. Zero-knowledge techniques, often used on top of blockchain, provide public verifiability of private information. These two technologies can be combined to create a system that is both publicly verifiable and private, while also ensuring the system's integrity and availability.

In the next article, we will discuss some technical and mathematical concepts related to these technologies and how they can be used to design a practical next-generation e-voting system.
