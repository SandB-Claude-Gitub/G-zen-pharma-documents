# ZenPharma — Hands-On Troubleshooting Labs

> **How to use this document:**  
> Each lab corresponds to one or more interview questions from `INTERVIEW-QA.md`.  
> You will intentionally break something in the live environment, observe what happens, and then fix it.  
> After each lab you will have personally experienced the failure mode — not just read about it.

---

## Before You Start — Create a Restore Point

Tag the current state of all four repositories. If anything goes wrong, one command brings you back.

### Step 1 — Tag Every Repo at its Current Stable State

```bash
# Run once before starting any lab
for repo in infra frontend backend gitops; do
  cd /Users/ravdsun/devops/zenpharma/$repo

  # Commit any uncommitted changes (so the tag is clean)
  git add -A
  git commit -m "chore: snapshot before troubleshooting labs" 2>/dev/null \
    || echo "$repo: nothing to commit"

  # Create an annotated tag
  git tag -a v1.0-stable -m "Stable state before troubleshooting course"

  echo "Tagged $repo at v1.0-stable"
  cd -
done
```

### Step 2 — Push Tags to GitHub

```bash
for repo in infra frontend backend gitops; do
  cd /Users/ravdsun/devops/zenpharma/$repo
  git push origin v1.0-stable
  cd -
done
```

---

### Emergency Restore — Full Reset to v1.0-stable

If a lab leaves things in an unrecoverable state, run this for the affected repo:

```bash
# Replace <repo> with: infra | frontend | backend | gitops
cd /Users/ravdsun/devops/zenpharma/<repo>

git fetch --tags
git reset --hard v1.0-stable
git push origin main --force          # only if you need GitHub to match
```

For the Kubernetes cluster (ArgoCD will re-sync from gitops automatically):

```bash
# Force ArgoCD to sync all apps from the restored gitops state
kubectl get applications -n argocd -o name | xargs -I{} kubectl patch {} -n argocd \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

---

## Labs Quick Reference

| Lab | Interview Q | Concept | Time |
|-----|-------------|---------|------|
| [LAB-01](#lab-01-argocd-selfheal--gitops-enforcement) | Q21 | ArgoCD selfHeal reverts manual kubectl changes | 10 min |
| [LAB-02](#lab-02-rollback-via-git-revert) | Q30 | Rollback a bad deployment using git revert | 15 min |
| [LAB-03](#lab-03-imagepullbackoff--wrong-image-tag) | Q28 | Diagnose and fix ImagePullBackOff | 10 min |
| [LAB-04](#lab-04-crashloopbackoff--broken-secret) | Q27, Q4, Q28 | Diagnose CrashLoopBackOff caused by missing secret | 15 min |
| [LAB-05](#lab-05-externalsecret-sync-failure) | Q27 | Force ESO re-sync; recover from SecretSyncedError | 15 min |
| [LAB-06](#lab-06-database-password-rotation) | Q9 | Rotate DB password with zero downtime | 20 min |
| [LAB-07](#lab-07-ingress-deletion--alb-reprovisioning) | Q5, Q6 | Delete Ingress, watch ALB deprovisioned, restore | 15 min |
| [LAB-08](#lab-08-gitleaks--secret-detection-in-pipeline) | Q31 | Trigger GitLeaks failure with a fake credential | 10 min |
| [LAB-09](#lab-09-argocd-prune--resource-deleted-from-git) | Q21 | ArgoCD prune deletes a resource removed from gitops | 15 min |
| [LAB-10](#lab-10-pod-security-context--readonlyrootfilesystem) | Q16, Q19 | Observe readOnlyRootFilesystem blocking writes | 10 min |
| [LAB-11](#lab-11-readiness-probe-failure) | Q19, Q28 | Change probe path → pod never becomes Ready | 10 min |
| [LAB-12](#lab-12-terraform-state-lock-stuck) | Q11 | Simulate stuck Terraform lock and release it | 10 min |

---

## LAB-01: ArgoCD selfHeal — GitOps Enforcement

**Interview Q:** Q21 — Can you explain the ArgoCD Application file?  
**Concept:** `selfHeal: true` means ArgoCD automatically reverts any manual change made directly against the cluster — git is the only source of truth.  
**Time:** ~10 minutes

---

### Break It

```bash
# Manually scale pharma-ui to 0 replicas — simulating a "quick fix" someone did in panic
kubectl scale deployment pharma-ui -n dev --replicas=0

