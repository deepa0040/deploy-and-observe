# EC2 Application Deployment Strategies – DevOps Notes

## 1. Understanding EC2 Application Deployment

Application deployment means releasing a new application version or updating an existing application on an EC2 instance.

A typical EC2 application environment contains:

- **EC2:** Hosts and runs the application.
- **Application service:** The actual application process (Java, Python, Node.js, etc.).
- **Systemd:** Manages application services and processes.
- **Monit:** Monitors application processes and can restart them when configured.
- **CloudWatch:** Collects infrastructure metrics, logs and alarms.
- **ALB:** Distributes application traffic across multiple targets (if configured).
- **Auto Scaling Group (ASG):** Maintains the desired number of EC2 instances.
- **CI/CD pipeline:** Automates build, testing and deployment.

### Example architecture

```text
Developer
    |
    v
GitHub Repository
    |
    v
CI/CD Pipeline
    |
    v
Build Application
    |
    v
Upload Artifact
    |
    v
EC2 Instance
    |
    +---- Application Service
    |
    +---- Systemd
    |
    +---- Monit
    |
    +---- CloudWatch Agent
```

---

## 2. Deployment Strategies

A deployment strategy defines how a new application version is released and how the existing version is replaced.

### A. Recreate Deployment

**Definition:** Stop the existing application version completely, deploy the new version, and start the application again.

Example:

```text
Before deployment:
Application V1 - Running

During deployment:
Application V1 - Stopped
Application V2 - Deploying

After deployment:
Application V2 - Running
```

**Deployment steps:**

1. Download the new application artifact.
2. Stop the existing application service.
3. Back up the previous application version.
4. Replace the application files.
5. Start the new application version.
6. Perform health checks.
7. Roll back if the new version fails.

**Use cases:**

- Development and testing environments.
- Single-instance applications.
- Legacy applications that cannot run multiple versions simultaneously.
- Applications requiring incompatible dependencies.
- Applications where downtime is acceptable.
- Applications using exclusive resources.

**Advantages:**
- Simple implementation.
- No requirement for additional EC2 instances.
- Lower infrastructure cost.

**Disadvantages:**
- Application downtime during deployment.
- Rollback requires restoring the previous version.
- Deployment failure can leave the application unavailable.

### B. Rolling Deployment

**Definition:** Gradually replace old application instances with new application instances.

Example: Three EC2 instances are running V1.

```text
Before:
EC2-1 -> V1
EC2-2 -> V1
EC2-3 -> V1

During:
EC2-1 -> V2
EC2-2 -> V1
EC2-3 -> V1

Next:
EC2-1 -> V2
EC2-2 -> V2
EC2-3 -> V1

After:
EC2-1 -> V2
EC2-2 -> V2
EC2-3 -> V2
```

**Use cases:**

- Production applications with multiple instances.
- Applications requiring high availability.
- Applications where old and new versions can coexist temporarily.

**Advantages:**
- Minimizes downtime.
- No need to replace all instances simultaneously.
- Gradual rollout.

**Disadvantages:**
- Old and new versions temporarily coexist.
- Requires compatibility between application versions.
- Rollback can be more complicated.

### C. Blue-Green Deployment

**Definition:** Maintain two separate environments: Blue (current version) and Green (new version).

```text
             Load Balancer
                   |
            Traffic Switch
              /       \
             /         \
        Blue (V1)    Green (V2)
        Current       New
```

**Deployment steps:**

1. Blue environment runs V1.
2. Deploy V2 to Green.
3. Test the Green environment.
4. Switch production traffic to Green.
5. Monitor the new version.
6. Keep Blue temporarily for rollback.

**Use cases:**

- Business-critical applications.
- Applications requiring minimal downtime.
- Applications requiring quick rollback.
- Major application releases.

**Advantages:**
- Quick traffic switching.
- Easy rollback by switching traffic back.
- New version can be tested before production traffic is moved.

**Disadvantages:**
- Additional infrastructure cost.
- Database compatibility must be considered.
- Requires traffic routing and environment management.

### D. Canary Deployment

**Definition:** Release a new application version to a small percentage of users before gradually increasing traffic.

Example:

```text
Initial:
V1 -> 100%

Canary:
V1 -> 90%
V2 -> 10%

Next:
V1 -> 50%
V2 -> 50%

Final:
V2 -> 100%
```

**Use cases:**

- Large-scale applications.
- Customer-facing applications.
- Applications where production behaviour needs validation.
- High-risk releases.

**Advantages:**
- Limits the impact of deployment failures.
- Real-user validation.
- Gradual traffic migration.

