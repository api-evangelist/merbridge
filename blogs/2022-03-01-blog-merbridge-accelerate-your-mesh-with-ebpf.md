---
title: "Blog: Merbridge - Accelerate your mesh with eBPF"
url: "/blog/2022/03/01/merbridge-introduce/"
date: "Tue, 01 Mar 2022 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
Merbridge - Accelerate your mesh with eBPF Replacing iptables rules with eBPF allows transporting data directly from inbound sockets to outbound sockets, shortening the datapath between sidecars and services. Introduction The secret of Istio’s abilities in traffic management, security, observability and policy is all in the Envoy proxy. Istio uses Envoy as the “sidecar” to intercept service traffic, with the kernel’s netfilter packet filter functionality configured by iptables.