# Verify it is gone
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui
# Expected: No resources found
```

### Observe It

```bash
# Watch ArgoCD detect the drift and correct it — should happen within 3 minutes
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui -w
# You will see: pharma-ui pod appear again without you doing anything

# Also watch in ArgoCD UI:
# https://localhost:8080 (if port-forwarded)
# The pharma-ui-dev app will briefly show "OutOfSync" then go back to "Synced"
```

Check the ArgoCD sync events:

```bash
kubectl describe application pharma-ui-dev -n argocd | grep -A5 "Events"
```

### Fix It

Nothing to fix — ArgoCD already fixed it. The point of the lab is to see the revert happen automatically.

To permanently set replicas, you must update the gitops values file:

```yaml
# gitops/envs/dev/values-pharma-ui.yaml
replicaCount: 2    # only way to make this stick
```

### What You Practised

- Observed `selfHeal: true` in action
- Understood why `kubectl edit` or `kubectl scale` is not a valid way to make changes in a GitOps environment
- Can explain this in an interview: *"ArgoCD reverted the manual change within 3 minutes — selfHeal ensures git is always the single source of truth"*

---

## LAB-02: Rollback via Git Revert

**Interview Q:** Q30 — What is your rollback process?  
**Concept:** The preferred rollback is a git revert in the gitops repo — it creates an auditable commit and lets ArgoCD re-sync back to the previous image.  
**Time:** ~15 minutes

---

### Break It

Simulate a bad deployment by pushing a nonexistent image tag to the gitops dev values file.

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Record the current good tag (you'll need it)
GOOD_TAG=$(grep "tag:" envs/dev/values-pharma-ui.yaml | awk '{print $2}')
echo "Current good tag: $GOOD_TAG"

# Push a bad tag
sed -i '' "s/tag: .*/tag: sha-0000bad/" envs/dev/values-pharma-ui.yaml

git add envs/dev/values-pharma-ui.yaml
git commit -m "ci(dev): update pharma-ui → sha-0000bad"
git push origin main
```

### Observe It

Wait ~3 minutes for ArgoCD to sync, then:

```bash
# Watch pharma-ui pod fail to pull the image
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui -w
# Expected: STATUS = ImagePullBackOff

# Confirm the image pull error
kubectl describe pod -n dev -l app.kubernetes.io/name=pharma-ui | grep -A8 "Events"
# Expected: "Failed to pull image ... not found"
```

### Fix It

**Option A — Git revert (preferred):**

```bash
cd /Users/ravdsun/devops/zenpharma/gitops
git revert HEAD --no-edit
git push origin main
# ArgoCD detects the revert and deploys the previous good image within 3 minutes
```

**Option B — ArgoCD History rollback (fast, from UI):**

```bash
# Port-forward if not already open
kubectl port-forward svc/argocd-server -n argocd 8080:443

# In browser: https://localhost:8080
# Click pharma-ui-dev → History and Rollback → click the previous revision → Rollback
```

Verify the pod recovers:

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui
# Expected: Running 1/1
```

### What You Practised

- Experienced an ImagePullBackOff caused by a bad image tag
- Rolled back using `git revert` — the preferred GitOps rollback method
- Saw that ECR kept the old image (ECR lifecycle keeps last 10) so the rollback target was always available

---

## LAB-03: ImagePullBackOff — Wrong Image Tag

**Interview Q:** Q28 — One microservice is not working. How do you diagnose it?  
**Concept:** ImagePullBackOff means Kubernetes cannot pull the container image — could be wrong tag, wrong ECR URL, or missing ECR permissions.  
**Time:** ~10 minutes

---

### Break It

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Set wrong tag for api-gateway
sed -i '' "s/tag: .*/tag: sha-notexist/" envs/dev/values-api-gateway.yaml

git add envs/dev/values-api-gateway.yaml
git commit -m "ci(dev): update api-gateway → sha-notexist"
git push origin main
```

### Observe It

Work through the full diagnostic flow:

