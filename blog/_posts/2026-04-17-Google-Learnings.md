---
title: "Things I learned about GCP Networking: Part 1"
description: "Coming from an Azure and AWS background, I thought GCP networking would be more of the same. I was wrong. Here is the 'culture shock' guide for Cloud Architects."
author: "Jason Phillips"
date: 2026-01-29
categories: [GCP, Infrastructure as Code, DevOps, Networking]
tags: [GCP, Google Cloud, Terraform, Hybrid Cloud, Networking]
image: /assets/img/google_learn.png
layout: post
---

# TLDR #
Recently, I was pulled into several projects focused on setting up hybrid connectivity and ensuring corporate boundary controls (such as "deny all" inbound to private networks). 

I felt pretty confident with my skills in Azure and AWS. I assumed GCP would be similar, but being wrong would be an understatement. GCP is a whole different animal. Not in a bad way, but there are several differentiating factors that should be called out if you ever find yourself in the middle of a hybrid network implementation on GCP.

# Virtual Networks are Global
Unlike Azure and AWS, where virtual networks (VNets/VPCs) are regional constructs, GCP's Software Defined Network (SDN) enables network connectivity in a **Global** space. In GCP, a VPC is not tied to a specific region. Alternatively, **subnets** are regional constructs. This makes global load balancing and cross-region communication much simpler than the peering or transit gateway meshes we are used to in other clouds.



# Private Endpoints / PSC
Another obvious difference is the private endpoint service function. In Azure, Private Endpoints are attached to PaaS services or internal load balancer services (Private Link Scope) to allow for private connection to resources on the virtual network or from on-prem in a hybrid network.

In GCP, those solutions typically take the form of **PSC (Private Service Connect)** or **Private Google Access**. These allow you to reach Google APIs or hosted services using internal IP addresses, ensuring traffic never leaves the Google network.

# Security Groups and Firewalls
In GCP, you won't see the term "Security Group." Google offers firewalls applied to the VPC. They also offer a concept called **VPC Service Controls (VPC-SC)**. These actually provide a security perimeter around a network, a project, or even a service account—something unique I haven't seen packaged quite like this with other providers. 

While Entra ID in Microsoft has ways to enable similar controls with Conditional Access policies or Workload Identity, GCP puts this right in the middle as a core networking service to prevent data exfiltration.

# Interconnects and Cloud Routers
**Cloud Interconnect** is the most straightforward service and "clicked" with me right away. It provides a dedicated connection from a co-lo provider (Equinix, Megaport, etc.) to the cloud, similar to DirectConnect or ExpressRoute.

However, **Cloud Routers** function a bit differently than a Virtual Network Gateway or Transit Gateway. You must explicitly configure them to advertise certain routes for private endpoints (PSC) and DNS forwarding for resources back to on-prem. This requires a bit more manual involvement and BGP knowledge than I’ve found necessary in other clouds.

# Logging
Logging can be configured for the VPC or the individual subnet. It works as I would expect, similar to AWS VPC Flow Logs or CloudWatch, providing the telemetry needed for auditing and troubleshooting hybrid traffic.

# Conclusion
Transitioning to GCP Networking requires unlearning the "regional" mindset that Azure and AWS bake into your brain. The global nature of the VPC is a massive advantage, but it introduces new complexities in how you handle routing and security perimeters like VPC-SC. 

Understanding these core differences is the first step toward a successful hybrid deployment. In **Part 2**, we will dive deeper into **Identity-Aware Proxy (IAP)** and how to manage these resources at scale using **Terraform**.
