# project-altus-gke
Project Altus is a real-world implementation of the "Divide" architectural strategy designed to validate a distributed batch processing engine. The specific goal was to execute 10,000 concurrent risk calculations (financial simulations) in under 20 minutes.

Project Altus: The Global Batch Engine
Executive Summary:
Project Altus is a real-world implementation of the "Divide" architectural strategy designed to validate a distributed batch processing engine. The specific goal was to execute 10,000 concurrent risk calculations (financial simulations) in under 20 minutes.


Technical Architecture (Hub-and-Spoke Fleet):
To overcome physical barriers like IP exhaustion and API Server saturation (the "Thundering Herd"), the infrastructure was divided into a 3-cluster GKE Fleet:


Cluster-Hub (us-east1): A stable GKE Autopilot cluster acting as the "Brain." It hosts the Job Queue (Redis) and the Results Dashboard (Web App).


Cluster-Workers (us-central1 & europe-west1): Two disposable GKE Standard clusters acting as the "Muscle." These clusters utilize Spot VMs to execute the heavy compute load.


Key Technologies:


Cloud Service Mesh: Enabled the 10,000 worker pods to connect securely to the Hub's Job Queue using internal DNS (queue.prod.svc.cluster.local) across different regions without requiring complex VPN tunnels.



Config Sync: Used to push the Kubernetes Job manifests to the worker clusters simultaneously, ensuring unified governance.



Spot VMs: Leveraged in the worker clusters to handle the burst workload.

Outcomes:


Control Plane Isolation: The primary "Hub" cluster remained stable because the massive scheduling load of 10,000 pods was offloaded to the disposable "Worker" control planes.


Cost Efficiency: Shifting the burst workload to Spot VMs in cheaper regions resulted in a 70% cost reduction compared to running on reserved instances.
