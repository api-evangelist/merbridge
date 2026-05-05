---
title: "Blog: Test OSM Edge Integration with Merbridge"
url: "/blog/2023/04/03/osm-edge-demo/"
date: "Mon, 03 Apr 2023 00:00:00 +0000"
author: ""
feed_url: "https://merbridge.io/index.xml"
---
<p>This page will walk you through how to integrate Merbridge into <a href="https://github.com/flomesh-io/osm-edge">OSM Edge</a> for mesh acceleration.</p>
<blockquote>
<p>This demo originated from <a href="https://github.com/cybwan/osm-edge-start-demo/blob/main/demo/merbridge/README.zh.md">cybwan&rsquo;s personal space</a>.</p>
</blockquote>
<h2 id="1-deploy-k8s">1. Deploy k8s</h2>
<h3 id="11-preparation">1.1 Preparation</h3>
<ul>
<li>
<p><input checked="" disabled="" type="checkbox" /> Deploy 3 virtual machines of <strong>ubuntu 22.04/20.04</strong>, one as master node and the other two as worker nodes.</p>
</li>
<li>
<p><input checked="" disabled="" type="checkbox" /> Name them as <code>master</code>, <code>node1</code>, and <code>node2</code>.</p>
</li>
<li>
<p><input checked="" disabled="" type="checkbox" /> Modify <code>/etc/hosts</code> to enable hostname-based connectivity between the three nodes.</p>
</li>
<li>
<p><input checked="" disabled="" type="checkbox" /> Update apt packages：</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash">sudo apt -y update <span style="color: #ce5c00; font-weight: bold;">&amp;&amp;</span> sudo apt -y upgrade
</code></pre></div></li>
<li>
<p><input checked="" disabled="" type="checkbox" /> Use root account to run the following commands.</p>
</li>
</ul>
<h3 id="12--deploy-resources-for-container-initialization-on-each-vm">1.2 Deploy resources for container initialization on each VM</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init.sh -O
chmod u+x install-k8s-node-init.sh
<span style="color: #000;">system</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>uname -s <span style="color: #000; font-weight: bold;">|</span> tr <span style="color: #ce5c00; font-weight: bold;">[</span>:upper:<span style="color: #ce5c00; font-weight: bold;">]</span> <span style="color: #ce5c00; font-weight: bold;">[</span>:lower:<span style="color: #ce5c00; font-weight: bold;">]</span><span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">arch</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>dpkg --print-architecture<span style="color: #204a87; font-weight: bold;">)</span>
./install-k8s-node-init.sh <span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span> <span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>
</code></pre></div><h3 id="13-deploy-k8s-tools-on-each-vm">1.3 Deploy k8s tools on each VM</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init-tools.sh -O
chmod u+x install-k8s-node-init-tools.sh
<span style="color: #000;">system</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>uname -s <span style="color: #000; font-weight: bold;">|</span> tr <span style="color: #ce5c00; font-weight: bold;">[</span>:upper:<span style="color: #ce5c00; font-weight: bold;">]</span> <span style="color: #ce5c00; font-weight: bold;">[</span>:lower:<span style="color: #ce5c00; font-weight: bold;">]</span><span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">arch</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>dpkg --print-architecture<span style="color: #204a87; font-weight: bold;">)</span>
./install-k8s-node-init-tools.sh <span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span> <span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>
<span style="color: #204a87;">source</span> ~/.bashrc
</code></pre></div><h3 id="14-launch-k8s-related-services-on-the-master-node">1.4 Launch k8s-related services on the master node</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-master-start.sh -O
chmod u+x install-k8s-node-master-start.sh
<span style="color: #8f5902; font-style: italic;"># replace with the IP of your master node</span>
<span style="color: #000;">MASTER_IP</span><span style="color: #ce5c00; font-weight: bold;">=</span>192.168.127.80
<span style="color: #8f5902; font-style: italic;"># Set flannel as CNI</span>
<span style="color: #000;">CNI</span><span style="color: #ce5c00; font-weight: bold;">=</span>flannel
./install-k8s-node-master-start.sh <span style="color: #4e9a06;">${</span><span style="color: #000;">MASTER_IP</span><span style="color: #4e9a06;">}</span> <span style="color: #4e9a06;">${</span><span style="color: #000;">CNI</span><span style="color: #4e9a06;">}</span>
<span style="color: #8f5902; font-style: italic;"># Wait a while...</span>
</code></pre></div><h3 id="15-launch-k8s-related-services-on-the-worker-nodes">1.5 Launch k8s-related services on the worker nodes</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-worker-join.sh -O
chmod u+x install-k8s-node-worker-join.sh
<span style="color: #8f5902; font-style: italic;"># replace with the IP of your master node</span>
<span style="color: #000;">MASTER_IP</span><span style="color: #ce5c00; font-weight: bold;">=</span>192.168.127.80
<span style="color: #8f5902; font-style: italic;"># enter root passwords of Master node as required</span>
./install-k8s-node-worker-join.sh <span style="color: #4e9a06;">${</span><span style="color: #000;">MASTER_IP</span><span style="color: #4e9a06;">}</span>
</code></pre></div><h3 id="16-check-status-of-k8s-related-pods-on-the-master-node">1.6 Check status of k8s-related pods on the master node</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash">kubectl get pods -A -o wide
</code></pre></div><h2 id="2-deploy-osm-edge">2. Deploy osm-edge</h2>
<h3 id="21-download-and-install-osm-edge-clt">2.1 Download and install osm-edge CLT</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="color: #000;">system</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>uname -s <span style="color: #000; font-weight: bold;">|</span> tr <span style="color: #ce5c00; font-weight: bold;">[</span>:upper:<span style="color: #ce5c00; font-weight: bold;">]</span> <span style="color: #ce5c00; font-weight: bold;">[</span>:lower:<span style="color: #ce5c00; font-weight: bold;">]</span><span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">arch</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>dpkg --print-architecture<span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">release</span><span style="color: #ce5c00; font-weight: bold;">=</span>v1.3.3
curl -L https://github.com/flomesh-io/osm-edge/releases/download/<span style="color: #4e9a06;">${</span><span style="color: #000;">release</span><span style="color: #4e9a06;">}</span>/osm-edge-<span style="color: #4e9a06;">${</span><span style="color: #000;">release</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>.tar.gz <span style="color: #000; font-weight: bold;">|</span> tar -vxzf -
./<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>/osm version
cp ./<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>/osm /usr/local/bin/
</code></pre></div><h3 id="22-install-osm-edge">2.2 Install osm-edge</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="color: #204a87;">export</span> <span style="color: #000;">osm_namespace</span><span style="color: #ce5c00; font-weight: bold;">=</span>osm-system
<span style="color: #204a87;">export</span> <span style="color: #000;">osm_mesh_name</span><span style="color: #ce5c00; font-weight: bold;">=</span>osm
osm install <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --mesh-name <span style="color: #4e9a06;">"</span><span style="color: #000;">$osm_mesh_name</span><span style="color: #4e9a06;">"</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --osm-namespace <span style="color: #4e9a06;">"</span><span style="color: #000;">$osm_namespace</span><span style="color: #4e9a06;">"</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.registry<span style="color: #ce5c00; font-weight: bold;">=</span>flomesh <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.tag<span style="color: #ce5c00; font-weight: bold;">=</span>1.3.3 <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.certificateProvider.kind<span style="color: #ce5c00; font-weight: bold;">=</span>tresor <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.pullPolicy<span style="color: #ce5c00; font-weight: bold;">=</span>Always <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.enablePermissiveTrafficPolicy<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87;">true</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.controllerLogLevel<span style="color: #ce5c00; font-weight: bold;">=</span>warn <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>900s
</code></pre></div><p>If you want to deploy <code>osm</code>, use the following commands:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="color: #000;">system</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>uname -s <span style="color: #000; font-weight: bold;">|</span> tr <span style="color: #ce5c00; font-weight: bold;">[</span>:upper:<span style="color: #ce5c00; font-weight: bold;">]</span> <span style="color: #ce5c00; font-weight: bold;">[</span>:lower:<span style="color: #ce5c00; font-weight: bold;">]</span><span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">arch</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87; font-weight: bold;">$(</span>dpkg --print-architecture<span style="color: #204a87; font-weight: bold;">)</span>
<span style="color: #000;">release</span><span style="color: #ce5c00; font-weight: bold;">=</span>v1.2.3
curl -L https://github.com/openservicemesh/osm/releases/download/<span style="color: #4e9a06;">${</span><span style="color: #000;">release</span><span style="color: #4e9a06;">}</span>/osm-<span style="color: #4e9a06;">${</span><span style="color: #000;">release</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>.tar.gz <span style="color: #000; font-weight: bold;">|</span> tar -vxzf -
./<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>/osm version
cp ./<span style="color: #4e9a06;">${</span><span style="color: #000;">system</span><span style="color: #4e9a06;">}</span>-<span style="color: #4e9a06;">${</span><span style="color: #000;">arch</span><span style="color: #4e9a06;">}</span>/osm /usr/local/bin/
<span style="color: #204a87;">export</span> <span style="color: #000;">osm_namespace</span><span style="color: #ce5c00; font-weight: bold;">=</span>osm-system
<span style="color: #204a87;">export</span> <span style="color: #000;">osm_mesh_name</span><span style="color: #ce5c00; font-weight: bold;">=</span>osm
osm install <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --mesh-name <span style="color: #4e9a06;">"</span><span style="color: #000;">$osm_mesh_name</span><span style="color: #4e9a06;">"</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --osm-namespace <span style="color: #4e9a06;">"</span><span style="color: #000;">$osm_namespace</span><span style="color: #4e9a06;">"</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.registry<span style="color: #ce5c00; font-weight: bold;">=</span>openservicemesh <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.tag<span style="color: #ce5c00; font-weight: bold;">=</span>v1.2.3 <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.certificateProvider.kind<span style="color: #ce5c00; font-weight: bold;">=</span>tresor <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.image.pullPolicy<span style="color: #ce5c00; font-weight: bold;">=</span>Always <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.enablePermissiveTrafficPolicy<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #204a87;">true</span> <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --set<span style="color: #ce5c00; font-weight: bold;">=</span>osm.controllerLogLevel<span style="color: #ce5c00; font-weight: bold;">=</span>warn <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --verbose <span style="color: #4e9a06;">\
</span><span style="color: #4e9a06;"></span> --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>900s
</code></pre></div><h2 id="3-deploy-merbridge">3. Deploy Merbridge</h2>
<div class="highlight"><pre tabindex="0"><code class="language-bash">curl -L https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one-osm.yaml -O
sed -i <span style="color: #4e9a06;">'s/--cni-mode=false/--cni-mode=true/g'</span> all-in-one-osm.yaml
sed -i <span style="color: #4e9a06;">'/--cni-mode=true/a\\t\t- --debug=true'</span> all-in-one-osm.yaml
sed -i <span style="color: #4e9a06;">'s/\t/ /g'</span> all-in-one-osm.yaml
kubectl apply -f all-in-one-osm.yaml
sleep 5s
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n osm-system -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>merbridge --field-selector spec.nodeName<span style="color: #ce5c00; font-weight: bold;">==</span>master --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>1800s
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n osm-system -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>merbridge --field-selector spec.nodeName<span style="color: #ce5c00; font-weight: bold;">==</span>node1 --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>1800s
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n osm-system -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>merbridge --field-selector spec.nodeName<span style="color: #ce5c00; font-weight: bold;">==</span>node2 --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>1800s
</code></pre></div><h2 id="4-test-if-merbridge-replaces-iptables">4. Test if Merbridge replaces iptables</h2>
<h3 id="41-deploy-business-pods">4.1 Deploy business pods</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="color: #8f5902; font-style: italic;"># simulate business services</span>
kubectl create namespace demo
osm namespace add demo
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
<span style="color: #8f5902; font-style: italic;"># schedule pods to different nodes</span>
kubectl patch deployments sleep -n demo -p <span style="color: #4e9a06;">'{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'</span>
kubectl patch deployments helloworld-v1 -n demo -p <span style="color: #4e9a06;">'{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'</span>
kubectl patch deployments helloworld-v2 -n demo -p <span style="color: #4e9a06;">'{"spec":{"template":{"spec":{"nodeName":"node2"}}}}'</span>
<span style="color: #8f5902; font-style: italic;"># wait supportive pods to run</span>
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n demo -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>sleep --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>180s
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n demo -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>helloworld -l <span style="color: #000;">version</span><span style="color: #ce5c00; font-weight: bold;">=</span>v1 --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>180s
kubectl <span style="color: #204a87;">wait</span> --for<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">condition</span><span style="color: #ce5c00; font-weight: bold;">=</span>ready pod -n demo -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>helloworld -l <span style="color: #000;">version</span><span style="color: #ce5c00; font-weight: bold;">=</span>v2 --timeout<span style="color: #ce5c00; font-weight: bold;">=</span>180s
</code></pre></div><h3 id="42scenarios-test">4.2 Scenarios Test</h3>
<h4 id="421-monitor-kernel-logs-on-node1-and-node2">4.2.1 Monitor kernel logs on node1 and node2</h4>
<div class="highlight"><pre tabindex="0"><code class="language-bash">cat /sys/kernel/debug/tracing/trace_pipe<span style="color: #000; font-weight: bold;">|</span>grep bpf_trace_printk<span style="color: #000; font-weight: bold;">|</span>grep -E <span style="color: #4e9a06;">"rewritten|redirect"</span>
</code></pre></div><h4 id="422-command-for-testing">4.2.2 Command for testing</h4>
<p>Run it multiple times:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash">kubectl <span style="color: #204a87;">exec</span> <span style="color: #204a87; font-weight: bold;">$(</span>kubectl get po -l <span style="color: #000;">app</span><span style="color: #ce5c00; font-weight: bold;">=</span>sleep -n demo -o<span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #000;">jsonpath</span><span style="color: #ce5c00; font-weight: bold;">=</span><span style="color: #4e9a06;">'{..metadata.name}'</span><span style="color: #204a87; font-weight: bold;">)</span> -n demo -c sleep -- curl -s helloworld:5000/hello
</code></pre></div><h4 id="423-test-results">4.2.3 Test results</h4>
<p>The expected output should be like:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash">Hello version: v1, instance: helloworld-v1-5d46f78b4c-hghcj
Hello version: v2, instance: helloworld-v2-6b56769f9d-stwrj
</code></pre></div>
