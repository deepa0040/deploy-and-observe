# Ques: Deployment Strategies: Interview-Ready Answer

## Each strategy in detail

### 1. Recreate
- **How it works:** Stop all old instances, then start the new version.
- **When and why:** Use it when downtime is acceptable (internal tools, off-hours maintenance) or when two versions can't coexist, such as a breaking DB schema change with no backward compatibility. It's the simplest option with the fewest moving parts.
- **Cost:** The cheapest, since there's no duplicate infrastructure.
- **Rollback:** Redeploy the previous version, which means another round of downtime. Recovery time depends on your build and startup time.

```
In a Recreate deployment, we shut down all instances of the old version first, and only then start the new version. Because of this, some downtime is expected between the two steps.

We choose this strategy when two versions of the application can't run at the same time. For example, when the new version has an incompatible database schema, different library or config requirements, or needs a fixed port or exclusive resource that the old version is holding.

The advantages are that it's simple, has no version-mixing problems, and needs no extra infrastructure. The drawbacks are the downtime and a slower rollback, since we have to redeploy the old version.

So I'd use it for dev/test environments, internal tools, or off-hours maintenance windows where downtime is acceptable. For production systems needing high availability, I'd prefer Rolling, Blue-Green, or Canary instead.

Note: We use Recreate when we have limited infrastructure, for example a single instance, so we can't run old and new versions side by side, or when the versions are incompatible and can't safely run together.
```

### 2. Rolling
- **How it works:** Replace instances in batches (Kubernetes `maxSurge` / `maxUnavailable` control this).
- **When and why:** It's the sensible default for stateless services. You get zero downtime without paying for a second environment.
- **Cost:** Low. You only need a little surge headroom.
- **Rollback:** Slow, because you have to roll the batches back, and during the process old and new versions serve traffic together. That makes **backward-compatible APIs and DB schemas** essential. Kubernetes supports `kubectl rollout undo`, but it's still gradual.

### 3. Blue/Green
- **How it works:** Run two full environments. Blue is live, and you deploy and test on Green, then switch the load balancer or DNS.
- **When and why:** Choose it when you need near-instant rollback and a full pre-production smoke test against production-like infra. It suits regulated or revenue-critical systems and big-bang releases.
- **Cost:** The most expensive, at roughly 2x infra while both environments run. You can lower it by scaling down the idle environment after a bake period, or by using cloud auto-scaling.
- **Rollback:** The fastest: point traffic back to Blue, which is still running the old version. The catch is **database compatibility**. If Green ran migrations or wrote new-format data, switching back can break things, so use expand/contract migrations.

### 4. Canary
- **How it works:** Send a small slice of traffic (5% → 25% → 100%) to the new version, checking metrics (error rate, latency, business KPIs) at each step. Tools include Argo Rollouts, Flagger, Istio and Spinnaker.
- **When and why:** Use it for large user bases where a bad release is costly and you want real production signal before full exposure. It limits blast radius.
- **Cost:** Moderate. Capacity overhead is small, but you need solid **observability, traffic routing (service mesh or ingress), and automated analysis**, which is the real investment.
- **Rollback:** Fast. Shift the canary's traffic back to 0%, and only the small slice of affected users sees the problem. Automated rollback on metric thresholds is common.

### 5. Feature flags (dark launches)
- **How it works:** Ship code to production switched off, then enable it for specific users, percentages or segments at runtime (LaunchDarkly, Unleash, Flagsmith).
- **When and why:** Use it when you want to decouple *deploy* from *release*, run A/B tests, do trunk-based development, or have a kill switch for risky features.
- **Cost:** Little extra infrastructure, but there's tooling cost, testing complexity (code paths multiply), and **flag debt** if you don't clean up old flags.
- **Rollback:** Instant and needs no redeploy: flip the flag off. It only covers the flagged code, so it won't undo infra or schema changes.

## How to choose

- **Downtime OK, simplest possible?** Recreate.
- **Stateless service, backward-compatible, cost-conscious?** Rolling.
- **Need instant rollback and can afford duplicate infra?** Blue/Green.
- **High traffic, strong metrics, minimizing user impact?** Canary.
- **Want to release independently of deploys?** Feature flags, usually *combined* with rolling or canary.

