# Recreate Deployment Strategy
In recreate deployment, we stop all existing application instances before deploying the new version. Once the old version is stopped, we deploy and start the new version.

- I would choose this strategy when downtime is acceptable, such as in development or testing environments, or when the old and new application versions cannot run simultaneously because of dependency or configuration incompatibilities.

- For example, if an application requires a major runtime upgrade, we might stop the old application, upgrade the runtime, and deploy the new version.

The **main disadvantage is downtime during deployment**, so I generally wouldn't prefer it for business-critical production applications.


# Recreate Deployment – Real-world examples

- **Runtime upgrade:** Upgrading Java 11 to Java 21 when old and new runtimes cannot coexist safely.
- **Database schema changes:** Application requires a new incompatible database schema.
- **OS upgrade:** Replacing an application environment running Ubuntu 20.04 with Ubuntu 24.04.
- **Dependency conflicts:** New application version requires incompatible libraries or packages.
- **Configuration changes:** Replacing old application configurations with a completely new configuration structure.
- **Single-instance application:** Deploying a new version when only one EC2 instance is available.
- **Batch processing:** Replacing a scheduled job application to prevent old and new versions from executing simultaneously.
- **Exclusive resources:** Applications requiring exclusive access to files, devices or locks.
- **Major application upgrade:** Replacing an old application with a completely redesigned version.

**When to use:** Dev/test environments, maintenance windows, legacy applications, or production workloads where downtime is acceptable.

**Main disadvantage:** Application downtime during deployment.


# Reasons for choosing recreate deployment

Recreate is a deployment strategy that can be used repeatedly whenever you release a new application version.

### Let's understand with a production example

Suppose you have an application running on one EC2 instance.

| Release   | Deployment activity                     |
| --------- | --------------------------------------- |
| Initial   | Deploy V1                               |
| Release 2 | Stop V1 → Deploy V2 → Start application |
| Release 3 | Stop V2 → Deploy V3 → Start application |
| Release 4 | Stop V3 → Deploy V4 → Start application |

Every release follows the same recreate process.

You don't need a new EC2 instance for every deployment.

### When would you repeatedly use recreate?

Real-world examples:

- Internal HR application where a few minutes of downtime is acceptable during every release.
- Development application where developers deploy multiple times daily.
- Legacy application that cannot run two versions simultaneously.
- Single-instance batch processing application where the old process must stop before the new version starts.
- On-premises applications where additional infrastructure for rolling or blue-green deployment is not available.

### Important distinction

There are two separate decisions:

1. **Infrastructure replacement:** Do I need a new EC2 instance, or can I reuse the existing one?
2. **Deployment strategy:** How do I replace the running application version?

Recreate does not require new infrastructure. You can reuse the same EC2 instance.

**Interview takeaway:**

"Recreate deployment is not a one-time deployment approach. We can use it for every application release. During each release, we stop the existing application, deploy the new version on the same or replacement infrastructure, and start it again. We choose this approach when downtime is acceptable and running multiple application versions simultaneously is not required or is unsafe."

# In-Place vs Recreate Deployment

- **In-place deployment** describes *where* the new application version is deployed: on the existing infrastructure.

- **Recreate deployment** describes *how* the old application version is replaced: stop the old version completely, then deploy the new version.

| Deployment Approach | What Happens? | Infrastructure Impact | Downtime |
|---|---|---|---|
| **In-place + Recreate** | Stop V1, remove/replace it with V2, and restart the application on the same EC2 instance. | Existing EC2 is reused. | Yes, usually |
| **In-place + Rolling** | Gradually deploy V2 across existing EC2 instances, updating them in batches. | Existing EC2 instances are reused. | Minimal or none, if capacity is sufficient |
| **Infrastructure Replacement** | Terminate or retire old EC2 instances and provision new EC2 instances with the updated application. | New EC2 instances are created. | Depends on replacement and traffic management |
| **Infrastructure Blue-Green** | Create a separate environment with new EC2 instances, deploy V2, validate it, and switch production traffic from the old environment to the new one. | Two environments temporarily coexist. | Usually minimal or none |

### Key Differences

