---
title: "Blog: Merbridge now supports Ambient Mesh, no worry about CNI compatibility!"
url: "/blog/2022/11/11/ambient-mesh-support/"
date: "Fri, 11 Nov 2022 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
<p>In the blog <a href="../ambient-mesh-data-path/index.md">Deep Dive into Ambient Mesh - Traffic Path</a>,
we analyzed how Ambient Mesh forwards the ingress and egress traffic of Pod to ztunnel.
It is implemented by iptables + TPROXY + routing table. The traffic datapath is relatively
long compared to sidecar mode, and the principle is complicated. Moreover, it uses routing marks, which may cause
unexpected behaviors in some cases when it relies on CNI or it is running in a CNI with the bridge mode.
These severely limit the applicable scope of ambient mesh.</p>
<p>The main purpose of Merbridge is to replace iptables with eBPF to accelerate applications running in
a service mesh. Ambient Mesh is a new mode of Istio. It is necessary for Merbridge to support this new mode.
iptables is a powerful tool to block unwanted traffic, allow desired traffic, and redirect packets to specific
addresses and ports, but it also has some weaknesses. First, iptables uses a linear matching method.
When several applications simultaneously call a same program, conflicts may arise and make some features become unavailable.
Second, although it is flexible enough, it still cannot be programmed as freely as eBPF.
Therefore, replacing iptables with eBPF can help Ambient Mesh achieve traffic interception.</p>
<h2 id="objectives">Objectives</h2>
<p>As mentioned in <a href="../ambient-mesh-data-path/index.md">Deep Dive into Ambient Mesh - Traffic Path</a>,
we set two objectives:</p>
<ul>
<li>Outgoing traffic from pods in Ambient Mesh should be intercepted and redirected to port 15001 of ztunnel.</li>
<li>Traffic sent from host applications to pods in Ambient Mesh should be redirected to port 15006 of ztunnel.</li>
</ul>
<p>Since istioin and other network interface cards (NICs) are completely designed to adapt to the native Ambient Mesh, we don&rsquo;t need to make any changes.</p>
<h2 id="pain-points-analysis">Pain points analysis</h2>
<p>Ambient Mesh has a different operation mechanism from sidecars. According to the official definition of Istio,
adding a Pod to Ambient Mesh does not require restarting the Pod and no any sidecar-related process is running in the Pod. It means:</p>
<ol>
<li>Merbridge used the CNI mode to enable the eBPF program to get the current Pod IP to make policy decisions,
which is incompatible with ambient mesh. The reason is a pod will not be restarted after joining or
leaving the ambient mesh, nor will it call the CNI plug-in.</li>
<li>In the sidecar mode, the only thing you need to change is the destination address to <code>127.0.0.1:15001</code> in
the connect hook of eBPF, but in the ambient mesh you need to replace the desitination IP with that of ztunnel.</li>
</ol>
<p>In addtion, no sidecar-related process exists in a Pod running in an ambient mesh,
so the legacy method of checking whether a port such as 15006 is listening in the current Pod is no longer applicable.
It is necessary to redesign the scheme to check the environment where processes are running.</p>
<p>Therefore, based on the above analysis, it is required to redesign the entire interception scheme so that Merbridge can support the ambient mesh.</p>
<p>In summary, we need to implement the following features:</p>
<ul>
<li>Redesign a scheme for judging whether a Pod is running in the ambient mesh</li>
<li>Use eBPF to perceive current Pod IP regardless of CNIs</li>
<li>Enable eBPF programs to know the ztunnel IP on the current node</li>
</ul>
<h2 id="solution">Solution</h2>
<p>In version 0.7.2, cgroup id is used to improve the performance of the connect program.
Usually, each container in a Pod has a proper cgroup id, which can be obtained through the <code>bpf_get_current_cgroup_id</code>
function in the BPF program. The speed of the connect program can be optimized by writing IP information to a specific <code>cgroup_info_map</code>.</p>
<p>An ambient mesh is different from the legacy CNI listening on a special port in the network namespace for storing Pod-related information.
In the ambient mesh, cgroup id is useful. If cgroup id can be associated with the Pod IP, you can get the current Pod IP in eBPF.</p>
<p>Since CNI cannot be relied on anymore, we need change the scheme for obtaining the information of Pod status.
For this reason, we detect the creation and revocation actions of local Pods by watching the process creation and revocation.
We created a new tool to watch the process changes on a host: <a href="https://github.com/merbridge/process-watcher">process-watcher project</a>.</p>
<p>Read the cgroup id and ip information from the process ID and writing it to the <code>cgroup_info_map</code>.</p>
<div class="highlight"><pre tabindex="0"><code class="language-go"><span style="color: #000;">tcg</span> <span style="color: #ce5c00; font-weight: bold;">:=</span> <span style="color: #000;">cgroupInfo</span><span style="color: #000; font-weight: bold;">{</span>
<span style="color: #000;">ID</span><span style="color: #000; font-weight: bold;">:</span> <span style="color: #000;">cgroupInode</span><span style="color: #000; font-weight: bold;">,</span>
<span style="color: #000;">IsInMesh</span><span style="color: #000; font-weight: bold;">:</span> <span style="color: #000;">in</span><span style="color: #000; font-weight: bold;">,</span>
<span style="color: #000;">CgroupIp</span><span style="color: #000; font-weight: bold;">:</span> <span style="color: #ce5c00; font-weight: bold;">*</span><span style="color: #000; font-weight: bold;">(</span><span style="color: #ce5c00; font-weight: bold;">*</span><span style="color: #000; font-weight: bold;">[</span><span style="color: #0000cf; font-weight: bold;">4</span><span style="color: #000; font-weight: bold;">]</span><span style="color: #204a87; font-weight: bold;">uint32</span><span style="color: #000; font-weight: bold;">)(</span><span style="color: #000;">_ip</span><span style="color: #000; font-weight: bold;">),</span>
<span style="color: #000;">Flags</span><span style="color: #000; font-weight: bold;">:</span> <span style="color: #000;">flag</span><span style="color: #000; font-weight: bold;">,</span>
<span style="color: #000;">DetectedFlags</span><span style="color: #000; font-weight: bold;">:</span> <span style="color: #000;">cgrinfo</span><span style="color: #000; font-weight: bold;">.</span><span style="color: #000;">DetectedFlags</span> <span style="color: #000; font-weight: bold;">|</span> <span style="color: #000;">AMBIENT_MESH_MODE_FLAG</span> <span style="color: #000; font-weight: bold;">|</span> <span style="color: #000;">ZTUNNEL_FLAG</span><span style="color: #000; font-weight: bold;">,</span>
<span style="color: #000; font-weight: bold;">}</span>
<span style="color: #204a87; font-weight: bold;">return</span> <span style="color: #000;">ebpfs</span><span style="color: #000; font-weight: bold;">.</span><span style="color: #000;">GetCgroupInfoMap</span><span style="color: #000; font-weight: bold;">().</span><span style="color: #000;">Update</span><span style="color: #000; font-weight: bold;">(</span><span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">cgroupInode</span><span style="color: #000; font-weight: bold;">,</span> <span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">tcg</span><span style="color: #000; font-weight: bold;">,</span> <span style="color: #000;">ebpf</span><span style="color: #000; font-weight: bold;">.</span><span style="color: #000;">UpdateAny</span><span style="color: #000; font-weight: bold;">)</span>
</code></pre></div><p>Then get the current cgroup-related information in eBPF:</p>
<div class="highlight"><pre tabindex="0"><code class="language-c"><span style="color: #000;">__u64</span> <span style="color: #000;">cgroup_id</span> <span style="color: #ce5c00; font-weight: bold;">=</span> <span style="color: #000;">bpf_get_current_cgroup_id</span><span style="color: #000; font-weight: bold;">();</span>
<span style="color: #204a87; font-weight: bold;">void</span> <span style="color: #ce5c00; font-weight: bold;">*</span><span style="color: #000;">info</span> <span style="color: #ce5c00; font-weight: bold;">=</span> <span style="color: #000;">bpf_map_lookup_elem</span><span style="color: #000; font-weight: bold;">(</span><span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">cgroup_info_map</span><span style="color: #000; font-weight: bold;">,</span> <span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">cgroup_id</span><span style="color: #000; font-weight: bold;">);</span>
</code></pre></div><p>Now, we can learn whether the current container has the ambient mesh enabled and it is located in a mesh or not.</p>
<p>Second, for the ztunnel IP, Istio implements it by adding NIC and binding fixed IPs.
This scheme may have the risk of conflict, and the original addresses may be lost in some cases (such as SNAT).
So Merbridge gives up the scheme and directly obtains the ztunnel IPs on the control plane,
writes it into the map, and enables the eBPF program read it (this is faster).</p>
<div class="highlight"><pre tabindex="0"><code class="language-c"><span style="color: #204a87; font-weight: bold;">static</span> <span style="color: #204a87; font-weight: bold;">inline</span> <span style="color: #000;">__u32</span> <span style="color: #ce5c00; font-weight: bold;">*</span><span style="color: #000;">get_ztunnel_ip</span><span style="color: #000; font-weight: bold;">()</span>
<span style="color: #000; font-weight: bold;">{</span>
<span style="color: #000;">__u32</span> <span style="color: #000;">ztunnel_ip_key</span> <span style="color: #ce5c00; font-weight: bold;">=</span> <span style="color: #000;">ZTUNNEL_KEY</span><span style="color: #000; font-weight: bold;">;</span>
<span style="color: #204a87; font-weight: bold;">return</span> <span style="color: #000; font-weight: bold;">(</span><span style="color: #000;">__u32</span> <span style="color: #ce5c00; font-weight: bold;">*</span><span style="color: #000; font-weight: bold;">)</span><span style="color: #000;">bpf_map_lookup_elem</span><span style="color: #000; font-weight: bold;">(</span><span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">settings</span><span style="color: #000; font-weight: bold;">,</span> <span style="color: #ce5c00; font-weight: bold;">&amp;</span><span style="color: #000;">ztunnel_ip_key</span><span style="color: #000; font-weight: bold;">);</span>
<span style="color: #000; font-weight: bold;">}</span>
</code></pre></div><p>Then use the connect program to rewrite the destination address:</p>
<div class="highlight"><pre tabindex="0"><code class="language-c"><span style="color: #000;">ctx</span><span style="color: #ce5c00; font-weight: bold;">-&gt;</span><span style="color: #000;">user_ip4</span> <span style="color: #ce5c00; font-weight: bold;">=</span> <span style="color: #000;">ztunnel_ip</span><span style="color: #000; font-weight: bold;">[</span><span style="color: #0000cf; font-weight: bold;">3</span><span style="color: #000; font-weight: bold;">];</span>
<span style="color: #000;">ctx</span><span style="color: #ce5c00; font-weight: bold;">-&gt;</span><span style="color: #000;">user_port</span> <span style="color: #ce5c00; font-weight: bold;">=</span> <span style="color: #000;">bpf_htons</span><span style="color: #000; font-weight: bold;">(</span><span style="color: #000;">OUT_REDIRECT_PORT</span><span style="color: #000; font-weight: bold;">);</span>
</code></pre></div><p>With the association with the cgroup id, the Pod IP of current processes can be obtained in eBPF, so as to enforce policies.
Forward the traffic from the Pod in the ambient mesh to the ztunnel, so that Merbridge can be compatible with the ambient mesh.</p>
<p>This will be a capability that is adaptable to all CNIs and can avoid the problem that the native ambient mesh cannot work well in most CNI modes.</p>
<h2 id="usage-and-feedback">Usage and feedback</h2>
<p>Since the ambient mesh is still in its early stage and the support for ambient mode is relatively preliminary,
some problems have not been well resolved, so the code of supporting for the ambient mode has not been merged into the main branch.
If you want to experience the capability of Merbridge to implement traffic interception for ambient mesh instead of iptables,
you can perform the following steps (it is required to install the ambient mesh in advance):</p>
<ol>
<li>Disable Istio CNI (set <code>--set components.cni.enabled=false</code> during installation, or delete Istio CNI&rsquo;s DaemonSet <code>kubectl -n istio-system delete ds istio-cni</code>).</li>
<li>Remove the init container of ztunnel (because it initializes iptables rules and NICs, which is not required for Merbridge).</li>
<li>Install Merbridge by running <code>kubectl apply -f https://github.com/merbridge/merbridge/raw/ambient/deploy/all-in-one.yaml</code></li>
</ol>
<p>After the Merbridge is ready, you can use all capabilities of ambient mesh.</p>
<p><strong>*Attentions:</strong></p>
<ol>
<li>The Ambient mode under Kind is not supported currently (we have a plan to support it in the future)</li>
<li>The host kernel version needs to be not less than 5.7</li>
<li>cgroup v2 is required to be enabled</li>
<li>This mode is also compatible with sidecars</li>
<li>The debug mode will be enabled by default in an ambient mesh, which will have certain impact on performance</li>
</ol>
<p>For more details see <a href="https://github.com/merbridge/merbridge/tree/ambient">source code</a>.</p>
<p>If you have any question, please reach out to us with <a href="https://join.slack.com/t/merbridge/shared_invite/zt-11uc3z0w7-DMyv42eQ6s5YUxO5mZ5hwQ">Slack</a> or add the wechat group to chat.</p>
