<h3 align="center"><img width="90%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/Federate.jpg"" /></h3>

## About Work
This work was originally published in 4 August 2020 at APAN50 by 
- Mr Chanpol Kongsute
- Assoc. Prof. Dr Chaodit Aswakul
<h2 align="center"><a href="https://github.com/IoTcloudServe/Establishment-of-IoTcloudServe-TEIN-testbed-infrastructure/tree/master/An%20Experimental%20Study%20of%20Kubernetes%20Cluster%20Peer-to-Peer%20Application-Level%20Federation%20via%20Istio%20Service%20Mesh">Please Click Here for More Detail</a></h2>

## The objective
we propose a pragmatic implementation framework in which Kubernetes clusters are federated in application-level as a peer-to-peer federation through Istio service mesh. This framework can be instantiated by configuring an Istio gateway to act as a cross-cluster load balancer handling traffic between clusters. This approach allows use cases such as cloud bursting and canary deployment. As a verification of this proposed federation framework, we use two Kubernetes clusters installed with echo servers. And by sending requests to the echo server locating in the cluster, we observe the behaviors and limitations of our proposed model.

## The Diagram of Proposed Medel (Cross-cluster communication)
Our design is based on the peer-to-peer federation model, where all management resources are distributed across all clusters, so there is no hierarchical relationship between clusters. In other words, each cluster will install its own Istio control plane. The only shared component is a common root certificate authority, which is needed for authentication in cross-cluster communication.
<h3 align="center"><img width="90%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/diagram1.png" /></h3>
In this model, cross-cluster communication is sent to the local Istio gateway and routed to the Istio gateway of another cluster by mTLS protocol. And this routing process can be configured so that an Istio gateway acts like a load balancer that controls traffic between clusters.

Somehow, we can also assign the rule to control incomming/out going traffic as the following picture below
<h3 align="center"><img width="90%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/rule_control.png" /></h3>


## Tools for Demonstratation
<h3 align="center"><img width="50%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/compo.png" /></h3>

---------------------------------------------------------------------
- Istio 1.5.10 <img width=4% src="/Agenda/picture/istio.png"/>
- Kiali <img width=4% src="/Agenda/picture/kiali.png"/>
- Python3 for HTTP parallel requests
```
import asyncio
import aiohttp
import time

websites = """http://202.28.193.112
http://202.28.193.112
http://202.28.193.112
http://202.28.193.112
http://202.28.193.112
http://202.28.193.112
http://202.28.193.112
"""

async def get(url):
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url=url) as response:
                resp = await response.read()
                print("Successfully got url {} with response of length {}.".format(url, len(resp)))
    except Exception as e:
        print("Unable to get url {} due to {}.".format(url, e.__class__))

async def main(urls, amount):
    ret = await asyncio.gather(*[get(url) for url in urls])
    print("Finalized all. ret is a list of len {} outputs.".format(len(ret)))


urls = websites.split("\n")
amount = len(urls)

start = time.time()
asyncio.run(main(urls, amount))
end = time.time()

print("Took {} seconds to pull {} websites.".format(end - start, amount))
```
- echo server image (k8s.gcr.io/echoserver:1.10)
- Destination rule in .yaml file (for only cluster 1)
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo
spec:
  host: echo.istio.svc.cluster.local
      #################
    ### Define policy ###
      #################
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http2MaxRequests: 1
        maxRequestsPerConnection: 1
        http1MaxPendingRequests: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 10s
      baseEjectionTime: 1m
      maxEjectionPercent: 100
    tls:
      mode: "ISTIO_MUTUAL"
```
- Service entry in .yaml (for only cluster 1)
```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: echo
spec:
  hosts:
  - echo.istio.svc.cluster.local
  ports:
  - name: http
    number: 80
    protocol: http
  resolution: STATIC
  location: MESH_INTERNAL
  endpoints:
  - address: 202.28.193.115 #point to the cluster2's gateway 
    locality: us-west1/us-west1-b
    ports:
      http: 15443 #tls protocol
```



## How it work
<h3 align="center"><img width="70%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/flow_chart.png" /></h3>

## Server's Details
<h3>Cluster 1</h3><br>
Istio-Gateway's IP: 202.28.193.112<br>
<h4>Visualization: <a href="http://202.28.193.111/kiali/console/graph/namespaces/?edges=noEdgeLabels&graphType=versionedApp&namespaces=default%2Cistio-system&unusedNodes=false&operationNodes=true&injectServiceNodes=true&duration=60&refresh=10000&layout=dagre">Click Here</a></h4>
<br>
<br>
<h3>Cluster 2</h3><br>
Istio-Gateway's IP: 202.28.193.115<br>
<h4>Visualization: <a href="http://202.28.193.109/kiali/console/graph/namespaces/?edges=noEdgeLabels&graphType=versionedApp&namespaces=default%2Cistio-system&unusedNodes=false&operationNodes=true&injectServiceNodes=true&duration=60&refresh=10000&layout=dagre">Click Here</a></h4>

## Let Start to Demonstrate
<h3>Scenario 1: cluster1 still be healthy</h3><br>
<h3 align="center"><img width="70%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/cluster_ok.png" /></h3>

<h3>Scenario 2: cluster2 not longer be healthy</h3><br>
<h3 align="center"><img width="70%" src="/Presentation_program/5_Experimental_Study_of_Kubernetes/picture/cluster_nok.png" /></h3>
