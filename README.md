# ollama-and-tailscale-docker-compose
Docker compose yaml for Ollama and Tailscale.
It has been tested locally on a Proxmox VM without GPU acceleration. Next I will try it out on EC2 instances with GPUs. My goal is to integrate it in some of my local workflows, for instance with N8N to accelerate LLMs.

## AWS EC2 Preparation
Some EC2 instances do have GPUs. For small to medium models in particular, the following instances are attractive. Note that prices may vary by region. The prices below are for the us-east-1 region as of November 2025.

| Instance Type | vCPUs | Architecture  | RAM   | GPU               | VRAM  | On-Demand Price   |
|---------------|-------|---------------|-------|-------------------|-------|-------------------|
| g4ad.xlarge   | 4     | x86_64        | 16GB  | Radeon Pro V520   | 8GB   | 0.38 USD/h        |
| g5g.xlarge    | 4     | arm64         | 8GB   | NVIDIA T4g        | 16GB  | 0.42 USD/h        |
| g4dn.xlarge   | 4     | x86_64        | 16GB  | NVIDIA T4         | 16GB  | 0.53 USD/h        | 
| g6.xlarge     | 4     | x86_64        | 16GB  | NVIDIA L4         | 22.4GB| 0.80 USD/h        | 
| g5.xlarge     | 4     | x86_64        | 16GB  | NVIDIA A10G       | 22.4GB| 1.01 USD/h        |
| g6e.xlarge    | 4     | x86_64        | 32GB  | NVIDIA L40S       | 44.7GB| 1.86 USD/h        |

To get access to these instances, it is necessary to increase the vCPU quotas on G instances, which is zero by default. To do that, go to [Service Quotas](https://us-east-1.console.aws.amazon.com/servicequotas/home?region=us-east-1).
In "Manage quotas", select Amazon Elastic Compute Cloud (Amazon EC2). Then search for "All G and VT Spot instance Requests" as well as "All G and VT On-Demand Instance Requests" and request a quota increase, for instance to 8 vCPUs each. The approval may take a few hours.

Note that the cheapest option is based on AMD GPUs, but standard Linux images provided by Amazon do not include the amdgpu driver.

