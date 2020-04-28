---
title: AWS Savings Plans Overview
tags:
  - AWS
  - Savings Plan
  - Reserved Instances
  - SRE
  - Cost Optimization
categories:
  - - SRE
  - - AWS
  - [Cost]
abbrlink: 95b1734e
date: 2020-04-28 19:57:35
---

# What are AWS Savings Plans
AWS Savings Plans is launched in November 2019, and allows customers to save up to 72% on Amazon EC2 / AWS Fargate in exchange for making a commitment (on how much they will spend per hour) to a consistent amount of compute usage for a 1 or 3-year term. 

Customers can choose how much they wish to commit to (minimum $0.001 per hour per year) and layer Savings Plans on top of one another.

The major difference between Reserved Instances and AWS Savings Plans are that, rather than committing to a specific instance type in return for a discount, you are committing to a specific spend per hour.

> It’s important to note that at this time, AWS doesn’t allow customers to change their Savings Plans contract once purchased or sell unused discounts in the AWS Marketplace. Once customers commit to a Savings Plan price, they are locked in for the one or three years they committed to. 

There are two types of AWS Savings Plans:

- EC2 Instance Saving Plans: like Standard RIs

- Compute Savings Plans: shares attributes with Convertible RIs with the added bonus that discounts can be applied to the Fargate container service.

These two types provide the choice between maximising financial benefit and sacrificing flexibility or maximising flexibility while benefiting from a smaller discount. 

Here’s a quick comparison of the two types:

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/Gx8Hj0.png)

## EC2 Instance Savings Plans
The main differences between EC2 Instance Saving Plans and Standard RIs is that Savings Plan discounts are applied automatically to any EC2 instance (within the same family and region) regardless of the operating system or tenancy.

At this stage it’s important to note AWS Savings Plans cannot yet be applied to RDS instances, AWS Redshift, or ElastiCache services. Customers using these services will have to continue using Reserved Instances.

## Compute Savings Plans
Commit to and pay for these in the same way as EC2 Instance Savings Plans, but can only get discounts equal to those offered by Convertible Reserved Instances.

The big selling point for Compute Savings Plans is that the discounts are automatically applied to EC2 instances of any family, size, AZ, region, OS or tenancy, and also apply to Fargate and Lambda usage.

# How it works
![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/GhwBkA.jpg)

![](https://raw.githubusercontent.com/davidlu1001/davidlu1001.github.io/hexo/uPic/qkNOIj.jpg)

# Findings
- There is no direct financial benefit of purchasing AWS Savings Plans to replace fully-optimized and effectively-managed RIs when reservations expire. However, it is anticipated many customers will opt for the new discount program due to its increased flexibility and reduced management overhead.

- Similarly to Reservations, unused discounts do not accumulate. Therefore it’s critical that need to keep track of any Savings Plan waste in order to minimize it in the future.

- In order to calculate whether or not layering a Savings Plan on top of an RI is financially viable, it’s important to have complete visibility of utilisation metrics (or accurately forecast demand) to avoid wasted Savings Plans discounts.

- Every hour, AWS will assess the possible discount programs that usage qualifies for, and will apply them in the following order: Standard Reservation, Convertible Reservation, Savings Plan.

So as the minimum commitment is `$0.001` per hour. This means only have to spend $8.76 per year ($0.001 x 24 hours x 365 days) in order to qualify for a discount - which might be safer? And if that works as we expected then can add another savings plans on top of it.

# FAQ
> **Can I continue to purchase EC2 RIs?**
> 
> Yes. You can continue purchasing RIs to maintain compatibility with your existing cost management processes, and your RIs will work along-side Savings Plans to reduce your overall bill. However as your RIs expire we encourage you to sign up for Savings Plans as they offer the same savings as RIs, but with additional flexibility.
> 
> **Do I have to choose between the two types of savings plans?**
> 
> In the same way as customers can purchase both Standard and Convertible RIs, it’s possible to purchase both EC2 Instance Savings Plans and Compute Savings Plans.
> 
> **What happens if I don’t spend my minimum monetary commitment?**
> 
> If you don’t meet the minimum monetary commitment in any given hour, that Savings Plan benefit is forfeited. For example, if you commit to $10 per hour, but only consumes $8 of services in any hour, you’ll lose the remaining $2. However, if you’re using linked or consolidated accounts, it’s likely that the unused Savings Plan commitment would have “floated” to benefit another account.

# References
- [AWS - New Savings Plans for AWS compute services](https://aws.amazon.com/blogs/aws/new-savings-plans-for-aws-compute-services/)

- [AWS Savings Plans - FAQ](https://aws.amazon.com/savingsplans/faq/)

- [AWS re:Invent 2019: Dive deep on how to save with AWS Savings Plans](https://www.youtube.com/watch?v=uQ9ry-9uUvo&t=1829s)