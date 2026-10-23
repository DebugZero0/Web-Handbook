# Kubernetes Networking

## 1. Why do Pods need an IP address?
In traditional Docker setups, containers usually share the host machine’s network. If you want to run two apps on port 80, they conflict, so you have to manually map them to random host ports (e.g., container port 80 maps to host port 32000). This gets messy very quickly at scale.

Kubernetes solves this by giving every single Pod its own dedicated IP address. This means:
<ul>
<li>No port conflicts: Your app thinks it is running on its own dedicated virtual machine with full access to standard ports (like port 80 or 8080).</li>
<li>Identical view: A Pod sees itself using its own IP regardless of which physical server or virtual machine it happens to land on.</li>
<li>Easy communication: Other components can talk to a Pod directly via standard TCP/IP networking without dealing with complex routing tables.</li>
</ul>

## 2. Who is connected to whom? (The 4 Core Communication Models)
Kubernetes networking covers four distinct communication scenarios:   
<ol>
<li>Container-to-Container (Inside the same Pod):   
<ul>
<li>Multiple containers can live inside a single Pod. These containers share the exact same Network Namespace (including the same IP address and loopback interface).</li>
<li>How they talk: They connect via localhost using different ports (e.g., a web app on localhost:8080 talking to a sidecar log collector on localhost:9001).</li>
</ul>
<li>Pod-to-Pod: 
<ul>
<li>This is the primary backbone of Kubernetes. Every Pod can talk to any other Pod directly, whether they live on the same physical node or on completely different worker nodes.</li>
<li>Crucially, there is no NAT (Network Address Translation)—Pod A sees Pod B's actual IP, and Pod B sees Pod A's actual IP.</li>
</ul>
<li>Pod-to-Service: 
<ul>
<li>Pods are ephemeral; they constantly crash, scale up, scale down, or get rescheduled, which means their IP addresses change constantly.</li>
<li>To solve this, Kubernetes introduces a Service. A Service acts as a stable "Virtual IP" and DNS name that front-ends a changing group of Pods. Pods talk to the Service name, and the Service automatically load-balances the traffic to healthy backend Pods.</li>
</ul>
</li>
<li>External-to-Internal: 
<ul>
<li>How traffic from the outside world enters the cluster (handled via resources like NodePort, LoadBalancers, or Ingress controllers).</li>
</ul>
</li>
</ol>