- **In-place:** Deploy on existing infrastructure.
- **Recreate:** Stop the old application completely before deploying the new version.
- **Rolling:** Update instances gradually in batches.
- **Infrastructure Replacement:** Replace existing EC2 instances with new ones.
- **Blue-Green:** Maintain old and new environments and switch traffic after validation.

# Recreate Deployment Strategy - for Application Explanation

## 1. What is Recreate Deployment?

Recreate is a deployment strategy in which the existing application version is completely stopped before the new version is deployed and started.

**Core principle:**

Stop old version → Deploy new version → Start new version → Validate application.

- The old and new application versions do not run simultaneously.
- The existing infrastructure can be reused.
- Downtime is generally expected during deployment.

---

## 2. Single-Instance Application (One EC2)

### Scenario

An application is running on a single EC2 instance.

**Before deployment:**

| EC2 Instance | Application |
|---|---|
| EC2-1 | V1 (Running) |

### Deployment steps

| Step | Action | Application status |
|---|---|---|
| 1 | Stop Application V1 | Stopped |
| 2 | Back up the existing application | Stopped |
| 3 | Deploy Application V2 | Stopped |
| 4 | Start Application V2 | Running |
| 5 | Perform health checks | Available |

### Architecture flow

```text
Before Deployment:

EC2-1
  └── Application V1 (Running)

During Deployment:

EC2-1
  └── Application (Stopped)
       └── Deploy V2

After Deployment:

EC2-1
  └── Application V2 (Running)
```

### Example

A Java application is running on EC2-1. You need to deploy a new JAR file.

```bash
# Stop old application
sudo systemctl stop myapp

# Back up the existing JAR
sudo cp /opt/myapp/myapp.jar /opt/myapp/myapp-backup.jar

# Deploy new version
sudo cp myapp-v2.jar /opt/myapp/myapp.jar

# Start new application
sudo systemctl start myapp

# Verify application
sudo systemctl status myapp
```

**Result:**
- The same EC2 instance is reused.
- V1 is stopped before V2 starts.
- Users experience downtime during deployment.

---

## 3. Multiple-Instance Application (Three EC2s)

### Scenario

An application is running on three EC2 instances behind an Application Load Balancer (ALB).

**Before deployment:**

| EC2 Instance | Application |
|---|---|
| EC2-1 | V1 (Running) |
| EC2-2 | V1 (Running) |
| EC2-3 | V1 (Running) |

### Deployment steps

| Step | Action | Application status |
|---|---|---|
| 1 | Stop V1 on all three EC2 instances | All stopped |
| 2 | Deploy V2 on all three instances | All stopped |
| 3 | Start V2 on all three instances | Starting |
| 4 | Perform health checks | Available |
| 5 | ALB routes traffic to healthy instances | Available |

### Architecture flow

```text
Before Deployment:

ALB
 ├── EC2-1 → V1 (Running)
 ├── EC2-2 → V1 (Running)
 └── EC2-3 → V1 (Running)

During Deployment:

ALB
 ├── EC2-1 → Application Stopped
 ├── EC2-2 → Application Stopped
 └── EC2-3 → Application Stopped

Deploy V2 on all instances.

After Deployment:

ALB
 ├── EC2-1 → V2 (Running)
 ├── EC2-2 → V2 (Running)
 └── EC2-3 → V2 (Running)
```

**Result:**
- All three instances are reused.
- The old version is stopped everywhere before the new version starts.
- Users experience downtime because no old application instances remain available during deployment.
- ALB health checks prevent traffic from being sent to unhealthy instances, but they do not eliminate downtime when every target is unavailable.

## 4. Multiple-Instances manage by ASG 
If an application is running on multiple EC2 instances managed by an Auto Scaling Group, I would generally prefer rolling deployment for production because it allows us to update instances gradually while maintaining healthy capacity.

For example, if we have three EC2 instances, we can update one instance at a time, validate the new application version, and then proceed to the next instance. The remaining healthy instances continue serving traffic.

I would choose recreate when downtime is acceptable or when the application requires all instances to be stopped before deploying the new version.

---

## 5. Recreate vs Rolling Deployment

Consider an application running on three EC2 instances.