**Disadvantages:**
- Requires traffic routing.
- More monitoring and deployment complexity.
- Requires compatible application and database changes.

---

## 3. Single EC2 Instance Deployment

### Scenario

An application runs on one EC2 instance.

Infrastructure:

```text
EC2 Instance
    |
    +---- Application V1
    |
    +---- Systemd
    |
    +---- Monit
```

There are no additional instances, replicas or load balancers.

### Which deployment strategy is suitable?

Recreate-style in-place deployment is generally suitable.

**Deployment process:**

1. Download the new artifact.
2. Validate the artifact.
3. Stop the existing application.
4. Back up the existing version.
5. Deploy the new version.
6. Start the application.
7. Check application health.
8. Roll back if required.

**Important:** Application downtime is expected while the service is stopped and restarted.

### Better implementation: Versioned release directories

Instead of directly overwriting application files, maintain separate release directories.

```text
/opt/myapp/
    |
    +---- releases/
    |       |
    |       +---- v1/
    |       +---- v2/
    |
    +---- current -> releases/v2/
```

Benefits:

- Keep multiple application versions.
- Easier rollback.
- Avoid partially overwritten application files.
- Prepare the new version before stopping the old version.

A symlink switch alone does not guarantee zero downtime. The application process may still need to restart.

---

## 4. EC2 Deployment with Auto Scaling Group (ASG)

An Auto Scaling Group manages the number of EC2 instances based on configured capacity and scaling policies.

Important parameters:

| Parameter | Meaning |
|---|---|
| Min capacity | Minimum number of instances ASG should maintain |
| Desired capacity | Target number of instances |
| Max capacity | Maximum number of instances ASG can maintain |

### Scenario 1: ASG with min = 1, desired = 1, max = 1

```text
ASG
Min: 1
Desired: 1
Max: 1

    |
    v

EC2-1 (Application V1)
```

Only one instance can run at a time.

**Option A: In-place application deployment**

- Stop V1.
- Deploy V2 on the same EC2 instance.
- Start V2.
- Perform health checks.

No EC2 replacement is required.

**Option B: EC2 instance replacement**

Using ASG Instance Refresh:

- Replace the old EC2 instance.
- Launch a new EC2 instance using the updated launch template or AMI.
- Initialize the new instance.
- Start the application.
- Perform health checks.

With max capacity = 1, ASG cannot launch an additional instance alongside the existing one.

A terminate-first replacement can cause downtime.

### Scenario 2: ASG with temporary additional capacity

Example:

| Parameter | Value |
|---|---:|
| Min | 1 |
| Desired | 1 |
| Max | 2 |

During deployment, the ASG can temporarily run two instances if the deployment configuration allows it.

```text
Before:
EC2-1 -> V1

During:
EC2-1 -> V1
EC2-2 -> V2

After:
EC2-2 -> V2
EC2-1 -> Terminated
```

This supports a launch-before-terminate replacement approach.

**Important considerations:**

- Configure Instance Refresh minimum and maximum healthy percentages.
- Ensure ASG max capacity allows temporary additional instances.
- Configure EC2 and application health checks.
- Ensure the new instance is ready before terminating the old instance.
- Use an ALB or another suitable traffic-routing mechanism if required.

Simply increasing max capacity does not guarantee zero downtime.

---

## 5. Deployment Strategies vs ASG Instance Refresh

These are related but different concepts.

| Concept | Purpose |
|---|---|
| Recreate deployment | Stop old application version and deploy new version |
| Rolling deployment | Gradually update application instances |
| Blue-Green deployment | Maintain two environments and switch traffic |
| Canary deployment | Release to a small percentage of traffic |
| ASG Instance Refresh | Replace EC2 instances managed by an ASG |

ASG Instance Refresh is an infrastructure replacement mechanism. It is not itself an application deployment strategy.

---

## 6. AWS CodeDeploy Deployment Types

AWS CodeDeploy supports different deployment approaches depending on the compute platform.

### EC2 / On-Premises

**In-place deployment:**

- Deploy application to existing EC2 instances.
- Stop or manage the existing application as configured.
- Install the new application version.
- Start the application.
- Validate deployment.

**Blue-Green deployment:**

- Provision or identify replacement instances.
- Deploy the new version.
- Validate the replacement environment.
- Shift traffic.
- Terminate or retain the old environment according to configuration.

### Important distinction

CodeDeploy's in-place deployment is not automatically identical to every recreate deployment.

The exact behaviour depends on deployment configuration, lifecycle hooks and application scripts.

---

## 7. Monitoring During Deployment

Deployment monitoring is important because a successful artifact deployment does not necessarily mean the application is healthy.