```bash
# Step 1 — What is the pod status?
kubectl get pods -n dev -l app.kubernetes.io/name=api-gateway
# Expected: 0/1  ImagePullBackOff

# Step 2 — What does describe say?
kubectl describe pod -n dev -l app.kubernetes.io/name=api-gateway
# Look at Events section — you will see:
# "Failed to pull image ... sha-notexist: not found"
# "Back-off pulling image"

# Step 3 — What image is being pulled? Is the ECR URL correct?
kubectl get pod -n dev -l app.kubernetes.io/name=api-gateway \
  -o jsonpath='{.items[0].spec.containers[0].image}'
# You will see: <account>.dkr.ecr.us-east-1.amazonaws.com/api-gateway:sha-notexist

# Step 4 — Does the tag actually exist in ECR?
aws ecr list-images --repository-name api-gateway \
  --query 'imageIds[*].imageTag' --output table
# You will see the real tags — sha-notexist is not there
```

### Fix It

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Restore the last good tag from git history
git revert HEAD --no-edit
git push origin main
```

### What You Practised

- Followed the diagnostic flow: pod status → describe → image URL → ECR verification
- Can distinguish ImagePullBackOff (image not found) from ErrImagePull (ECR access denied)
- Knows the fix: correct the tag in gitops values file, not `kubectl set image`

---

## LAB-04: CrashLoopBackOff — Broken Secret

**Interview Q:** Q27, Q4, Q28  
**Concept:** CrashLoopBackOff on a Spring Boot service usually means a required environment variable is missing or wrong. The most common cause is an ExternalSecret that failed to sync.  
**Time:** ~15 minutes

---

### Break It

Delete the Kubernetes Secret that holds DB credentials — simulating what happens when a secret is accidentally deleted or ESO is not yet synced.

```bash
# Delete the DB secret in the dev namespace
kubectl delete secret pharma-dev-db-secret -n dev

# Immediately restart auth-service to pick up the missing secret
kubectl rollout restart deployment/auth-service -n dev
```

### Observe It

```bash
# Watch auth-service crash
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service -w
# Expected: CrashLoopBackOff within 30–60 seconds

# Check the crash logs
kubectl logs -n dev -l app.kubernetes.io/name=auth-service --previous
# Expected: Spring Boot startup failure — something like:
# "Failed to configure a DataSource: 'url' attribute is not specified"
# or: "java.lang.IllegalStateException: Cannot load configuration class"

# Check the pod events
kubectl describe pod -n dev -l app.kubernetes.io/name=auth-service
# Look for: "Error: secret 'pharma-dev-db-secret' not found"
```

### Fix It

ESO will automatically recreate the Kubernetes Secret within its sync interval. To force it immediately:

```bash
# Force ESO to re-sync right now
kubectl annotate externalsecret pharma-dev-db-secret -n dev \
  force-sync=$(date +%s) --overwrite

# Wait ~10 seconds, then verify the secret is back
kubectl get secret pharma-dev-db-secret -n dev
# Expected: Opaque secret with 3 keys

# Restart auth-service to pick up the recreated secret
kubectl rollout restart deployment/auth-service -n dev

# Watch it recover
kubectl rollout status deployment/auth-service -n dev
```

### What You Practised

- Experienced CrashLoopBackOff caused by a missing secret
- Used `--previous` flag on logs to see crash output from a dead container
- Learned that ESO recreates secrets automatically (self-healing secret management)
- Used `force-sync` annotation to trigger immediate ESO resync

---

## LAB-05: ExternalSecret Sync Failure

**Interview Q:** Q27 — What is External Secrets Operator and how does it work?  
**Concept:** If ESO cannot read from Secrets Manager (wrong IAM permissions or wrong secret name), it shows `SecretSyncedError` and the Kubernetes Secret is not created/updated.  
**Time:** ~15 minutes

---

### Break It

Patch the ExternalSecret to point to a nonexistent Secrets Manager key.

```bash
# Save the current spec so you can restore it
kubectl get externalsecret pharma-dev-db-secret -n dev -o yaml > /tmp/eso-backup.yaml

# Patch to use a wrong Secrets Manager key name
kubectl patch externalsecret pharma-dev-db-secret -n dev \
  --type json \
  -p '[{"op":"replace","path":"/spec/data/0/remoteRef/key","value":"pharma-dev-db-secret-WRONG"}]'
