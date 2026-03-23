---
title: "Understanding ARP Spoofing and DNS Poisoning Through a Controlled Lab"
date: 2026-03-23
draft: false
summary: "A technical walkthrough of a controlled lab project exploring ARP spoofing and DNS poisoning, how the attack flow works conceptually, what was observed in practice, and where the implementation breaks down."
tags: ["network-security", "cybersecurity", "protocols", "lab", "scapy"]
---

> This article summarizes a university lab project that explored ARP spoofing and DNS poisoning in an isolated virtual environment for educational analysis. It focuses on protocol behavior, experiment design, observations, and defensive takeaways rather than operational misuse.

## Motivation

One of the recurring patterns in network security is that many foundational protocols were designed in a much more trusting era of computing. Two classic examples are ARP and DNS. Neither protocol was originally built with strong authenticity guarantees at the point where everyday local-network decisions get made, and that gap creates room for traffic manipulation.

In this lab project, we studied that trust model by building an automated tool in a controlled virtual environment and using it to observe two well-known attack classes: ARP spoofing and DNS poisoning. The goal was not just to demonstrate that packets can be rerouted, but to understand where the weak assumptions live, how automation changes the operator workflow, and what practical limitations appear once theory meets implementation.

## What this project explored

The project centered around a Python-based tool using Scapy and supporting networking libraries in a Linux environment. Its purpose was to automate host discovery, user-driven target selection, ARP spoofing, and DNS response manipulation in a small VM-based setup. :contentReference[oaicite:3]{index=3}

At a high level, the workflow looked like this:

1. enumerate available interfaces
2. discover active hosts on the selected network
3. let the operator choose a victim and a spoof target
4. redirect local traffic by poisoning address mappings
5. in the DNS scenario, intercept requests and return attacker-controlled answers

That structure is one of the more interesting parts of the paper: it turns two protocol abuse patterns into a single operator flow, which makes the system easier to reason about from an engineering perspective.

## Why ARP is vulnerable in practice

ARP exists to answer a simple question on a LAN: _which MAC address corresponds to this IP address?_ Devices cache those answers locally because they need fast, repeated lookups during normal communication.

The weakness is that ARP trusts replies too easily. If a device accepts a forged association between an IP address and a malicious MAC address, packets can start flowing through the wrong machine. In practical terms, the attacker inserts themselves into the communication path and becomes a man-in-the-middle. The report describes exactly this trust failure as the core of the ARP spoofing stage. :contentReference[oaicite:4]{index=4}

That makes ARP spoofing conceptually powerful even though the underlying mechanism is simple: the attack does not need to break encryption at this stage. It only needs to convince systems to send frames to the wrong place.

## Why DNS poisoning matters

DNS plays a different role. Instead of mapping IP addresses to hardware addresses, it maps human-friendly domain names to IP addresses. The paper outlines the relationship between recursive DNS servers, authoritative DNS servers, and cached responses, then explains how a forged answer can redirect users to an unintended destination. :contentReference[oaicite:5]{index=5}

That distinction matters because DNS poisoning changes _where a user thinks they are going_, while ARP spoofing changes _where traffic is actually flowing on the local network_. Combined, they form a useful teaching pair:

- ARP spoofing compromises local network trust
- DNS poisoning compromises name resolution trust

Together, they demonstrate how layered assumptions can fail in different ways.

## Lab environment

The implementation was tested in a virtual setup with three machines:

- an Ubuntu attacker VM
- a Windows XP victim VM
- an Oracle server VM

The tool was developed for Linux using Python 2.7 together with Scapy, netifaces, netaddr, and time. The report also notes that the victim and server machines needed manual networking adjustments in VirtualBox for the experiments to work correctly. :contentReference[oaicite:6]{index=6}

That kind of isolated setup is exactly the right environment for studying this topic. It reduces risk, gives repeatable network conditions, and makes state transitions easier to observe.

## From theory to operator workflow

One of the better design choices in the project is that the tool begins with a discovery phase. It does not hardcode a single target. Instead, it walks the user through:

- interface selection
- active host enumeration
- target choice
- spoof target choice

The report describes this discovery-first design explicitly and separates it from the later spoofing phases. :contentReference[oaicite:7]{index=7}

That separation is worth calling out because it improves both usability and clarity. In security tooling, the distinction between _finding the environment_ and _acting on the environment_ is often where most of the engineering effort goes.

## ARP spoofing phase

In the ARP scenario, three machines participate: attacker, client, and server. The victim begins with an empty or untrusted ARP cache state, then the operator selects the target and spoofed host from the discovered list. Once the forged mappings are accepted, the victim starts sending packets to the attacker instead of the intended destination. The paper describes this handoff and validates it by inspecting the victim’s ARP cache. :contentReference[oaicite:8]{index=8}

A useful implementation detail is that the project also attempted restoration after a timeout window by sending corrective information to repair the mapping state. That matters because it shows awareness of lifecycle management, not just attack execution. The report mentions a restore step after 512 seconds to re-establish correct MAC associations. :contentReference[oaicite:9]{index=9}

### Observation

This part of the project demonstrates a recurring security truth: once a protocol assumes honesty by default, the attacker’s work often becomes less about sophistication and more about timing and placement.

## DNS poisoning phase

The DNS stage follows a similar pattern but changes the point of interception. After selecting the relevant interface and victim, the attacker forges gateway-related trust and waits for DNS requests. When a request is observed, the tool returns an attacker-chosen destination, redirecting traffic intended for a real domain to a lab-controlled IP. The report states that, in this implementation, DNS requests were redirected to `10.0.2.6`, and the lab demonstration used `google.com` as the visible example. :contentReference[oaicite:10]{index=10}

This is a good example of why DNS manipulation is so educational in a classroom setting: the result is visible immediately. A user types one address, but the browser lands somewhere else.

### Observation

The interesting lesson here is not just that the redirection worked. It is that the success depends on far more than crafting one reply. The timing of interception, the handling of non-target traffic, and the relationship between local routing and DNS response processing all affect whether the browser experience looks convincing.

## Engineering the tool

The engineering section of the report is where the project becomes most interesting as a software artifact. The implementation was split into three major parts:

- automation logic
- ARP spoofing logic
- DNS spoofing logic

The tool first gathered local interfaces and their properties, then computed network scope, then used host discovery to populate a list of reachable systems. The report specifically describes a flow involving interface enumeration, network/IP derivation, and host discovery through Scapy-assisted probing. :contentReference[oaicite:11]{index=11}

That design choice is solid because it reduces manual operator error and makes the program reusable across slightly different lab topologies.

## Safe pseudocode view of the architecture

Below is a **non-operational** representation of the tool’s structure, included only to explain the software design:

```python
def main():
    mode = choose_attack_mode()
    iface = choose_interface()
    hosts = discover_hosts(iface)

    if mode == "arp":
        victim = choose_victim(hosts)
        peer = choose_peer(hosts)
        run_arp_experiment(victim, peer, iface)
    elif mode == "dns":
        victim = choose_victim(hosts)
        run_dns_experiment(victim, iface)