### Monit

Monit is used for process monitoring and management.

Examples:

- Detect whether an application process has stopped.
- Restart a process when configured.
- Monitor process resource usage.
- Execute configured recovery actions.
- Send alerts.

### CloudWatch

CloudWatch is used for infrastructure and application monitoring.

Examples:

- EC2 CPU utilization.
- Memory utilization (with CloudWatch Agent).
- Disk utilization (with CloudWatch Agent).
- Application logs.
- Custom application metrics.
- CloudWatch alarms.

### Health checks

Validate:

- Application process is running.
- Application port is listening.
- Health endpoint returns HTTP 200.
- Database connectivity works.
- Application dependencies are available.
- No unexpected errors appear in logs.

**Important:** A running process does not necessarily mean the application is healthy.

---

## 8. Rollback Strategy

Rollback means restoring the previous stable application version when the new deployment fails.

### Example

```text
Current Version: V1

Deploy V2
    |
    v
Health Check
    |
    +---- Success
    |       |
    |       v
    |   Deployment Complete
    |
    +---- Failure
            |
            v
       Stop V2
            |
            v
       Restore V1
            |
            v
       Start V1
            |
            v
       Verify Health
```

### Rollback considerations

- Keep previous application artifacts.
- Back up application configuration.
- Maintain versioned release directories.
- Validate database compatibility.
- Avoid destructive database migrations without a recovery plan.
- Verify application health after rollback.
- Check Monit and CloudWatch alerts.

---

## 9. Production Deployment Decision Table

| Production scenario | Suggested approach |
|---|---|
| Single EC2, no replicas | Recreate-style in-place deployment |
| Single EC2, downtime not acceptable | Add redundancy or another environment |
| Multiple EC2 instances behind ALB | Rolling deployment |
| Critical application requiring fast rollback | Blue-Green |
| High-risk customer-facing release | Canary |
| ASG with max = 1 | In-place or terminate-first replacement with downtime |
| ASG with additional capacity | Rolling or launch-before-terminate replacement |
| Legacy application with incompatible versions | Recreate during maintenance |
| Scheduled batch processing | Controlled replacement with job coordination |

---

## 10. Important Interview Questions

### Q1. Which deployment strategy do you use for a single EC2 instance?

For a single-instance application without replicas, I would generally use a recreate-style in-place deployment. I would stop the existing application, deploy the new version, start it and perform health checks. I would maintain the previous version for rollback.

### Q2. Can we use rolling deployment with a single EC2 instance?

Traditional rolling deployment requires multiple instances to gradually replace. With a single instance, there is no second instance to serve traffic while the existing one is updated. I would consider in-place deployment or introduce additional capacity.

### Q3. What happens if ASG has min = 1, desired = 1 and max = 1?

ASG maintains one instance. If we perform an in-place application deployment, the EC2 instance remains the same. If we use instance replacement, there cannot be an additional instance running simultaneously under that capacity configuration, so downtime may occur.

### Q4. How can we achieve zero downtime with ASG?

Allow temporary additional capacity, launch a new instance, validate application health, shift traffic to the new instance, and then terminate the old instance. An ALB and correctly configured health checks can help manage traffic.

### Q5. What is the difference between Monit and CloudWatch?

Monit primarily monitors and manages processes on a server. CloudWatch collects infrastructure and application metrics, logs and alarms at the AWS monitoring level.

### Q6. What happens if the application process is running but the application is not responding?

A process-level check alone may not detect this problem. I would implement application-level health checks, verify dependencies, check logs and configure suitable monitoring and alerting.

### Q7. How do you handle deployment failure?

I would stop the failed version, inspect deployment logs, restore the previous stable application version, restart the service and verify application health. I would also investigate the root cause before retrying deployment.

---

## 11. Key Takeaways

1. Deployment strategy determines how a new application version replaces the old version.
2. A single EC2 instance generally uses in-place, recreate-style deployment.
3. Recreate deployment can be repeated for every application release.
4. Rolling deployment requires multiple instances or additional deployment capacity.
5. Blue-Green requires two environments and a traffic-switching mechanism.
6. Canary releases expose a small percentage of traffic to the new version.
7. ASG manages EC2 capacity; it does not automatically choose an application deployment strategy.
8. Min = 1, desired = 1, max = 1 means only one instance can run at a time.
9. Monit monitors and manages processes; it is not a deployment tool.
10. Always plan health checks, monitoring and rollback before production deployment.

**Golden rule:** Choose a deployment strategy based on application availability requirements, infrastructure capacity, version compatibility, deployment risk and rollback requirements.