In practice, mature teams combine them: **rolling or canary for the deployment, plus feature flags for the release**.

## Cross-cutting point interviewers like

**Rollback is only as good as your data layer.** Application rollback is easy, but database rollback is hard. Use the **expand/contract pattern** (add new columns first, deploy code that handles both, and drop old columns later) so any strategy can safely roll back.

## One-line summary for an interview

> "Recreate is cheapest but has downtime. Rolling is the cheap default with slow rollback. Blue/Green gives instant rollback at about 2x cost. Canary limits blast radius but needs strong observability. Feature flags decouple deploy from release with instant rollback, at the price of flag debt. I'd pick based on downtime tolerance, risk, traffic volume and budget, and often combine canary or rolling with flags."

# Ques: Data layer Role in Deployment Strategies
Both of these come down to one problem: during a deployment, **old and new code often run at the same time, and they both talk to the same database.**

## What "backward-compatible" means

A new version is backward-compatible if it keeps working with everything the old version worked with, so the old and new versions can run side by side without breaking each other.

It applies in three places:

- **APIs:** If v2 of your service renames a JSON field from `user_name` to `username`, any client or service still expecting `user_name` breaks. A backward-compatible change *adds* `username` and keeps returning `user_name` for a while.
- **Database schema:** If v2 expects a new column `phone_number`, but v1 instances are still running and don't know about it, you need to make sure v1 doesn't crash when the column exists (and v2 doesn't crash if it's not populated yet).
- **Messages/events:** If a service publishes events to a queue, new-format messages shouldn't crash old consumers still reading them.

**Why it matters for deployment:** in rolling and canary deployments, both versions are live at once, serving real traffic. If they're incompatible, you get errors during the rollout, or you can't roll back.

## The data layer's role

Application code is **stateless**: rolling back means swapping to the old binary or container. The database is **stateful**: it holds real data, and schema changes and data writes can't simply be "un-done" by redeploying old code.

That creates three problems:

**1. Mixed versions share one database.**
During a rolling or canary release, v1 and v2 both read and write the same tables. If v2 changed the schema in a way v1 can't handle, v1 instances start failing mid-rollout.

**2. Rollback after a migration may be impossible.**
Say v2 runs a migration that drops a column or changes a data format, then you find a bug and roll back to v1. Now v1 expects the old column, which is gone. This is why Blue/Green's "instant rollback" has a catch: switching traffic back is instant, but the database is not switched back with it, since both Blue and Green usually point at the same DB.

**3. New data written by v2 may be unreadable by v1.**
Even without schema changes, if v2 starts writing data in a new format, rolling back leaves v1 reading data it doesn't understand.

## The fix: the expand/contract pattern

Never make a breaking schema change in one step. Split it into phases:

**Example: renaming a column `name` → `full_name`**

1. **Expand:** Add the new column `full_name`. Keep `name`. (Old code still works.)
2. **Deploy code that writes to both** columns and reads from the new one (falling back to the old). Now v1 and v2 can both run safely, and rollback is safe.
3. **Backfill:** Copy existing `name` values into `full_name`.
4. **Contract:** Once you're confident and no old version is running, deploy code that only uses `full_name`, then drop `name` in a later release.

Each step is independently safe to roll back. The cost is that a single change takes several releases instead of one.

## How this connects to each strategy

| Strategy | Why the data layer matters |
|---|---|
| **Recreate** | Least problem: only one version is ever live, so a breaking migration is fine (you already accept downtime). Rollback still needs a DB restore if the migration was destructive. |
| **Rolling** | Old and new versions overlap, so schema and API **must** be compatible with both. |
| **Blue/Green** | Both environments usually share one DB, so the schema must work for Blue *and* Green. Otherwise instant rollback is unsafe. |
| **Canary** | The canary and the stable version share the DB, so the new version must be compatible with the old one's reads and writes. |
| **Feature flags** | Flags can be turned off instantly, but any data the flagged code already wrote stays, so old paths must tolerate it. |

## One-line answer for an interview

> "Backward-compatible means the new version works alongside the old one, so both can run during a rollout or after a rollback. The data layer is the hard part because code is stateless but the database isn't. Both versions share it, and migrations can't be undone by redeploying. So I use expand/contract migrations: add first, migrate code and data, remove old structures later, so every deploy step stays safely reversible."