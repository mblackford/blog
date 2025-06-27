---
title: "Using Fargate Spot in Production Without Getting Burned (and Still Save ~60%)"
date: 2025-03-08
tags: [aws, devops, cost-optimisation]
summary: >
A guide to safely using AWS ECS Fargate Spot for real workloads without compromising reliability.
---

# Using Fargate Spot in Production Without Getting Burned (and Still Save ~60%)

In a recent consulting project, I tackled an interesting challenge that combined my passion for cloud cost optimization with the practicalities of AWS infrastructure. I devised a strategy that allowed me to reduce compute costs by around **60%** using **AWS Fargate Spot** instances, all while maintaining reliability. Here's how I did it.

---

## Why Fargate Spot?

Fargate Spot offers a fantastic opportunity to optimise compute costs by taking advantage of **spare AWS capacity** at up to \~70% off the standard rate. The catch? Spot capacity can be reclaimed by AWS at any time with very short notice.

You might expect that the [capacity provider strategy](https://docs.aws.amazon.com/AmazonECS/latest/userguide/cluster-capacity-providers.html) would automatically rebalance workloads between Fargate and Spot. But it doesn't. If AWS takes your Spot capacity, the service won’t automatically launch new tasks on regular Fargate to compensate — it just drops capacity.

> "I thought ECS would fallback to regular Fargate when Spot fails, but nope — it just silently drops tasks. Had to learn the hard way."
> — [https://www.reddit.com/r/aws/comments/147hwx0/ecs_capacity_providers_fargate_and_fargate_spot/](https://www.reddit.com/r/aws/comments/147hwx0/ecs_capacity_providers_fargate_and_fargate_spot/)

> *"Fargate doesn’t replace Spot capacity with on-demand capacity."*
> — [https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html)

---

## The Naive Approach

One common way to work around this is to use **CloudWatch alarms and Lambda functions** to detect low task counts and then dynamically shift the capacity provider strategy.

> "I've built a Lambda to detect task count drops and flip the service to regular Fargate. It works, but testing it is rough."
> — [https://gist.github.com/ahmadnassri/1be7a3910c7cf56e65d25b377731e3f1](https://gist.github.com/ahmadnassri/1be7a3910c7cf56e65d25b377731e3f1)

Testing this reliably is difficult. Spot interruptions are inherently **unpredictable and infrequent**, making it tough to validate if your failover logic really works. Even AWS acknowledges this limitation in their documentation:

> *"Fargate Spot runs on spare capacity and there might be no availability at times, which makes testing fallback solutions difficult."*
> — [https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html)

---

## The Dual-Service Solution

To overcome these limitations, I devised a more robust solution: **run two separate ECS Fargate services behind the same load balancer**.

* One service runs entirely on **Fargate Spot**
* The other runs on **regular Fargate**

By tuning the **auto-scaling policies**, the Spot service handles the majority of traffic. If Spot capacity is taken away, the regular Fargate service **remains as a warm backup** and scales up to handle the load.

This design doesn’t rely on reacting to events — it's **proactively resilient**.

---

## How It Works

### 1. Shared Task Definition

Both services share the same `TaskDefinition`:

```yaml
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    RequiresCompatibilities: [FARGATE]
    Cpu: !Ref Cpu
    Memory: !Ref Memory
    NetworkMode: awsvpc
    ...
```

### 2. Spot-Only Service

```yaml
SpotService:
  Type: AWS::ECS::Service
  Properties:
    CapacityProviderStrategy:
      - CapacityProvider: FARGATE_SPOT
        Weight: 1
    DesiredCount: !Ref MinCapacity
```

### 3. On-Demand Backup Service

```yaml
OnDemandService:
  Type: AWS::ECS::Service
  Properties:
    CapacityProviderStrategy:
      - CapacityProvider: FARGATE
        Weight: 1
    DesiredCount: 1
```

### 4. Smart Scaling

Each service gets its own `ScalableTarget` and `ScalingPolicy`. Here's an example for CPU scaling:

```yaml
SpotCpuScaling:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    ScalingTargetId: !Ref SpotScalingTarget
    TargetTrackingScalingPolicyConfiguration:
      TargetValue: 50.0  # More aggressive
```

```yaml
FargateCpuScaling:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    ScalingTargetId: !Ref FargateScalingTarget
    TargetTrackingScalingPolicyConfiguration:
      TargetValue: 80.0  # Higher threshold
```

The lower threshold on Spot ensures it handles load first, while the Fargate service only kicks in when needed.

---

## Results

This setup has now been running for **6+ months** in a production environment. The result?

* **\~60% cost savings** on compute
* **No downtime** due to Spot interruptions
* **No Lambda glue code** or runtime logic
* Easily testable, reproducible deployment via CloudFormation

It’s simple, robust, and safe — and you can deploy it in any ECS cluster with minimal changes.

---

## Try It Yourself

You can adopt this strategy by copying and adapting [this CloudFormation template](https://github.com/mblackford/blog/blob/main/code/2025-03-08-fargate-spot-in-production/production-ready-fargate-spot-template.yaml). It uses:

* Dual ECS services with shared task definition
* Tuned CPU auto-scaling policies for service balancing
* Log configuration to combine logs from both services 
* Private subnet network configuration

All production-ready and reusable.

---

## Final Thoughts

Fargate Spot is underused because of its **unpredictability**, but this architecture shows that with a bit of redundancy and the right scaling strategy, it’s possible to build **cost-optimised and resilient workloads** without complexity.

Got questions or want to share how you're using Spot? Drop me a note!

- [LinkedIn](https://www.linkedin.com/in/matthew-blackford/)

---

> © 2025 Matt Blackford.  
> Text content is licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/).  
> Code snippets are licensed under the [MIT License](https://opensource.org/license/mit/).
