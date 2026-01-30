---
title: "ExcelAsCode: Because Your Infrastructure Team Already Speaks Excel"
description: "As organizations adopt Infrastructure as Code best practices, infrastructure teams face a dilemma: they're not developers, but they need to deploy infrastructure with Terraform, ARM, or Bicep. What if the solution was already sitting on everyone's desktop?"
author: "Jason Phillips"
date: 2026-01-29
categories: [Azure, Infrastructure as Code, DevOps, Automation]
tags: [Azure, Terraform, Excel, IaC, VBA, Cloud Automation]
image: /assets/img/excelascode.png
layout: post
---

# ExcelAsCode: When Excel Becomes Your Infrastructure as Code Engine

![Server Team vs DevOps Team Meme](/assets/images/excelascode-meme.png)

## The Infrastructure as Code Dilemma

Picture this: It's Friday afternoon. Your infrastructure team needs to deploy 50 new VMs across multiple Azure regions by Monday. The business requirements are crystal clear, meticulously documented in—you guessed it—an Excel spreadsheet.

But there's a problem.

Your DevOps team insists on "proper Infrastructure as Code practices." They want Terraform configurations. ARM templates. Bicep files. Version control. GitOps workflows. All the modern best practices that make perfect sense... if you're a developer.

But your infrastructure team isn't a room full of developers. They're seasoned IT professionals who've been managing infrastructure for decades. They know networking, storage, compute, security. They can design complex multi-tier architectures in their sleep.

They just don't speak HashiCorp Configuration Language.

## The Shadow IT Reality

Here's what actually happens in most organizations:

1. Infrastructure team plans deployments in Excel (because it works)
2. DevOps team spends days translating requirements into Terraform
3. Changes come in, Excel gets updated
4. DevOps team re-writes Terraform again
5. Rinse and repeat, creating bottlenecks and frustration on both sides

Meanwhile, the Excel spreadsheet—the one with the actual requirements—sits there, perfectly organized, with dropdowns for regions and VM sizes, formulas calculating costs, conditional formatting highlighting issues. It's beautiful. It's functional. It's just... not "Infrastructure as Code."

Or is it?

## What If Excel WAS the Infrastructure as Code?

That's the question that led to ExcelAsCode.

Not "how do we force infrastructure teams to learn Terraform," but "what if we met them where they already are?"

## Introducing ExcelAsCode

ExcelAsCode is an open-source Excel template that generates Infrastructure as Code directly from spreadsheets. No Python scripts to maintain. No complex workflows. Just Excel, doing what Excel does best, but now with superpowers.

### How It Works

**1. Open the Template**
Download the Excel file, enable macros, and you're presented with a familiar spreadsheet interface.

**2. Authenticate with Azure**
Click the "Authenticate" button. ExcelAsCode uses OAuth 2.0 device code flow to securely connect to your Azure tenant. No stored credentials, no service principals to manage (unless you want to).

**3. Dynamic Dropdowns Appear**
This is where the magic starts. The template makes live API calls to Azure and populates dropdowns with:
- Your actual Azure regions
- Available VM SKUs in each region
- Current OS images (Windows Server, Ubuntu, RHEL, etc.)
- Your resource groups
- Virtual networks and subnets
- Network security groups

No more typos. No more guessing at resource names. No more checking the Azure Portal to verify what's available.

**4. Fill in Your Requirements**
Use Excel like you always have:
- One row per VM
- Columns for VM name, size, region, OS, disk size
- Add your own columns for cost center, owner, purpose
- Use Excel formulas to calculate costs
- Apply conditional formatting for validation

**5. Generate Infrastructure as Code**
Click "Generate Terraform" or "Generate ARM Template." ExcelAsCode:
- Validates your inputs
- Generates properly formatted IaC files
- Creates modular, production-ready code
- Includes variables, outputs, and documentation
- Saves files ready for git commit

**6. Deploy or Review**
You can either:
- **Direct Deploy**: Click "Deploy to Azure" and ExcelAsCode provisions everything
- **Export for Review**: Save the Terraform/ARM files, commit to git, run through your CI/CD pipeline

## Real-World Capabilities

### Bulk VM Provisioning
Need to deploy 100 VMs across 5 regions? Fill in the spreadsheet once, generate once, deploy once. What used to take days now takes minutes.

### Disaster Recovery Environments
Your production environment is documented in Excel? Clone the template, change the region dropdown to your DR region, regenerate. DR environment ready to go.

### Dev/Test Environment Spinning
Developers need isolated environments? Give them the template. They fill it in, submit for approval, infrastructure team clicks deploy. Self-service infrastructure without sacrificing governance.

### Cost Estimation Before Deployment
Excel's formulas work perfectly with Azure pricing. Calculate costs before deploying a single resource. Compare configurations. Optimize before spending.

### Shadow IT Governance
That Excel file the business team has been using to track their "testing VMs"? Import it into ExcelAsCode, see what they actually need, provision it properly with tagging, monitoring, and compliance.

## Technical Deep Dive

### Architecture
