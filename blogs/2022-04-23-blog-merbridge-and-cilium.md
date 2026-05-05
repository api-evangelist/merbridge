---
title: "Blog: Merbridge and Cilium"
url: "/blog/2022/04/23/merbridge-and-cilium/"
date: "Sat, 23 Apr 2022 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
<h1 id="merbridge-and-cilium">Merbridge and Cilium</h1>
<p><a href="https://cilium.io/">Cilium</a> is a great open source software that provides a lot of networking capabilities for cloud native applications based on eBPF, with a lot of great designs. Among others, Cilium designed a set of sockmap-based redir capabilities to help accelerate network communications, which inspired us and is the basis for Merbridge to provide network acceleration. It is a really great design.</p>
<p>Merbridge leverages the great foundation that Cilium has provided, along with some targeted adaptations we&rsquo;ve made in the Service Mesh, to make it easier to apply eBPF technology to Service Mesh.</p>
<p>Our development team have learned a lot eBPF theoretical knowledge, practical methods, and testing methods, from Cilium&rsquo;s detailed documentation and our frequent exchanges with the Cilium technical team. All these together helps make Merbridge possible.</p>
<p>Thanks again to the Cilium project and community, and to Cilium for these great designs.</p>
