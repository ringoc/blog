---
title: "Azure Bastion on Demand"
date: 2021-07-09T23:30:35+10:00
draft: false
description : "A smart and cost effective way to use Azure Bastion"
slug : "azure-bastion-on-demand"
authors : ["ringoc"]
tags : ["azure","bastion","logicapp", "severless","cloud"]
categories : ["cloud"]
externalLink : ""
series : []
disableComments: false
---
## Background
Internet opens up tons of possibility, but at the same time your properties can become vulunaerable for security breach. It is important for everyone to maintain highest level of security for their properties. 

In the Azure cloud environment, accessing your resources e.g. VM through SSH/RDP means you have to expose your port `22` or `3389` to the internet. One of the nice way is to use Azure Bastion. 

## What is Azure Bastion ?
> Azure Bastion is a new fully platform-managed PaaS service you provision inside your virtual network. It provides secure and seamless RDP/SSH connectivity to your VMs directly in the Azure portal over SSL. When you connect via Azure Bastion, your virtual machines do not need a public IP address. 

# Challenge
However, the cost of Azure Bastion is not cheap. It added up to near $200 Australian Dollar per months not counting the outgoing traffic. 
![Azure Bastion Pricing](../../images/bastion/azure-bastion-pricing.png) <!-- .element height="50%" width="50%" -->
Of course one can delete the Azure Bastion at the end of the day. And keep spinning up a new one the next day. After a few times, you may feel sick of it at least for me. I need to think of a way to automcate this process. 

## Solution
I am going to show you how to setup Azure Bastion on demand with Serverless on Azure Portal

Here are the steps: 

### Create Login App for `createBastionByEmail` 

![Create Bastion by Email](../../images/bastion/create-bastion-by-email-email.png)

![Create Bastion by Email](../../images/bastion/create-bastion-by-email-http.png)

![Create Bastion by Email](../../images/bastion/create-bastion-by-email-output.png)
user@example.com


### 2. Create Login App for `deleteBastionBySchedule` 

![Create Bastion by Email](../../images/bastion/delete-bastion-1.png)
![Create Bastion by Email](../../images/bastion/delete-bastion-2.png)
![Create Bastion by Email](../../images/bastion/delete-bastion-3.png)
![Create Bastion by Email](../../images/bastion/delete-bastion-4.png)
![Create Bastion by Email](../../images/bastion/delete-bastion-5.png)