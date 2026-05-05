---
title: "Blog: Merbridge CNI Mode"
url: "/blog/2022/05/18/cni-mode/"
date: "Wed, 18 May 2022 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
<!--Merbridge CNI 模式的出现，旨在能够更好地适配服务网格的功能。之前没有 CNI 模式时，Merbridge 能够做得事情比较有限。其中最大的问题是不能适配注入 Istio 的 Sidecar Annotation，这就导致 Merbridge 无法排除某些端口或 IP 段的流量等。同时，由于之前 Merbridge 只处理 Pod 内部的连接请求，这就导致，如果是外部发送到 Pod 的流量，Merbridge 将无法处理。-->
<p>The CNI mode is designed to better adapt to the service mesh functions. Before having this mode, Merbridge was limited to certain scenarios. The biggest problem was that it could not adapt to the sidecar annotations from the container injected by Istio, which led to Merbridge cannot exclude traffic from certain ports and IP ranges. Furthermore, Merbridge was only able to handle requests inside the pod, which means the external traffic sent to the pod was not handled.</p>
<!--为此，我们精心设计了 Merbridge CNI，旨在解决这些问题。-->
<p>Therefore, we have implemented the Merbridge CNI to address these issues.</p>
<!--## 为什么需要 CNI 模式？-->
<h2 id="why-cni-mode-is-needed">Why CNI mode is needed</h2>
<!--其一，之前的 Merbridge 只有一个很小的控制面，其监听 Pod 资源，将当前节点的 IP 信息写入 `local_pod_ips` 的 map，以供 connect 使用。但是，connect 程序由于工作在主机内核层，其无法知道当前正在处理的是哪个 Pod 的流量，就没法处理如 `excludeOutboundPorts` 等配置。为了能够适配注入 `excludeOutboundPorts` 的 Sidecar Annotation，我们需要让 eBPF 程序能够得知当前正在处理哪个 Pod 的请求。-->
<p>First, Merbridge had a small control plane before, which listened to pods resources, and wrote the current node IP into the map of <code>local_pod_ips</code> for use by <code>connect</code>. However, since the <code>connect</code> program only works at the host kernel layer, it won&rsquo;t know which pod&rsquo;s traffic is being processed. Thus, configurations like <code>excludeOutboundPorts</code> cannot be handled. In order to be able to adapt to the injected sidecar annotation <code>excludeOutboundPorts</code>, we need to let the eBPF program know which Pod&rsquo;s request is currently being processed.</p>
<!--为此，我们设计了一套方法，与 CNI 配合，能够获取当前 Pod 的 IP，以适配针对 Pod 的特殊配置。-->
<p>To this end, we have designed a method to cooperate with the CNI, through which you can get the current Pod IP to validate special configurations for the Pod.</p>
<!--其二，在之前的 Merbridge 版本中，只有 connect 会处理主机发起的请求，这在同一台主机上的 Pod 互相通讯时，是没有问题的。但是在不同主机之间通讯时就会出现问题，因为按照之前的逻辑，在跨节点通讯时流量不会被修改，这会导致在接收端还是离不开 iptables。-->
<p>Second, for early versions of Merbridge, only <code>connect</code> would process requests from the host, which had no problem for intra-node pod communication. However, it becomes problematic when traffic flows between different nodes. According to the previous logic, the traffic will not be modified during the cross-node communication, which will lead to the use of <code>iptables</code> at the end.</p>
<!--这次，我们依靠 XDP 程序，解决入口流量处理的问题。因为 XDP 程序需要挂载网卡，所以也需要借助 CNI。-->
<p>Here, we turned to the XDP program for processing the inbound traffic. The XDP program needs to mount a network card, which also needs to use CNI.</p>
<!--## CNI 如何解决问题？-->
<h2 id="how-does-cni-work">How does CNI work</h2>
<!--这里我们将探讨 CNI 的工作原理，以及如何使用 CNI 来解决问题。-->
<p>This section will explore how CNI works and how to use CNI to solve the issues mentioned above.</p>
<!--### 如何通过 CNI 让 eBPF 程序获取当前正在处理的 Pod IP？-->
<h3 id="how-to-use-cni-to-let-ebpf-have-the-current-pod-ip">How to use CNI to let eBPF have the current Pod IP</h3>
<!--我们通过 CNI，在 Pod 创建的时候，将 Pod 的 IP 信息写入一个 Map（`mark_pod_ips_map`），其 Key 为一个随机的值，Value 为 Pod 的 IP。然后，在当前 Pod 的 NetNS 里面监听一个特殊的端口 39807，将 Key 使用 `setsockopt` 写入这个端口 socket 的 mark。-->
<p>When a pod is created, we write Pod IP into the map <code>mark_pod_ips_map</code> through CNI, where the key is a random value, and the value is the Pod IP. Then, we listen to a special port <code>39807</code> in the NetNS of the current Pod, and write the key to the mark of this port socket using <code>setsockopt</code>.</p>
<!--在 eBPF 中，我们通过 `bpf_sk_lookup_tcp` 取得端口 39807 的 Mark 信息，然后从 `mark_pod_ips_map` 中即可取得当前 NetNS（也是当期 Pod）的 IP。-->
<p>In eBPF, we get the recorded mark information of port <code>39807</code> through <code>bpf_sk_lookup_tcp</code>, and use it to get the current Pod IP (also the current NetNS) from <code>mark_pod_ips_map</code>.</p>
<!--有了当前 Pod IP 之后，我们可以根据这个 Pod 的配置，确认流量处理路径（比如 `excludeOutboundPorts`）。-->
<p>With the current Pod IP, we can determine the path to route traffic (such as <code>excludeOutboundPorts</code>) according to the configuration of this Pod.</p>
<!--同时，我们还使用 Pod 优化了之前解决四元组冲突的方案，改为使用 `bpf_bind` 绑定源 IP，目的 IP 直接使用 `127.0.0.1`，为了后续支持 IPv6 做准备。-->
<p>In addition, we also optimized the quadruple conflicts by using <code>bpf_bind</code> to bind the source IP and using <code>127.0.0.1</code> as the destination IP, which also prepares for future support of IPv6.</p>
<!--### 如何处理入口流量？-->
<h3 id="how-to-handle-ingress-traffic">How to handle ingress traffic</h3>
<!--为了能够处理入口流量，我们引入了 XDP 程序，XDP 程序作用在网卡上，能够对原始数据包做修改。-->
<p>In order to handle inbound traffic, we introduced the XDP program, which works on the network card and can modify the original data packets.</p>
<!--我们借助 XDP 程序，在流量到达 Pod 的时候，修改目的端口为 15006 以完成流量转发。-->
<p>We use the XDP program to modify the destination port as <code>15006</code> when the traffic reaches the Pod, so as to complete traffic forwarding.</p>
<!--同时考虑到可能存在主机直接访问 Pod 的情况，也为了减小影响范围，我们选择将 XDP 程序附加到 Pod 的网卡上。这就需要借助 CNI 的能力，在创建 Pod 时进行附加操作。-->
<p>At the same time, considering the possibility that the host directly accesses the Pod, and in order to reduce the scope of influence, we choose to attach the XDP program to the Pod&rsquo;s network card. This requires the ability of CNI to perform additional operations when creating Pods</p>
<!--## 如何体验 CNI 模式？-->
<h2 id="how-to-use-cni-mode">How to use CNI mode?</h2>
<!--CNI 模式默认被关闭，需要手动开启。-->
<p>CNI mode is disabled by default. You need to enable it manually with the following command.</p>
<!--可以使用以下命令一键开启：-->
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -sSL https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml <span style="color: #000; font-weight: bold;">|</span> sed <span style="color: #4e9a06;">'s/--cni-mode=false/--cni-mode=true/g'</span> <span style="color: #000; font-weight: bold;">|</span> kubectl apply -f -
</code></pre></div><!--## 注意事项-->
<h2 id="notes">Notes</h2>
<!--### CNI 模式处于测试阶段-->
<h3 id="cni-mode-is-in-beta">CNI mode is in beta</h3>
<!--CNI 模式刚被设计和开发出来，可能存在不少问题，我们欢迎大家在测试阶段进行反馈，或者提出更好的建议，以帮助我们改进 Merbridge！-->
<p>The CNI mode is a new feature that may not be perfect. We welcome your feedback and suggestions to help improve Merbridge.</p>
<!--如果需要使用注入 Istio perf benchmark 等工具进行测试性能，请开启 CNI 模式，否则会导致性能测试结果不准确。-->
<p>If you are trying to do benchmark test using tools like <code>Istio perf benchmark</code>, it is suggested to enable the CNI mode. Otherwise the test results will be inaccurate.</p>
<!--### 需要注意主机是否可开启 hardware-checksum 能力-->
<h3 id="check-whether-the-host-can-enable-the-hardware-checksum-capability">Check whether the host can enable the hardware-checksum capability</h3>
<!--为了保证 CNI 模式的正常运行，我们默认关闭了 hardware-checksum 能力，这可能会影响到网络性能。建议大家在开启 CNI 模式前，先确认主机是否可开启 hardware-checksum 能力。如果可以开启，建议设置 `--hardware-checksum=true` 以获得最佳的性能表现。-->
<p>In order to ensure the CNI mode works properly, the <code>hardware-checksum</code> capability is disabled by default, which may affect network performance. It is recommended to check whether you can enable this capability on the host before enabling the CNI mode. If yes, we suggest to set <code>--hardware-checksum=true</code> for best performance.</p>
<!--测试方法：`ethtool -k <网卡> | grep tx-checksum-ipv4` 为 on 表示开启。-->
<p>Test method: if <code>ethtool -k &lt;network card&gt; | grep tx-checksum-ipv4</code> is on, it means enabled.</p>
