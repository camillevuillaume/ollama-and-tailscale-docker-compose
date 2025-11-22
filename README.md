# ollama-and-tailscale-docker-compose
Docker compose yaml for Ollama and Tailscale.
It has been tested locally on a Proxmox VM without GPU acceleration. Next I will try it out on EC2 instances with GPUs. My goal is to integrate it in some of my local workflows, for instance with N8N to accelerate LLMs.

## AWS EC2 Preparation
Some EC2 instances do have GPUs. For small to medium models in particular, the following instances are attractive.

g4ad.xlarge 4 vCPUs x86_64 16GB RAM Radeon Pro V520 8GB VRAM 0.38 USD/h (US east 1)
g5g.xlarge 4 vCPUs arm64 8GB RAM NVIDIA T4g 16GB VRAM 0.42 USD/h (US east 1)
g4dn.xlarge 4 vCPUs x86_64 16GB RAM NVIDIA T4 16GB VRAM 0.53 USD/h (US east 1)
g6.xlarge 4 vCPUs x86_64 16GB RAM NVIDIA L4 22.4GB VRAM 0.80 USD/h (US east 1)
g5.xlarge 4 vCPUs x86_64 16GB RAM NVIDIA A10G 22.4GB VRAM 1.01 USD/h (US east 1)
g6e.xlarge 4 vCPUs x86_64 32GB RAM NVIDIA L40S 44.7GB VRAM 1.86 USD/h (US east 1)

To get access to these instances, it is necessary to increase the vCPU quotas on G instances, which is zero by default. To do that, go to "Service Quotas".
https://us-east-1.console.aws.amazon.com/servicequotas/home?region=us-east-1
In "Manage quotas", select Amazon Elastic Compute Cloud.

Note that the cheapest option is based on AMD GPUs, but standard Linux images provided by Amazon do not include the amdgpu driver.

