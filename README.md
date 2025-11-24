# Ollama with Tailscale on AWS EC2

The purpose of this guide is to help you set up Ollama on an AWS EC2 instance with GPU support, and make it accessible securely over the internet using Tailscale. This setup allows you to run large language models on a cloud instance while keeping the communication secure and private.

## AWS EC2 Preparation

### EC2 Instance selection

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

Spot instances can be a good option to reduce costs further, but keep in mind that they can be interrupted by AWS with a 2 minutes warning when the capacity is needed elsewhere. For more information, check the [Spot Instances documentation](https://aws.amazon.com/ec2/spot/). Otherwise, on-demand instances are a good choice for flexibility.

We will use the g4ad.xlarge instance for this guide, which is the cheapest option with a GPU. However, there are some caveats to consider, as AMD GPU drivers are not included in standard Linux images provided by Amazon, and the GPU only has 8GB of VRAM, which might be insufficient for some models.

### EC2 Instance Launch

Go to the [EC2 dashboard](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home:) and click on on "Launch Instance".
- Select and appropriate name for the instance.
- Select Ubuntu (Ubuntu Server 24.04 LTS). Note that there are options with pre-installed NVIDIA drivers, but not for AMD GPUs. Notice the user name (usually "ubuntu").
- Select the instance type (g4ad.xlarge).
- If you do not have a key pair yet, create a new one and download the private key file (.pem). Download it and keep it safe as you will need it to access the instance.
- In "Network settings", make sure to allow SSH (port 22). You can leave it open to the world, since SSH access will be protected by the key pair, but for better security, you can restrict access to your IP address only.
- In "Configure storage", the default 8GB disk is not enough. Increase it to 64GB, or more if you plan to use larger models.
- In "Advanced settings", select "Spot instance" if you want to use spot pricing, which is cheaper but less reliable, and might not be available. You can leave the other settings as default.
Finally, review the instance settings and click on "Launch instance". Once the instance is running, click on it to see its details, including the public IP address.

Now, you can connect to the instance using SSH. Open a terminal and run the following command, replacing `<path-to-your-key.pem>` with the path to your private key file and `<public-ip>` with the public IP address of your instance:
```
ssh -i <path-to-your-key.pem> ubuntu@<public-ip>
```

### AMD GPU driver installation

Note that the cheapest option is based on AMD GPUs, but standard Linux images provided by Amazon do not include the amdgpu driver. To install the AMD GPU driver, we are going to follow the [instructions from AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-amd-driver.html).

First of all, we need to make sure the system is up to date, has the necessary build tools, and has the required kernel modules. Run the following commands:
```
sudo apt-get update --fix-missing && sudo apt-get upgrade -y:
sudo apt install build-essential -y
sudo apt install linux-firmware linux-modules-extra-aws -y
sudo reboot
```
After the system reboots, connect again via SSH and run the following commands to download and install the AMD GPU driver:
```
wget https://repo.radeon.com/amdgpu-install/7.1/ubuntu/jammy/amdgpu-install_7.1.70100-1_all.deb
sudo apt install ./amdgpu-install_7.1.70100-1_all.deb -y
sudo amdgpu-install --usecase=rocmdev
sudo usermod -a -G render $LOGNAME

sudo reboot
```

Check the [AMD GPU driver download page](https://www.amd.com/en/support/download/linux-drivers.html) for the latest version of the drivers.
Note that the installation process may take some time.
After the system reboots, connect again via SSH and verify that the AMD GPU driver is installed correctly by running:
```
rocminfo
```

## Tailscale Setup

The guide follows the [official Tailscale documentation](https://tailscale.com/kb/1282/docker).
- In the Tailscale admin console, go to "Access Controls", click on "Tags" and create a tag, for instance "container".
- Next, in "Setting", go to "Keys" under Personal Setting. Here, create a new key with "Genereate auth key". Make sure to select the tag created before. Then, copy the generated key and save it for later use. Make sure to keep it secret.

Be sure to check the Tailscale documentation and video for more details.

## Docker Compose Setup

First, we need to install Docker and Docker Compose on the EC2 instance. Run the following commands:
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Now that we have everything ready, we can create the compose.yaml file. In the EC2 instance, create a new directory and inside it, create a file named `compose.yaml`. You can copy the contents of the file from this repository.
In addition, we need to create a `.env` file to store environment variables. Create a file named `.env` in the same directory as `compose.yaml` and add the following content, replacing the placeholder with your Tailscale auth key created earlier:
```
TS_AUTH_KEY=tskey-yourgeneratedkeyhere
```
Now, you can start the containers with the following command:
```
sudo docker compose up -d
```
Docker will download the necessary images and start the containers in detached mode.
Now let's check if everything is running correctly. First, from the Tailscale admin console, you should see the new EC2 instance connected to your Tailscale network with the "container" tag. Next, from a browser and a machine connected to the same Tailscale network, you can access the Ollama web interface by navigating to `http://<tailscale-ip>:11434`, replacing `<tailscale-ip>` with the Tailscale IP address of your EC2 instance. You should see the following message: "Ollama is running".

## Using Ollama

From your machine connected to the same Tailscale network, you can use the Ollama CLI to interact with the models hosted on the EC2 instance.
First, set the `OLLAMA_HOST` environment variable to point to the Tailscale IP address of your EC2 instance. For example, in a bash shell, run:
```
export OLLAMA_HOST=http://<tailscale-ip>:11434
```
Now, you can use the Ollama CLI as if it were running locally. For example, to list the available models, run:
```
ollama list
```

