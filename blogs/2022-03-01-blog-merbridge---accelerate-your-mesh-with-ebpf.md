---
title: "Blog: Merbridge - Accelerate your mesh with eBPF"
url: "/blog/2022/03/01/merbridge-introduce/"
date: "Tue, 01 Mar 2022 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
<h1 id="merbridge---accelerate-your-mesh-with-ebpf">Merbridge - Accelerate your mesh with eBPF</h1>
<p><em>Replacing iptables rules with eBPF allows transporting data directly from inbound sockets to outbound sockets, shortening the datapath between sidecars and services.</em></p>
<h2 id="introduction">Introduction</h2>
<p>The secret of Istio’s abilities in traffic management, security, observability and policy is all in the Envoy proxy. Istio uses Envoy as the “sidecar” to intercept service traffic, with the kernel’s <code>netfilter</code> packet filter functionality configured by iptables.</p>
<p>There are shortcomings in using iptables to perform this interception. Since netfilter is a highly versatile tool for filtering packets, several routing rules and data filtering processes are applied before reaching the destination socket. For example, from the network layer to the transport layer, netfilter will be used for processing for several times with the rules predefined, like <code>pre_routing</code>, <code>post_routing</code> and etc. When the packet becomes a TCP packet or UDP packet, and is forwarded to user space, some additional steps like packet validation, protocol policy processing and destination socket searching will be performed. When a sidecar is configured to intercept traffic, the original data path can become very long, since duplicated steps are performed several times.</p>
<p>Over the past two years, eBPF has become a trending technology, and many projects based on <a href="https://ebpf.io/">eBPF</a> have been released to the community. Tools like <a href="https://cilium.io/">Cilium</a> and <a href="https://px.dev/">Pixie</a> show great use cases for eBPF in observability and network packet processing. With eBPF’s <code>sockops</code> and <code>redir</code> capabilities, data packets can be processed efficiently by directly being transported from an inbound socket to an outbound socket. In an Istio mesh, it is possible to use eBPF to replace iptables rules, and accelerate the data plane by shortening the data path.</p>
<p>We have created an open source project called Merbridge, and by applying the following command to your Istio-managed cluster, you can use eBPF to achieve such network acceleration.</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash">kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml
</code></pre></div><blockquote>
<p>Attention: Merbridge uses eBPF functions which require a Linux kernel version ≥ 5.7.</p>
</blockquote>
<p>With Merbridge, the packet datapath can be shortened directly from one socket to another destination socket, and here’s how it works.</p>
<h3 id="using-ebpf-sockops-for-performance-optimization">Using eBPF <code>sockops</code> for performance optimization</h3>
<p>Network connection is essentially socket communication. eBPF provides a function <a href="https://man7.org/linux/man-pages/man7/bpf-helpers.7.html">bpf_msg_redirect_hash</a>, to directly forward the packets sent by the application in the inbound socket to the outbound socket. By entering the function mentioned before, developers can perform any logic to decide the packet destination. According to this characteristic, the datapath of packets can noticeably be optimized in the kernel.</p>
<p>The <code>sock_map</code> is the crucial piece in recording information for packet forwarding. When a packet arrives, an existing socket is selected from the <code>sock_map</code> to forward the packet. As a result, we need to save all the socket information for packets to make the transportation process function properly. When there are new socket operations — like a new socket being created — the <code>sock_ops</code> function is executed. The socket metadata is obtained and stored in the <code>sock_map</code> to be used when processing packets. The common key type in the <code>sock_map</code> is a “quadruple” of source and destination addresses and ports. With the key and the rules stored in the map, the destination socket will be found when a new packet arrives.</p>
<h2 id="the-merbridge-approach">The Merbridge approach</h2>
<p>Let’s introduce the detailed design and implementation principles of Merbridge step by step, with a real scenario.</p>
<h3 id="istio-sidecar-traffic-interception-based-on-iptables">Istio sidecar traffic interception based on iptables</h3>
<p><img alt="Istio Sidecar Traffic Interception Based on iptables" src="./imgs/1.png" /></p>
<p>When external traffic hits your application’s ports, it will be intercepted by a <code>PREROUTING</code> rule in iptables, forwarded to port 15006 of the sidecar container, and handed over to Envoy for processing. This is shown as steps 1-4 in the red path in the above diagram.</p>
<p>Envoy processes the traffic using the policies issued by the Istio control plane. If allowed, the traffic will be sent to the actual container port of the application container.</p>
<p>When the application tries to access other services, it will be intercepted by an <code>OUTPUT</code> rule in iptables, and then be forwarded to port 15001 of the sidecar container, where Envoy is listening. This is steps 9-12 in the red path, similar to inbound traffic processing.</p>
<p>Traffic to the application port needs to be forwarded to the sidecar, then sent to the container port from the sidecar port, which is overhead. Moreover, iptables’ versatility determines that its performance is not always ideal because it inevitably adds delays to the whole datapath with different filtering rules applied. Although iptables is the common way to do packet filtering, in the Envoy proxy case, the longer datapath amplifies the bottleneck of packet filtering process in the kernel.</p>
<p>If we use <code>sockops</code> to directly connect the sidecar’s socket to the application’s socket, the traffic will not need to go through iptables rules, and thus performance can be improved.</p>
<h3 id="processing-outbound-traffic">Processing outbound traffic</h3>
<p>As mentioned above, we would like to use eBPF’s <code>sockops</code> to bypass iptables to accelerate network requests. At the same time, we also do not want to modify any parts of Istio, to make Merbridge fully adaptive to the community version. As a result, we need to simulate what iptables does in eBPF.</p>
<p>Traffic redirection in iptables utilizes its <code>DNAT</code> function. When trying to simulate the capabilities of iptables using eBPF, there are two main things we need to do:</p>
<ol>
<li>Modify the destination address, when the connection is initiated, so that traffic can be sent to the new interface.</li>
<li>Enable Envoy to identify the original destination address, to be able to identify the traffic.</li>
</ol>
<p>For the first part, we can use eBPF’s <code>connect</code> program to process it, by modifying <code>user_ip</code> and <code>user_port</code>.</p>
<p>For the second part, we need to understand the concept of <code>ORIGINAL_DST</code> which belongs to the <code>netfilter</code> module in the kernel.</p>
<p>When an application (including Envoy) receives a connection, it will call the <code>get_sockopt</code> function to obtain <code>ORIGINAL_DST</code>. If going through the iptables <code>DNAT</code> process, iptables will set this parameter, with the “original IP + port” value, to the current socket. Thus, the application can get the original destination address according to the connection.</p>
<p>We have to modify this call process through eBPF’s <code>get_sockopts</code> function. (<code>bpf_setsockopt</code> is not used here because this parameter does not currently support the optname of <code>SO_ORIGINAL_DST</code>).</p>
<p>Referring to the figure below, when an application initiates a request, it will go through the following steps:</p>
<ol>
<li>When the application initiates a connection, the <code>connect</code> program will modify the destination address to <code>127.x.y.z:15001</code>, and use <code>cookie_original_dst</code> to save the original destination address.</li>
<li>In the <code>sockops</code> program, the current socket information and the quadruple are saved in <code>sock_pair_map</code>. At the same time, the same quadruple and its corresponding original destination address will be written to <code>pair_original_dst</code>. (Cookie is not used here because it cannot be obtained in the <code>get_sockopt</code> program).</li>
<li>After Envoy receives the connection, it will call the <code>get_sockopt</code> function to read the destination address of the current connection. <code>get_sockopt</code> will extract and return the original destination address from <code>pair_original_dst</code>, according to the quadruple information. Thus, the connection is completely established.</li>
<li>In the data transport step, the <code>redir</code> program will read the sock information from <code>sock_pair_map</code> according to the quadruple information, and then forward it directly through <code>bpf_msg_redirect_hash</code> to speed up the request.</li>
</ol>
<p><img alt="Processing Outbound Traffic" src="./imgs/2.png" /></p>
<p>Why do we set the destination address to <code>127.x.y.z</code> instead of <code>127.0.0.1</code>? When different pods exist, there might be conflicting quadruples, and this gracefully avoids conflict. (Pods’ IPs are different, and they will not be in the conflicting condition at any time.)</p>
<h3 id="inbound-traffic-processing">Inbound traffic processing</h3>
<p>The processing of inbound traffic is basically similar to outbound traffic, with the only difference: revising the port of the destination to 15006.</p>
<p>It should be noted that since eBPF cannot take effect in a specified namespace like iptables, the change will be global, which means that if we use a Pod that is not originally managed by Istio, or an external IP address, serious problems will be encountered — like the connection not being established at all.</p>
<p>As a result, we designed a tiny control plane (deployed as a DaemonSet), which watches all pods — similar to the kubelet watching pods on the node — to write the pod IP addresses that have been injected into the sidecar to the <code>local_pod_ips</code> map.</p>
<p>When processing inbound traffic, if the destination address is not in the map, we will not do anything to the traffic.</p>
<p>The other steps are the same as for outbound traffic.</p>
<p><img alt="Processing Inbound Traffic" src="./imgs/3.png" /></p>
<h3 id="same-node-acceleration">Same-node acceleration</h3>
<p>Theoretically, acceleration between Envoy sidecars on the same node can be achieved directly through inbound traffic processing. However, Envoy will raise an error when accessing the application of the current pod in this scenario.</p>
<p>In Istio, Envoy accesses the application by using the current pod IP and port number. With the above scenario, we realized that the pod IP exists in the <code>local_pod_ips</code> map as well, and the traffic will be redirected to the pod IP on port 15006 again because it is the same address that the inbound traffic comes from. Redirecting to the same inbound address causes an infinite loop.</p>
<p>Here comes the question: are there any ways to get the IP address in the current namespace with eBPF? The answer is yes!</p>
<p>We have designed a feedback mechanism: When Envoy tries to establish the connection, we redirect it to port 15006. However, in the <code>sockops</code> step, we will determine if the source IP and the destination IP are the same. If yes, it means the wrong request is sent, and we will discard this connection in the <code>sockops</code> process. In the meantime, the current <code>ProcessID</code> and <code>IP</code> information will be written into the <code>process_ip</code> map, to allow eBPF to support correspondence between processes and IPs.</p>
<p>When the next request is sent, the same process need not be performed again. We will check directly from the <code>process_ip</code> map if the destination address is the same as the current IP address.</p>
<blockquote>
<p>Envoy will retry when the request fails, and this retry process will only occur once, meaning subsequent requests will be accelerated.</p>
</blockquote>
<p><img alt="Same-node acceleration" src="./imgs/4.png" /></p>
<h3 id="connection-relationship">Connection relationship</h3>
<p>Before applying eBPF using Merbridge, the data path between pods is like:</p>
<p><img alt="iptables&rsquo;s data path" src="./imgs/5.png" /></p>
<blockquote>
<p>Diagram From: <a href="https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22">Accelerating Envoy and Istio with Cilium and the Linux Kernel</a></p>
</blockquote>
<p>After applying Merbridge, the outbound traffic will skip many filter steps to improve the performance:</p>
<p><img alt="eBPF&rsquo;s data path" src="./imgs/6.png" /></p>
<blockquote>
<p>Diagram From: <a href="https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22">Accelerating Envoy and Istio with Cilium and the Linux Kernel</a></p>
</blockquote>
<p>If two pods are on the same machine, the connection can even be faster:</p>
<p><img alt="eBPF&rsquo;s data path on the same machine" src="./imgs/7.png" /></p>
<blockquote>
<p>Diagram From: <a href="https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22">Accelerating Envoy and Istio with Cilium and the Linux Kernel</a></p>
</blockquote>
<h2 id="performance-results">Performance results</h2>
<blockquote>
<p>The below tests are from our development, and not yet validated in production use cases.</p>
</blockquote>
<p>Let’s see the effect on overall latency using eBPF instead of iptables (lower is better):</p>
<p><img alt="Latency vs Client Connections Graph" src="./imgs/8.png" /></p>
<p>We can also see overall QPS after using eBPF (higher is better):</p>
<p><img alt="QPS vs Client Connections Graph" src="./imgs/9.png" /></p>
<blockquote>
<p>Test results are generated with <code>wrk</code>.</p>
</blockquote>
<h2 id="summary">Summary</h2>
<p>We have introduced the core ideas of Merbridge in this post. By replacing iptables with eBPF, the data transportation process can be accelerated in a mesh scenario. At the same time, Istio will not be changed at all. This means if you do not want to use eBPF any more, just delete the DaemonSet, and the datapath will be reverted to the traditional iptables-based routing without any problems.</p>
<p>Merbridge is a completely independent open source project. It is still at an early stage, and we are looking forward to having more users and developers to get engaged. It would be greatly appreciated if you would try this new technology to accelerate your mesh, and provide us with some feedback!</p>
<p>Merbridge Project: <a href="https://github.com/merbridge/merbridge">https://github.com/merbridge/merbridge</a></p>
<h2 id="see-also">See also</h2>
<ul>
<li>
<p><a href="https://ebpf.io/">https://ebpf.io/</a></p>
</li>
<li>
<p><a href="https://cilium.io/">https://cilium.io/</a></p>
</li>
<li>
<p><a href="https://github.com/merbridge/merbridge">Merbridge on GitHub</a></p>
</li>
<li>
<p><a href="https://developpaper.com/kubecon-2021-%EF%BD%9C-using-ebpf-instead-of-iptables-to-optimize-the-performance-of-service-grid-data-plane/">Using eBPF instead of iptables to optimize the performance of service grid data plane</a> by Liu Xu, Tencent</p>
</li>
<li>
<p><a href="https://jimmysong.io/en/blog/sidecar-injection-iptables-and-traffic-routing/">Sidecar injection and transparent traffic hijacking process in Istio explained in detail</a> by Jimmy Song, Tetrate</p>
</li>
<li>
<p><a href="https://01.org/blogs/xuyizhou/2021/accelerate-istio-dataplane-ebpf-part-1">Accelerate the Istio data plane with eBPF</a> by Yizhou Xu, Intel</p>
</li>
<li>
<p><a href="https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/original_dst_filter">Envoy&rsquo;s Original Destination filter</a></p>
</li>
<li>
<p><a href="https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22">Accelerating Envoy and Istio with Cilium and the Linux Kernel</a></p>
</li>
</ul>