```

### Observe It

```bash
# Watch the ExternalSecret status change
kubectl get externalsecret -n dev -w
# STATUS column will change to: SecretSyncedError

# Get the error detail
kubectl describe externalsecret pharma-dev-db-secret -n dev
# Look for Conditions section:
# Type:    Ready
# Status:  False
# Reason:  SecretSyncedError
# Message: "ResourceNotFoundException: Secrets Manager can't find the specified secret"

# Check ESO controller logs for more detail
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=30
```

### Fix It

```bash
# Restore from the backup you saved
kubectl apply -f /tmp/eso-backup.yaml

# Wait for sync
kubectl get externalsecret -n dev -w
# STATUS should return to: SecretSynced within 30 seconds
```

### What You Practised

- Triggered and read a `SecretSyncedError` status
- Identified the Secrets Manager key name mismatch from ESO events
- Restored the ExternalSecret config
- Can explain in interview: *"We saw this failure when a secret was renamed in Secrets Manager but the ExternalSecret wasn't updated — pods started failing because ESO couldn't find the key"*

---

## LAB-06: Database Password Rotation

**Interview Q:** Q9 — How do you rotate/update the database password?  
**Concept:** Change password in Secrets Manager → ESO re-syncs the Kubernetes Secret → rolling restart picks up the new credentials. Zero downtime if done correctly.  
**Time:** ~20 minutes

---

### Break It (Simulate Credential Mismatch)

This simulates what happens if Secrets Manager has the new password but pods still use the old one.

```bash
# Get the current secret ARN
aws secretsmanager list-secrets \
  --query "SecretList[?contains(Name,'pharma-dev-db')].ARN" \
  --output text

# Save the current value
aws secretsmanager get-secret-value \
  --secret-id pharma-dev-db-secret \
  --query SecretString --output text > /tmp/original-secret.json

cat /tmp/original-secret.json
# Note the current password field value
```

Simulate a wrong password update (update Secrets Manager to a bad password):

```bash
# Get original values
ORIG_JSON=$(cat /tmp/original-secret.json)

# Push a wrong password to Secrets Manager (simulates accidental rotation to wrong value)
aws secretsmanager put-secret-value \
  --secret-id pharma-dev-db-secret \
  --secret-string '{"username":"pharmaadmin","password":"WrongPassword999"}'
```

Force ESO to re-sync immediately so the K8s Secret gets the wrong password:

```bash
kubectl annotate externalsecret pharma-dev-db-secret -n dev \
  force-sync=$(date +%s) --overwrite

# Verify the K8s Secret was updated
kubectl get secret pharma-dev-db-secret -n dev \
  -o jsonpath='{.data.SPRING_DATASOURCE_PASSWORD}' | base64 -d
# You should see: WrongPassword999

# Now restart auth-service to pick up the wrong password
kubectl rollout restart deployment/auth-service -n dev
```

### Observe It

```bash
# Watch auth-service fail to connect to DB
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service -w
# Expected: CrashLoopBackOff (DB connection refused / auth failure)

kubectl logs -n dev -l app.kubernetes.io/name=auth-service --previous
# Expected: "password authentication failed for user pharmaadmin"
# or: "HikariPool-1 - Connection is not available, request timed out"
```

### Fix It

Restore the correct password in Secrets Manager:

```bash
# Restore original secret value
aws secretsmanager put-secret-value \
  --secret-id pharma-dev-db-secret \
  --secret-string "$ORIG_JSON"

# Force ESO to re-sync
kubectl annotate externalsecret pharma-dev-db-secret -n dev \
  force-sync=$(date +%s) --overwrite

# Wait for sync
sleep 15

# Rolling restart — zero downtime (new pods start with correct credentials before old ones die)
kubectl rollout restart deployment/auth-service -n dev

# Watch recovery
kubectl rollout status deployment/auth-service -n dev
```

Verify service is healthy:

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service
# Expected: 1/1 Running
```

### What You Practised

