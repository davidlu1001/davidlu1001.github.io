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

## Compute Savings Plans
Commit to and pay for these in the same way as EC2 Instance Savings Plans, but can only get discounts equal to those offered by Convertible Reserved Instances.

The big selling point for Compute Savings Plans is that the discounts are automatically applied to EC2 instances of any family, size, AZ, region, OS or tenancy, and also apply to Fargate and Lambda usage.

# FAQ
> Can I continue to purchase EC2 RIs?
>
> Yes. You can continue purchasing RIs to maintain compatibility with your existing cost management processes, and your RIs will work along-side Savings Plans to reduce your overall bill. However as your RIs expire we encourage you to sign up for Savings Plans as they offer the same savings as RIs, but with additional flexibility.

AWS will always prioritise applying the Reservation first, then fill in any gaps with a Savings Plan. Still need to actively manage your Reservations, to ensure you get the full value out of your commitment.

# References
- https://aws.amazon.com/blogs/aws/new-savings-plans-for-aws-compute-services/

- https://aws.amazon.com/savingsplans/faq/