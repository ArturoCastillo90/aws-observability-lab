# AWS Observability Lab: EC2 + CloudWatch Agent + Grafana

![AWS](https://img.shields.io/badge/AWS-CloudWatch-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

## Overview

This project implements an **end-to-end observability lab** using AWS services and an EC2 instance configured with the CloudWatch Agent and Grafana. The goal is to demonstrate how to collect, publish, visualize, and monitor system metrics using an architecture similar to production-grade monitoring stacks.

### Key Features

- Custom VPC with public subnet
- EC2 Ubuntu instance
- Installation and configuration of the CloudWatch Agent
- IAM Role for secure metric publishing
- Grafana connected to CloudWatch as data source
- Custom dashboards visualizing CPU, memory, disk, and network metrics
- Stress testing to validate metric ingestion and visualization

## Architecture

```
                 ┌────────────────────────────┐
                 │        AWS CloudWatch       │
                 │  - Metrics (AWS/EC2)        │
                 │  - Metrics (CWAgent)        │
                 └──────────────┬─────────────┘
                                │
                                │ CloudWatch API (IAM Role)
                                │
        ┌───────────────────────┴────────────────────────┐
        │                                                │
┌───────▼──────────┐                             ┌───────▼──────────┐
│ CloudWatch Agent │                             │ Grafana on EC2   │
│ (CPU, RAM, Disk) │                             │ CloudWatch DS    │
│ Publishes to CW   │                             │ Dashboards        │
└───────┬──────────┘                             └────────┬─────────┘
        │                                                │
        └────────────────────────────────────────────────┘
                     EC2 Instance in Public Subnet
```

### AWS Components

- **VPC** with internet gateway and public subnet
- **EC2 instance** with public IP
- **IAM Role** providing CloudWatch write and read permissions
- **CloudWatch Metrics** and Logs services

## Requirements

- AWS Account
- IAM administrative permissions
- EC2 Ubuntu instance
- Inbound access to ports:
  - `22` (SSH)
  - `3000` (Grafana)
- Optional: SSH tunneling for access behind corporate networks

## Setup Guide

### Step 1: Create VPC and Subnets

1. Create a VPC with CIDR `10.0.0.0/16`
2. Create a public subnet (example: `10.0.1.0/24`)
3. Attach an Internet Gateway
4. Create a route table allowing `0.0.0.0/0` → IGW
5. Associate the public subnet with the route table

### Step 2: Launch EC2 Instance

- **AMI**: Ubuntu 22.04
- **Instance Type**: t2.micro or t3.micro
- **Network**: Assign public IP
- **IAM Role**: Attach the role described below
- **Security Group**: Allow ports 22, 3000

### Step 3: IAM Role Configuration

Create an IAM role for EC2 with the following managed policies:

- `CloudWatchAgentServerPolicy`
- `CloudWatchReadOnlyAccess`

Attach this role to your EC2 instance.

## Installation

### Installing Grafana on EC2

```bash
sudo apt-get update
sudo apt-get install -y software-properties-common

# Add Grafana GPG key
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://packages.grafana.com/gpg.key | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add Grafana repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list > /dev/null

# Install and start Grafana
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access Grafana at: `http://<EC2-PUBLIC-IP>:3000`
- Default credentials: `admin` / `admin`

### Installing and Configuring CloudWatch Agent

**Download and install the agent:**

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

**Create the configuration file:**

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Paste the following configuration:**

```json
{
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_active"
        ],
        "metrics_collection_interval": 60
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "netstat": {
        "measurement": [
          "tcp_established",
          "tcp_time_wait"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

**Start the CloudWatch Agent:**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Verify the agent is running:**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a query \
  -m ec2 \
  -s
```

## Configuring Grafana

1. Access Grafana at `http://<EC2-PUBLIC-IP>:3000`
2. Go to **Configuration** → **Data Sources**
3. Add **CloudWatch** as a data source
4. Select **Authentication Provider**: `AWS SDK Default`
5. Set **Default Region**: Your EC2 region (e.g., `us-east-1`)
6. Click **Save & Test**

### Creating Dashboards

Create dashboards to visualize:
- CPU usage (`cpu_usage_active`)
- Memory usage (`mem_used_percent`)
- Disk usage (`disk_used_percent`)
- Network connections (`tcp_established`, `tcp_time_wait`)

## Testing with Stress

Install stress tool to generate load:

```bash
sudo apt-get install -y stress
```

Run a CPU stress test:

```bash
stress --cpu 4 --timeout 300s
```

Monitor the metrics in your Grafana dashboard in real-time.

## 📁 Project Structure

```
aws-observability-lab/
│
├── cloudwatch-agent/
│   └── amazon-cloudwatch-agent.json
│
├── grafana/
│   └── dashboards/
│
├── ec2-setup/
│   └── commands.md
│
├── vpc-diagram/
│   └── architecture.png
│
└── README.md
```

## Troubleshooting

### CloudWatch Agent not publishing metrics

```bash
# Check agent status
sudo systemctl status amazon-cloudwatch-agent

# View agent logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Grafana cannot connect to CloudWatch

- Verify IAM role is attached to EC2
- Check IAM permissions include `CloudWatchReadOnlyAccess`
- Verify region is correctly configured

## Resources

- [CloudWatch Agent Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [Grafana CloudWatch Data Source](https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/)
- [AWS CloudWatch Metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html)

## License

This project is licensed under the MIT License.

## Author

Your Name - [GitHub Profile](https://github.com/ArturoCastillo90)

---

If you found this project helpful, please consider giving it a star!