- Full credential rotation flow: Secrets Manager → ESO → K8s Secret → pod
- Observed DB authentication failure in Spring Boot logs
- Performed zero-downtime rolling restart
- Can explain: *"The rotation path is always Secrets Manager first, then ESO re-syncs, then rolling restart — pods with old credentials keep serving traffic until new pods with correct credentials are healthy"*

---

## LAB-07: Ingress Deletion — ALB Deprovisioning

**Interview Q:** Q5 — How does ingress traffic flow? / Q6 — How did you migrate to AWS Load Balancer Controller?  
**Concept:** The ALB is created and destroyed by the AWS Load Balancer Controller in response to Ingress resources. Delete the Ingress → ALB disappears. Recreate the Ingress → new ALB is provisioned.  
**Time:** ~15 minutes

---

### Break It

```bash
# Record the current ALB DNS name
ALB_URL=$(kubectl get ingress -n dev pharma-ui \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Current ALB: $ALB_URL"

# Delete the pharma-ui Ingress
kubectl delete ingress pharma-ui -n dev
```

### Observe It

```bash
# Kubernetes Ingress is gone
kubectl get ingress -n dev
# Expected: No resources found

# In AWS Console: EC2 → Load Balancers
# The pharma-dev ALB will show "deleting" status then disappear within 2–5 minutes

# Also verify via CLI
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName,`pharma-dev`)].State.Code' \
  --output text
# Eventually returns nothing (load balancer removed)

# The app is now unreachable
curl -s -o /dev/null -w "%{http_code}" http://$ALB_URL/
# Expected: Could not connect / 000
```

### Fix It