| Comparison | Recreate | Rolling |
|---|---|---|
| Deployment approach | Stop old version everywhere, then deploy new version | Update instances gradually in batches |
| Deployment sequence | Stop all → Deploy all → Start all | Update EC2-1 → EC2-2 → EC2-3 |
| Old and new versions | Do not run simultaneously | Can run simultaneously during deployment |
| Infrastructure | Existing EC2 instances reused | Existing EC2 instances reused |
| Downtime | Usually yes | Can be avoided if sufficient healthy capacity remains |
| Complexity | Relatively simple | Requires capacity and health-check planning |

---

## 6. Key Points to Remember

1. Recreate is an application deployment strategy, not necessarily an infrastructure replacement strategy.
2. It can be used with a single EC2 instance or multiple EC2 instances.
3. The existing infrastructure can be reused.
4. The old application version must stop before the new version starts.
5. Downtime is generally expected.
6. ALB health checks help identify healthy targets but cannot provide availability when all targets are unavailable.
7. Always keep a backup or previous release available for rollback.

## 7. Interview Answer

"In a single-instance application, recreate means stopping the existing application, deploying the new version on the same EC2 instance, and starting it again.

In a multiple-instance application, we stop the old application version across all instances, deploy the new version, and then start the new version. Since the old version is unavailable before the new version becomes available, this approach generally causes downtime.

I would use this strategy when downtime is acceptable or when the application requires a complete shutdown before deploying a new version."

# Example in Detail

### Major runtime upgrade (Java 11 → Java 21)
Scenario:

You have a payment-processing application running on an EC2 instance.
- Current version: Application V1
- Runtime: Java 11
- New version: Application V2
- Required runtime: Java 21
Suppose the application uses native libraries or system-level dependencies that conflict with the new runtime configuration.

You decide not to run both application versions simultaneously on the same environment.

Deployment process:
1. Schedule a maintenance window.
2. Stop incoming traffic or put the application into maintenance mode.
3. Stop the existing application.
4. Back up the current application artifact and configuration.
5. Upgrade Java from 11 to 21.
6. Deploy the new application artifact.
7. Start the application.
8. Perform health checks and smoke tests.
9. Restore normal traffic.
What if deployment fails?
- Stop the new application.
- Restore Java 11 if required.
- Deploy the previous stable artifact.
- Restart the application.
- Verify application health.
Why recreate?

Because you want a clean replacement where the old runtime and new runtime do not interfere with each other.

Note: A Java upgrade does not automatically require recreate deployment. You can also use a separate replacement environment to avoid downtime.

---

### Database migration with incompatible application versions
Scenario:

Your application currently uses an old database schema.

For example:

```
Old database table:
CREATE TABLE employees (
    id INT,
    name VARCHAR(100)
);
```


The new application expects a different schema:
```
CREATE TABLE employees (
    id INT,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    department VARCHAR(100)
);
```

The old application expects the name column, while the new application expects first_name and last_name.

If both application versions run simultaneously, one version may fail because it expects columns that the other schema does not support.

Deployment process:
1. Schedule a maintenance window.
2. Stop the old application.
3. Take a database backup or establish a tested recovery point.
4. Run database migration scripts.
5. Deploy the new application version.
6. Start the application.
7. Validate database connectivity and application functionality.
Why recreate?

Because the old and new application versions are not compatible with the same database schema.

Important production consideration:

For critical applications, prefer backward-compatible database migrations (expand-and-contract pattern). Recreate deployment does not itself protect your database from failed or irreversible migrations.

---

### Batch processing application or scheduled jobs
Relevant to DevOps operations
Scenario:

Imagine your company has a financial reporting application running scheduled jobs through cron.

For example:
- Generate daily financial reports.
- Process customer transactions.
- Export data to S3.
- Send reports to downstream systems.
Suppose you need to deploy a new version of the application.

If the old and new versions run simultaneously, both might execute the same scheduled job.

This can result in:
- Duplicate reports.
- Duplicate transaction processing.
- Duplicate notifications.
- Data inconsistency.
Deployment process:
1. Pause the scheduled jobs.
2. Wait for currently running jobs to complete.
3. Stop the old application.
4. Deploy the new version.
5. Validate configuration and dependencies.
6. Resume scheduled jobs.
7. Monitor the next execution.
Why recreate?

To ensure only one version of the application is responsible for processing scheduled workloads.

An important distinction: recreate deployment alone does not prevent duplicate processing. You should also implement job locking, idempotency, or a single-worker mechanism.

---
