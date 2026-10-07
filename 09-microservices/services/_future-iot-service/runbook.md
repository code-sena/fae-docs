# Runbook — IoT Service

> A runbook is for someone on-call at 3am. Each section must be executable without needing additional context.

**Service:** 06-iot-service
**Repository:** [Service repo URL]
**Version:** 1.0
**Last updated:** YYYY-MM-DD

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Local port | 8087 |
| Production URL | https://api.domain.com/06-iot-service |
| Staging URL | https://staging.api.domain.com/06-iot-service |
| Dashboard | [Grafana URL] |
| Alert channel | [#alerts / PagerDuty] |
| Escalation | [On-call owner] |
| Target RTO | 30 min |
| Target RPO | 5 min |

---

## 2. Verify the service is healthy

```bash
# Health check
curl http://localhost:8087/health
```


---

## 3. Frequent alerts and what to do

### Alert: High 5xx error rate

**Symptom:** Error rate > 2% for 5 minutes.

```bash
kubectl logs -n [namespace] -l app=06-iot-service --tail=100 | grep ERROR
kubectl get pods -n [namespace] -l app=06-iot-service
```

**Decision tree:**
- Crash loops present? → See section 4.1
- DB errors? → See section 4.2
- External dependency issue? → See section 4.3
- Recent deploy? → Evaluate rollback (section 5.3)
---

### Alert: High latency (p95 > SLO)

```bash
kubectl top pods -n [namespace] -l app=06-iot-service
```
---

## 4. Procedures by component

### 4.1 Database

```bash
kubectl exec -n [namespace] [postgres-pod] -- pg_isready -U [user]
```
### 4.2 External dependencies

**If a dependency fails:**
- The service should fail fast and report degraded status
- Check upstream health and dependency status page
- Notify the escalation owner if the issue impacts the SLO

---

## 5. Maintenance operations

### 5.1 Scale the service

```bash
kubectl scale deployment/06-iot-service -n [namespace] --replicas=[N]
```
### 5.2 Rollback

```bash
kubectl rollout history deployment/06-iot-service -n [namespace]
kubectl rollout undo deployment/06-iot-service -n [namespace]
```
---

## 6. Communication during incident

| Event | Channel | Sample message |
|-------|---------|---------------|
| P0 detected | #incidents | [P0 START] IoT Service degraded. Investigating. |
| Update every 15 min | #incidents | [P0 UPDATE] Cause under review. Action in progress. |
| Resolution | #incidents + stakeholders | [P0 RESOLVED] Recovery completed. Post-mortem scheduled. |

---

## 7. Post-incident checklist

- [ ] Service stable with metrics at SLO target
- [ ] Error budget updated in dashboard
- [ ] Incident recorded in 13-operations/incident-management.md
- [ ] Stakeholders notified of resolution
- [ ] Improvement ticket created in project backlog