ArgoCD will detect the missing Ingress (it's in the gitops repo) and re-apply it automatically within ~3 minutes because of `selfHeal: true`. To trigger it immediately:

```bash
# Force ArgoCD to sync pharma-ui-dev
argocd app sync pharma-ui-dev --auth-token $(
  kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
) --server localhost:8080 --insecure 2>/dev/null \
|| echo "If argocd CLI not set up, wait 3 min for auto-selfheal"

# Or trigger via kubectl patch
kubectl patch application pharma-ui-dev -n argocd \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

Watch ALB come back:

```bash
# Watch for Ingress to be re-created and get an address
kubectl get ingress -n dev -w
# ADDRESS column will populate once the new ALB is provisioned (~2–4 minutes)

# Test the new ALB URL
NEW_ALB=$(kubectl get ingress -n dev pharma-ui \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "%{http_code}" http://$NEW_ALB/
# Expected: 200
```

### What You Practised

- Saw that deleting an Ingress deprovisions the AWS ALB (the ALB is not independent of Kubernetes)
- Observed ALB reprovisioning via selfHeal
- Can explain: *"This is exactly why we must delete Ingress resources before running terraform destroy — otherwise Terraform can't delete the VPC because the ALB controller left an ALB attached to the subnets"*

---

## LAB-08: GitLeaks — Secret Detection in Pipeline

**Interview Q:** Q31 — What was a recent issue you faced?  
**Concept:** GitLeaks scans commit history for secrets before the pipeline proceeds. A fake AWS access key in any tracked file will fail the scan.  
**Time:** ~10 minutes

---

### Break It

Add a fake (not real) AWS access key pattern to a test file in the backend repo:

```bash
cd /Users/ravdsun/devops/zenpharma/backend

# Create a test file that contains a fake AWS credential pattern
# This is NOT a real key — it just matches the regex pattern
cat > /tmp/fake-secret-test.txt << 'EOF'
# Test file for GitLeaks lab
# The following is a FAKE credential (not real, never was real)
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

cp /tmp/fake-secret-test.txt api-gateway/src/test/resources/fake-test.properties

git add api-gateway/src/test/resources/fake-test.properties
git commit -m "test: adding test properties file"
git push origin develop
```

### Observe It

```bash
# Go to GitHub: https://github.com/<your-org>/backend → Actions
# The CI/CD — api-gateway workflow triggers on push to develop
# The FIRST job is GitLeaks scan — it should FAIL before Maven even starts

# Expected pipeline output in the GitLeaks step:
# ╭─────────────────────────────────────╮
# │ 1 leak found in 1 commits scanned  │
# │ File: api-gateway/src/test/.../    │
# │ Rule: aws-access-token             │
# ╰─────────────────────────────────────╯
# exit code: 1  → pipeline FAILS
```

### Fix It

```bash
cd /Users/ravdsun/devops/zenpharma/backend

# Remove the offending file
git rm api-gateway/src/test/resources/fake-test.properties
git commit -m "fix: remove file containing fake credential patterns"
git push origin develop
```

If GitLeaks scans full history and still finds the removed commit:

```bash
# Rewrite history to remove the file entirely (destructive — use carefully)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch api-gateway/src/test/resources/fake-test.properties' \
  --prune-empty --tag-name-filter cat -- --all

git push origin develop --force
```

### What You Practised

- Triggered a GitLeaks failure with a fake credential
- Saw that the scan runs before any other CI step — fail fast principle
- Practised the clean-up: remove file → rewrite history if needed
- Can tell the interview story from experience: *"I added a test file with a fake key to see what GitLeaks would do — pipeline failed within 30 seconds of the push, before Maven ran a single test"*

---

## LAB-09: ArgoCD Prune — Resource Deleted from Git

**Interview Q:** Q21 — Can you explain the ArgoCD Application file?  
**Concept:** `prune: true` means ArgoCD deletes Kubernetes resources that were removed from the gitops repo. Remove a manifest from git → ArgoCD removes the resource from the cluster.  
**Time:** ~15 minutes

---

### Break It

Remove the HPA (HorizontalPodAutoscaler) for pharma-ui from the gitops Helm chart values — ArgoCD will delete it from the cluster.

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Check if HPA exists
kubectl get hpa -n dev

# Disable HPA for pharma-ui by setting autoscaling.enabled: false in values
# (This is safer than deleting a core resource)
sed -i '' 's/autoscaling:/autoscaling:\n  enabled: false  # LAB-09/' \
  envs/dev/values-pharma-ui.yaml 2>/dev/null || true

# Or patch directly in values-pharma-ui.yaml — open file and set:
# autoscaling:
#   enabled: false

git add envs/dev/values-pharma-ui.yaml
git commit -m "chore(lab): disable pharma-ui HPA for prune test"
git push origin main
```

### Observe It

```bash
# Watch ArgoCD sync within 3 minutes
kubectl get applications pharma-ui-dev -n argocd -w
# STATUS: Synced

# The HPA that existed in the cluster should now be gone (pruned)
kubectl get hpa -n dev
# pharma-ui HPA no longer exists

# In ArgoCD UI: click pharma-ui-dev → App Details → you can see it removed the HPA
```

### Fix It

Restore the HPA by reverting the values change:

```bash
cd /Users/ravdsun/devops/zenpharma/gitops
git revert HEAD --no-edit
git push origin main
```

### What You Practised

- Saw `prune: true` delete a real cluster resource when its definition was removed from git
- Understood why prune is powerful but requires care: removing a values key can delete production resources
- Can explain: *"prune: true means git is truth for what should exist in the cluster — not just what should be configured"*

---

## LAB-10: Pod Security Context — readOnlyRootFilesystem

**Interview Q:** Q16, Q19 — Security enforced in your project / Deployment manifest  
**Concept:** `readOnlyRootFilesystem: true` prevents any process in the container from writing to the container's filesystem — even as an attacker with code execution.  
**Time:** ~10 minutes

---

### Observe It (No Breaking Required)

```bash
# Exec into the pharma-ui container
kubectl exec -it -n dev \
  $(kubectl get pod -n dev -l app.kubernetes.io/name=pharma-ui -o name | head -1) \
  -- /bin/sh

# Inside the container — try to write to the filesystem
echo "test" > /usr/share/nginx/html/test.txt
# Expected: /bin/sh: can't create /usr/share/nginx/html/test.txt: Read-only file system

echo "test" > /etc/hacked
# Expected: Read-only file system

# But writable volumes (emptyDir) work fine
echo "test" > /tmp/test.txt
# Expected: succeeds — /tmp is an emptyDir volume mount

echo "test" > /var/cache/nginx/test.txt
# Expected: succeeds — mounted as emptyDir

exit
```

### Break It (Optional — more dramatic)

Change the security context to `readOnlyRootFilesystem: false` in the gitops values file and see if an exec can now write to disk:

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Edit envs/dev/values-pharma-ui.yaml
# Find securityContext block and change:
# readOnlyRootFilesystem: false

git add envs/dev/values-pharma-ui.yaml
git commit -m "chore(lab): disable readOnlyRootFilesystem for testing"
git push origin main
```

After ArgoCD syncs, exec in and try writing to filesystem — it will succeed.

### Fix It

```bash
cd /Users/ravdsun/devops/zenpharma/gitops
git revert HEAD --no-edit
git push origin main
```

### What You Practised

- Verified that readOnlyRootFilesystem actually blocks writes
- Confirmed emptyDir mounts are still writable (necessary for Nginx temp files)
- Experienced the defence: *"Even with code execution, an attacker can't modify the Nginx binary, write a backdoor, or persist files — the filesystem is sealed"*

---

## LAB-11: Readiness Probe Failure

**Interview Q:** Q19, Q28 — Deployment manifest / diagnosing a broken microservice  
**Concept:** A pod with a failing readiness probe stays `Running` but shows `0/1` Ready — it receives no traffic from Services/ALB. The ALB will mark the target as unhealthy.  
**Time:** ~10 minutes

---

### Break It

Change the readiness probe path for api-gateway to a nonexistent endpoint:

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Open envs/dev/values-api-gateway.yaml
# Find the readinessProbe section and change the path:
# readinessProbe:
#   path: /actuator/health/readiness   ← change to /actuator/health/BROKEN

# Quick sed:
sed -i '' 's|path: /actuator/health/readiness|path: /actuator/health/BROKEN|' \
  envs/dev/values-api-gateway.yaml

git add envs/dev/values-api-gateway.yaml
git commit -m "chore(lab): break api-gateway readiness probe"
git push origin main
```

### Observe It

Wait ~3 minutes for ArgoCD to sync the new values:

```bash
# Check pod status
kubectl get pods -n dev -l app.kubernetes.io/name=api-gateway
# Expected: READY = 0/1  STATUS = Running
# The pod is running but NOT ready — probe is failing

# Get probe failure details
kubectl describe pod -n dev -l app.kubernetes.io/name=api-gateway
# Look in Events section:
# "Readiness probe failed: HTTP probe failed with statuscode: 404"

# The pod is Running but receives no traffic
# Any request through the ALB to /api/* will return 502/503

# Test: the frontend page loads but API calls fail
ALB_URL=$(kubectl get ingress -n dev pharma-ui \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "%{http_code}" http://$ALB_URL/
# Expected: 200 (frontend loads — it doesn't go through api-gateway)

curl -s -o /dev/null -w "%{http_code}" http://$ALB_URL/api/actuator/health
# Expected: 502 or 503 (api-gateway target is unhealthy)
```

### Fix It

```bash
cd /Users/ravdsun/devops/zenpharma/gitops
git revert HEAD --no-edit
git push origin main
```

Verify recovery:

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=api-gateway
# Expected: READY = 1/1 within 1–2 minutes of ArgoCD sync
```

### What You Practised

- Distinguished `Running 0/1` (probe failing) from `CrashLoopBackOff` (process crashing)
- Read probe failure messages from `kubectl describe`
- Understood that a pod can be Running but receive no traffic — the readiness probe is the traffic gate
- Can explain: *"The pod was Running but 0/1 Ready — the readiness probe was returning 404, so Kubernetes never added it to the Service endpoints, which means the ALB kept routing to... nothing"*

---

## LAB-12: Terraform State Lock Stuck

**Interview Q:** Q11 — CI/CD pipeline end-to-end  
**Concept:** When a Terraform pipeline is cancelled mid-run, it leaves a `.tflock` file in S3. Subsequent runs fail with "lock already acquired." You must delete the lock file manually.  
**Time:** ~10 minutes

---

### Break It

Manually create a fake lock file in S3 — simulating a cancelled pipeline leaving a lock behind.

```bash
# Create a fake lock file
echo '{"ID":"fake-lock-for-lab","Operation":"OperationTypeApply"}' > /tmp/fake.tflock

# Upload it to the S3 state location
aws s3 cp /tmp/fake.tflock \
  s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock

echo "Fake lock created"
```

Now try to run Terraform locally to see the error:

```bash
cd /Users/ravdsun/devops/zenpharma/infra/envs/dev

# Initialize
terraform init

# Attempt a plan — should fail on lock
terraform plan
# Expected error:
# "Error: Error acquiring the state lock"
# "Lock Info: ID: fake-lock-for-lab"
# "Operation: OperationTypeApply"
```

### Observe It

```bash
# Verify the lock file exists in S3
aws s3 ls s3://zen-pharma-terraform-state-ravdy/envs/dev/ | grep tflock
# Expected: terraform.tfstate.tflock listed

# In a real scenario, check if any pipeline is actually running first
# If a pipeline is in progress, DO NOT delete the lock — wait for it to finish
# Only delete if you're certain no pipeline is mid-run
```

### Fix It

```bash
# Remove the stuck lock file
aws s3 rm s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock

# Verify it is gone
aws s3 ls s3://zen-pharma-terraform-state-ravdy/envs/dev/ | grep tflock
# Expected: no output (file is gone)

# Now terraform plan should work
terraform plan
# Expected: plan completes without lock error
```

### What You Practised

- Created and diagnosed a stuck Terraform state lock
- Understood the S3 native locking mechanism (`use_lockfile = true` creates `.tflock` files)
- Practised the safe recovery: check if a pipeline is running → then delete the lock
- Can answer: *"We use S3 native state locking with `use_lockfile = true`. If a pipeline crashes mid-run it leaves a `.tflock` file. We remove it with `aws s3 rm` after confirming no other run is in progress"*

---

## After You Are Done — Restore to v1.0-stable

Run this to bring every repo back to the tagged stable state:

```bash
for repo in infra frontend backend gitops; do
  cd /Users/ravdsun/devops/zenpharma/$repo
  echo "=== Restoring $repo ==="
  git fetch --tags
  git reset --hard v1.0-stable
  git push origin main --force
  echo "=== $repo restored ==="
  cd -
done
```

Then force ArgoCD to re-sync all apps from the restored gitops:

```bash
kubectl get applications -n argocd -o name | \
  xargs -I{} kubectl patch {} -n argocd \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

Wait 2–3 minutes. Verify full health:

```bash
kubectl get pods -n dev
# All pods: 1/1 Running

kubectl get applications -n argocd
# All apps: Synced + Healthy

kubectl get externalsecret -n dev
# All: SecretSynced
```

---

## Lab-to-Interview-Question Map

| Lab | Interview Q | If Asked This, Mention the Lab |
|-----|-------------|-------------------------------|
| LAB-01 | Q21 ArgoCD selfHeal | "I tested this by scaling to 0 and watching ArgoCD revert it in 3 minutes" |
| LAB-02 | Q30 Rollback | "I pushed a sha-0000bad tag and rolled back with git revert — ArgoCD deployed the previous image" |
| LAB-03 | Q28 Diagnose | "ImagePullBackOff — I described the pod and saw the exact image URL that failed" |
| LAB-04 | Q27, Q4, Q28 | "Deleted the DB secret, watched CrashLoopBackOff, forced ESO re-sync, pod recovered" |
| LAB-05 | Q27 ESO | "Pointed ExternalSecret at a wrong key name — got SecretSyncedError, read the error from describe" |
| LAB-06 | Q9 Password rotation | "Pushed wrong password to Secrets Manager, watched DB auth fail in Spring Boot logs, rotated back" |
| LAB-07 | Q5, Q6 Ingress | "Deleted the Ingress — ALB was deprovisioned in 3 minutes. ArgoCD re-created it via selfHeal" |
| LAB-08 | Q31 GitLeaks | "I added a fake AKIA key to a test file — GitLeaks failed the pipeline before Maven ran a single test" |
| LAB-09 | Q21 prune | "Disabled HPA in values file — ArgoCD pruned it from the cluster within 3 minutes of the git push" |
| LAB-10 | Q16, Q19 security | "Exec'd into the Nginx container and tried to write to the filesystem — got read-only file system error" |
| LAB-11 | Q19, Q28 probe | "Changed readiness probe to /BROKEN path — pod showed 0/1 Running, ALB returned 502 for /api/*" |
| LAB-12 | Q11 pipeline | "Created a fake .tflock file in S3 — Terraform failed with lock error. Removed it, plan succeeded" |
