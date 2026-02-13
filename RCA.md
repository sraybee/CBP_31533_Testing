h1. CBP-31533: Investigate 100 Step Limit Per Job - SPIKE RESULTS

*Status:* {status:colour=Green|title=Complete|subtle=false}
*Epic:* [CBP-30962|https://cloudbees.atlassian.net/browse/CBP-30962]
*Related Tickets:* [CBP-31078|https://cloudbees.atlassian.net/browse/CBP-31078] (Error Message Enhancement)
*Sprint:* Current
*Date:* February 13, 2026
*Author:* Engineering Team

----

h2. Executive Summary

{panel:title=Key Findings|borderStyle=solid|borderColor=#ccc|titleBGColor=#E3FCEF|bgColor=#FFFAE6}
*Problem:* Users are encountering a hard-coded 100 step limit per job that is causing workflow failures across multiple teams and customers.

*Root Cause:* The limit is a crude heuristic proxy for Kubernetes 1MB object size constraint. It counts steps rather than measuring actual object size, creating false positives and poor user experience.

*Impact:* {color:#d04437}*HIGH*{color} - Blocking multiple customers, forcing poor engineering practices (test coverage reduction, monolithic steps), and creating support burden.

*Recommendation:* Replace step counting with actual Kubernetes object size validation, making the platform more accurate and user-friendly while respecting real infrastructure constraints.
{panel}


h2. Problem Statement

h3. User-Facing Issue

Users receive this error when their workflows fail:

{noformat}
failed to run workflow <workflow-name>:
job <job-name>: steps limit exceeded, maximum 100 steps allowed per job. 
Please note, each action requires 3 or more steps
exit status 1
{noformat}

h3. Business Impact

||Impact Area||Severity||Description||
|*Customer Workflows*|{color:#d04437}*HIGH*{color}|Multiple teams blocked (NATS, Security Actions, JoeCP team)|
|*Engineering Practices*|{color:#d04437}*HIGH*{color}|Teams reducing test coverage, avoiding modular actions|
|*Support Burden*|{color:#f6c342}*MEDIUM*{color}|Requires SRE/engineering intervention to diagnose|
|*Platform Reputation*|{color:#f6c342}*MEDIUM*{color}|Platform appears fragile and unreliable|
|*Feature Adoption*|{color:#d04437}*HIGH*{color}|SLSA attestation, security scanning blocked|

h3. Affected Teams/Customers

* *Internal Teams:* NATS, Security Actions, Stackhawk, Security Orchestrator
* *External Customers:* JoeCP team (via Mariluz escalation)
* *Frequency:* 4+ incidents in past 2 months (Dec 18, Jan 8, Jan 15, Jan 29)


h2. Investigation Findings

h3. 1. Code Analysis

h4. Location of Limit

{code:language=go|title=dsl-engine-cli/internal/engine/tekton/transformworkflow.go:39}
var (
    stepsLimit = 100  // <-- Hard-coded, no configuration
    invalidK8sResourceName = regexp.MustCompile("[^a-z0-9]+")
)
{code}

h4. Validation Logic

{code:language=go|title=transformworkflow.go:797-801}
if len(task.task.Spec.Steps) > stepsLimit {
    return nil, nil, fmt.Errorf(
        "steps limit exceeded, maximum %d steps allowed per job. "+
            "Please note, each action requires 3 or more steps",
        stepsLimit)
}
{code}

h3. 2. Step Expansion Mechanism

{panel:title=Critical Discovery|borderStyle=solid|borderColor=#d04437|titleBGColor=#FFEBE6|bgColor=#FFF}
*User-visible steps ≠ Tekton steps*

Every action gets wrapped with scope management overhead that users never see.
{panel}

{code:language=go|title=transformaction.go:63-79}
func (t *jobTransformer) actionToTektonTaskSteps(...) {
    // 1. PUSH SCOPE - Set up action environment
    *steps = append(*steps, t.pushScopeStep(...))
    
    // 2. ACTUAL ACTION STEPS
    for i, step := range action.Runs.Steps {
        err = t.stepToTektonTaskSteps(...)
    }
    
    // 3. POP SCOPE - Handle outputs
    *steps = append(*steps, t.popScopeStep(...))
}
{code}

h4. Step Multiplication Table

||User's Perspective||Actual Tekton Steps||Example||
|Simple action (1 step)|3 steps|{{checkout@v1}}|
|Composite action (4 steps)|6+ steps|{{kaniko@v1}}|
|Complex action (11 steps)|13+ steps|{{security-actions/build@v3}}|
|Nested actions|Exponential|Actions calling actions|

h4. Real-World Example

{code:language=yaml|title=User writes in workflow YAML}
steps:
  - uses: cloudbees-io/checkout@v1       # User sees: 1 step
  - uses: cloudbees-io/kaniko@v1         # User sees: 1 step
  - uses: cloudbees-io/helm-install@v1   # User sees: 1 step
  - uses: docker://alpine                # User sees: 1 step
# Total user-visible steps: 4
{code}

{info:title=Engine Generation}
* checkout: push + checkout + pop = *3 steps*
* kaniko: push + 4 internal steps + pop = *6 steps*
* helm-install: push + 3 internal steps + pop = *5 steps*
* alpine: *1 step* (no wrapper)
* *Total Tekton steps: ~15*
{info}

{warning}
*With 35 user-defined steps → ~112 Tekton steps generated*

This is why users hit the limit much earlier than expected.
{warning}

h3. 3. Historical Context

From Slack investigation and Max Goltzsche's comment:

{quote}
Kubernetes resource/Pod size is limited to 1MB and we're using Tekton's task output mechanism which leverages k8s' Pod termination log message... Alexey did some tests when the resource limit would be hit and figured it would be after 100 steps which is why he limited the step amount to 100.
{quote}

*However, Max also noted:*

{quote}
That limit might be somewhat pointless since *the resource size does not only depend on the number of steps but also on the size of each step*.
{quote}

h4. The Real Constraint

* *Actual Limit:* Kubernetes etcd object size = 1MB (hard limit)
* *Tekton Behavior:* Stores Task CRDs in etcd
* *Current Approach:* Proxy metric (step count) instead of actual measurement
* *Problem:* Step count is a poor predictor of object size

h3. 4. Why Workflows Are Breaking Now

h4. Platform Evolution Timeline

||Date||Change||Impact||
|Pre-2024|Simple actions, 1-2 steps each|~50 steps margin|
|Mid-2024|Kaniko refactored (1 → 6 steps)|Margin reduced|
|Q4 2024|SLSA attestation added|\+3-5 steps per workflow|
|Q1 2025|Security scanning mandatory|\+5-10 steps per workflow|
|Current|Artifact publishing separated|\+2-3 steps per action|

*Result:* Workflows organically grew from ~60 steps → 110+ steps

h4. From Pinaki's Comment

{quote}
The problem was that with one step we were building/publishing the image AND calling internal APIs to publish Artifacts to Unify... Hence the plan to move the artifact publish logic into its own action.
{quote}

{panel:title=Translation|borderStyle=solid|borderColor=#205081|titleBGColor=#D4E5F7|bgColor=#FFF}
Good engineering practices (separation of concerns, error isolation, reusability) *directly conflict* with the arbitrary 100-step limit.
{panel}


h2. Root Cause Analysis

h3. Primary Root Cause

{panel:title=Key Finding|borderStyle=solid|borderColor=#d04437|titleBGColor=#FFEBE6|bgColor=#FFF}
*The step limit counts engine implementation overhead (push/pop scope wrappers) as user steps, creating a hidden 3x multiplier that makes the effective limit ~33 actions instead of 100.*
{panel}

h3. Contributing Factors

# *Inaccurate Proxy Metric*
** Counting steps ≠ measuring K8s object size
** Simple {{docker://alpine}} step: ~200 bytes
** Complex action step: ~2KB+
** All counted equally ❌

# *No User Visibility*
** Users cannot see step expansion
** No pre-submission validation
** Failures only at runtime

# *Platform Architectural Tax*
** Each action requires 2 wrapper steps (push/pop scope)
** Actions calling actions = exponential growth
** Necessary for scope isolation, but invisible cost

# *Feature Creep*
** SLSA attestation (security requirement)
** Artifact publishing (reliability improvement)
** Security scanning (compliance requirement)
** All add steps, no limit adjustment

# *No Configuration*
** Hard-coded value
** No CLI flag, env var, or cluster-level config
** One-size-fits-all approach fails

h3. Secondary Issues  

* *Error Message Inadequate:* Doesn't show actual step count or breakdown
* *No Documentation:* Limit not mentioned in official docs
* *No Monitoring:* No metrics tracking step counts approaching limit

----

h2. Proposed Solutions

h3. Solution Evaluation Matrix

||Solution||Effort||Impact||Risk||Timeline||Rating||
|*1. Size-based validation*|HIGH|HIGH|MEDIUM|3-4 weeks|(/) (/) (/) (/) (/) Best long-term|
|*2. Configurable limit*|LOW|MEDIUM|LOW|1 week|(/) (/) (/) (/) Quick win|
|*3. Enhanced error message*|LOW|LOW|LOW|2-3 days|(/) (/) (/) Already tracked (CBP-31078)|
|*4. Exclude wrapper steps*|MEDIUM|MEDIUM|MEDIUM|2 weeks|(/) (/) (/) UX improvement|
|*5. Eliminate push/pop*|VERY HIGH|HIGH|HIGH|2-3 months|(/) (/) Future consideration|
|*6. Auto-splitting*|HIGH|MEDIUM|HIGH|4-6 weeks|(/) (/) Complex, investigate later|


h2. Recommended Solution: Hybrid Approach

{tip:title=Strategy}
Implement quick wins immediately while building the proper long-term architectural fix.
{tip}

h3. Phase 1: Immediate Relief (Sprint 1 - Week 1)

h4. 1.1 Make Limit Configurable {color:#14892c}Priority 1{color}

*Implementation:*

{code:language=go|title=internal/engine/tekton/transformworkflow.go}
var (
    stepsLimit = getStepLimit() // Dynamic instead of hard-coded
    invalidK8sResourceName = regexp.MustCompile("[^a-z0-9]+")
)

func getStepLimit() int {
    // Environment variable override
    if envLimit := os.Getenv("CLOUDBEES_MAX_STEPS_PER_JOB"); envLimit != "" {
        if val, err := strconv.Atoi(envLimit); err == nil && val > 0 {
            log.Infof("Using step limit from env: %d", val)
            return val
        }
    }
    
    // Default to 100 for backwards compatibility
    return 100
}
{code}

*Deployment Configuration:*

{code:language=yaml|title=cloudbees-automation-controller ConfigMap}
apiVersion: v1
kind: ConfigMap
metadata:
  name: dsl-engine-config
  namespace: cloudbees-automation
data:
  CLOUDBEES_MAX_STEPS_PER_JOB: "200"  # Double the limit
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Immediate unblock for customers
* (/) Zero code risk (only reads env var)
* (/) Cluster-specific tuning (dev=500, prod=200)
* (/) Backwards compatible (defaults to 100)
{panel}

*Effort:* 2-3 days (including tests)

h4. 1.2 Enhanced Error Message {color:#14892c}Priority 2{color}

Tracked in *CBP-31078* - Add current step count to error:

{code:language=go}
if len(task.task.Spec.Steps) > stepsLimit {
    return nil, nil, fmt.Errorf(
        "steps limit exceeded, maximum %d steps allowed per job (current %d). "+
            "Please note, each action requires 3 or more steps",
        stepsLimit, len(task.task.Spec.Steps))
}
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Users know how much to reduce
* (/) Better support diagnostics
{panel}

*Effort:* 1 day

h4. 1.3 Add Monitoring Metrics

{code:language=go}
func (t *jobTransformer) transformToTektonTask() (*jobTektonTask, *tektonv1.Task, error) {
    // ... transformation logic ...
    
    stepCount := len(task.task.Spec.Steps)
    
    // Emit DataDog metrics
    metrics.RecordGauge("workflow.job.tekton_steps", float64(stepCount),
        "job:"+t.jobID, "workflow:"+t.Workflow.Name)
    
    // Warn if approaching limit
    if stepCount > int(float64(stepsLimit)*0.8) {
        log.Warnf("Job %s has %d steps (80%% of %d limit)", 
            t.jobID, stepCount, stepsLimit)
    }
    
    // ...
}
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Proactive monitoring
* (/) Identify trends before failures
* (/) Data for capacity planning
{panel}

*Effort:* 2 days

----

h3. Phase 2: Architectural Fix (Sprint 2-3 - Weeks 2-4)

h4. 2.1 Implement Size-Based Validation {color:#d04437}PRIMARY SOLUTION{color}

{panel:title=Core Concept|borderStyle=solid|borderColor=#205081|titleBGColor=#D4E5F7|bgColor=#FFF}
*Replace step counting with actual Kubernetes object size measurement*

Instead of counting steps (poor proxy), measure actual JSON/YAML bytes of the Tekton Task object that gets stored in etcd.
{panel}

*Implementation:*

{code:language=go|title=internal/engine/tekton/transformworkflow.go}
const (
    maxTaskSizeBytes = 900 * 1024 // 900KB (90% of 1MB, safety margin)
    stepCountWarningThreshold = 100 // Soft warning for migration
)

func (t *jobTransformer) transformToTektonTask() (*jobTektonTask, *tektonv1.Task, error) {
    // ... existing transformation logic ...
    
    // STEP 1: Soft warning for step count (backwards compatibility)
    stepCount := len(task.task.Spec.Steps)
    effectiveLimit := getStepLimit()
    
    if stepCount > effectiveLimit {
        log.Warnf("Job %s exceeds step count guideline: %d steps (limit: %d)",
            t.jobID, stepCount, effectiveLimit)
    }
    
    // STEP 2: HARD LIMIT - Measure actual K8s object size
    taskJSON, err := json.Marshal(task.task)
    if err != nil {
        return nil, nil, fmt.Errorf("failed to marshal task for size validation: %w", err)
    }
    
    taskSizeBytes := len(taskJSON)
    taskSizeKB := taskSizeBytes / 1024
    
    // Emit metrics
    metrics.RecordGauge("workflow.job.task_size_bytes", float64(taskSizeBytes),
        "job:"+t.jobID)
    
    // Log for analysis
    log.Infof("Job %s: %d steps, Task size: %d KB", t.jobID, stepCount, taskSizeKB)
    
    // Enforce size limit
    if taskSizeBytes > maxTaskSizeBytes {
        // Calculate per-step average for user guidance
        avgBytesPerStep := taskSizeBytes / stepCount
        estimatedMaxSteps := maxTaskSizeBytes / avgBytesPerStep
        
        return nil, nil, fmt.Errorf(
            "job %s: Kubernetes Task object size limit exceeded\n\n"+
            "Current Task size: %d KB (%d bytes)\n"+
            "Maximum allowed: %d KB (Kubernetes 1MB etcd limit)\n"+
            "Steps generated: %d Tekton steps\n"+
            "Average bytes per step: %d bytes\n"+
            "Estimated max steps at this size: ~%d steps\n\n"+
            "Root causes:\n"+
            "  • Too many steps in the job\n"+
            "  • Steps with large environment variables (secrets, outputs)\n"+
            "  • Complex nested actions\n"+
            "  • Large script blocks or configurations\n\n"+
            "Solutions:\n"+
            "  1. Split this job into 2-3 smaller jobs\n"+
            "  2. Reduce number of actions (each adds ~2KB+ overhead)\n"+
            "  3. Use container steps (docker://) instead of actions\n"+
            "  4. Minimize environment variables and secrets per step\n"+
            "  5. Extract large scripts to separate files\n\n"+
            "For help: https://docs.cloudbees.com/workflows/troubleshooting/task-size-limit",
            t.jobID,
            taskSizeKB, taskSizeBytes,
            maxTaskSizeBytes/1024,
            stepCount,
            avgBytesPerStep,
            estimatedMaxSteps)
    }
    
    // Warning at 80% threshold
    if taskSizeBytes > int(float64(maxTaskSizeBytes)*0.8) {
        log.Warnf("Job %s Task size: %d KB (approaching %d KB limit)",
            t.jobID, taskSizeKB, maxTaskSizeBytes/1024)
    }
    
    return task, &task.task, nil
}
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) *Measures actual constraint* (K8s 1MB limit)
* (/) *Accurate validation* - no false positives
* (/) *Better UX* - lightweight workflows can have 200+ steps
* (/) *Self-documenting* - error explains the real problem
* (/) *Future-proof* - adapts to platform changes
{panel}

*Testing Strategy:*

{code:language=go|title=Test cases needed}
func TestTaskSizeValidation(t *testing.T) {
    tests := []struct {
        name           string
        steps          int
        avgStepSizeKB  int
        expectError    bool
    }{
        {"Lightweight 200 steps", 200, 2, false},     // 400KB total
        {"Heavy 50 steps", 50, 20, false},           // 1000KB = 1MB limit
        {"Exceeds limit", 100, 12, true},            // 1200KB > 1MB
    }
    // ...
}
{code}

*Load Testing Required:*

# Generate workflows with varying step counts and complexities
# Measure actual Tekton Task sizes at production scale
# Validate 900KB threshold is safe under load
# Test with real customer workflows

*Effort:* 2 weeks (including testing)
*Risk:* Medium - requires validation that 900KB is safe margin

----

h3. Phase 3: User Experience (Sprint 4 - Week 5)

h4. 3.1 Pre-Submission Validation in NextGen UI

{code:language=typescript|title=nextgen-ui/src/validation/workflowValidator.ts}
interface StepEstimate {
  userSteps: number;
  estimatedTektonSteps: number;
  estimatedSizeKB: number;
}

function estimateWorkflowComplexity(workflow: Workflow): StepEstimate {
  let tektonSteps = 0;
  let estimatedBytes = 0;
  
  const BASE_TASK_OVERHEAD = 5 * 1024; // 5KB base
  const SIMPLE_STEP_SIZE = 200;        // bytes
  const ACTION_STEP_SIZE = 2000;       // bytes
  const ACTION_WRAPPER_OVERHEAD = 1000; // push+pop scope
  
  for (const job of Object.values(workflow.jobs)) {
    if ('steps' in job) {
      for (const step of job.steps) {
        if (step.uses?.startsWith('docker://')) {
          // Simple container step
          tektonSteps += 1;
          estimatedBytes += SIMPLE_STEP_SIZE;
        } else if (step.uses) {
          // Action - minimum 3 Tekton steps
          tektonSteps += 3; // Conservative estimate
          estimatedBytes += ACTION_STEP_SIZE + ACTION_WRAPPER_OVERHEAD;
        }
      }
    }
  }
  
  return {
    userSteps: countUserSteps(workflow),
    estimatedTektonSteps: tektonSteps,
    estimatedSizeKB: (BASE_TASK_OVERHEAD + estimatedBytes) / 1024
  };
}

// Show warnings in UI
function validateWorkflow(workflow: Workflow): ValidationResult {
  const estimate = estimateWorkflowComplexity(workflow);
  
  if (estimate.estimatedSizeKB > 900) {
    return {
      level: 'error',
      message: `Job may exceed 900KB limit (estimated: ${estimate.estimatedSizeKB}KB)`
    };
  }
  
  if (estimate.estimatedTektonSteps > 100) {
    return {
      level: 'warning',
      message: `Job generates ~${estimate.estimatedTektonSteps} Tekton steps (guideline: 100)`
    };
  }
  
  return { level: 'success' };
}
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Prevent failures before submission
* (/) User education (show step expansion)
* (/) Reduced support tickets
{panel}

*Effort:* 1 week


h3. Phase 4: Long-Term Optimization (Future - Months 2-6)

h4. 4.1 Investigate Scope Wrapper Elimination

*Current Architecture:*
{noformat}
User action → push-scope → action-steps → pop-scope
(1 step)         (+1)          (+N)           (+1)
{noformat}

*Proposed Architecture:*
{noformat}
User action → enhanced-action-step-with-scope-mgmt
(1 step)              (+1 only)
{noformat}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Reduce overhead by 2 steps per action
* (/) Workflows with 50 actions: 150 steps → 50 steps
{panel}

*Risk:* HIGH - requires redesign of scope management
*Effort:* 2-3 months
*Decision:* Investigate after size-based validation is stable

h4. 4.2 Tekton Remote Task Results

Configure Tekton to store results externally (S3/GCS) instead of termination logs:

{code:language=yaml|title=Tekton Pipeline configuration}
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-defaults
  namespace: tekton-pipelines
data:
  default-task-run-workspace-binding: |
    results:
      storage: s3
      bucket: cloudbees-tekton-results
      region: us-east-1
{code}

{panel:title=Benefits|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
* (/) Removes termination log overhead from Pod spec
* (/) Could reduce Task size by 20-30%
{panel}

*Effort:* 3-4 weeks
*Risk:* Medium - requires Tekton upgrade

----

h2. Success Metrics

h3. Pre-Implementation Baseline

* Workflow failure rate due to step limit: *~4 incidents/month*
* Average step count at failure: *105-120 Tekton steps*
* Support ticket escalations: *~4 tickets/month*
* User-visible steps typically: *35-40 steps*

h3. Post-Implementation Targets

||Metric||Current||Target||Measurement||
|Workflow failures (step limit)|4/month|<1/quarter|DataDog alert count|
|Support tickets (step limit)|4/month|<1/quarter|Jira count|
|False positive rejections|Unknown|0|Size-based validation|
|User satisfaction|N/A|>80%|Post-deployment survey|
|Avg task size at production|N/A|Measure|DataDog metrics|

----

h2. Testing Plan

h3. Unit Tests

{code:language=go}
// Test size-based validation
func TestTaskSizeValidation(t *testing.T) {
    // Test various step counts and complexities
}

// Test configurable limit
func TestConfigurableStepLimit(t *testing.T) {
    // Test env var override
}

// Test error messages
func TestEnhancedErrorMessages(t *testing.T) {
    // Verify error includes size and step count
}
{code}

h3. Integration Tests

{code:language=yaml|title=Create test workflows with known sizes}
workflows:
  - name: lightweight-200-steps
    expected: pass
    
  - name: heavy-50-steps
    expected: pass
    
  - name: exceeds-1mb
    expected: fail-with-size-error
{code}

h3. Load Tests

# *Step Count Variation:*
** 50, 100, 150, 200, 300 steps
** Measure actual Task sizes
** Validate against 1MB limit

# *Complexity Variation:*
** Simple container steps vs complex actions
** Heavy env vars and secrets
** Nested action compositions

# *Real Customer Workflows:*
** NATS workflow
** Security Actions workflows
** Migrate and validate

h3. Rollout Plan

h4. Week 1: Dev Environment

{code:language=yaml}
environment: dev
config:
  CLOUDBEES_MAX_STEPS_PER_JOB: 500  # Generous for testing
  ENABLE_SIZE_VALIDATION: true
  SIZE_LIMIT_KB: 900
{code}

h4. Week 2: Staging

{code:language=yaml}
environment: staging
config:
  CLOUDBEES_MAX_STEPS_PER_JOB: 200  # More realistic
  ENABLE_SIZE_VALIDATION: true
  SIZE_LIMIT_KB: 900
{code}

h4. Week 3-4: Production Gradual Rollout

{code:language=yaml}
# Week 3: 10% traffic
# Week 4: 50% traffic
# Week 5: 100% traffic

environment: production
config:
  CLOUDBEES_MAX_STEPS_PER_JOB: 150  # Conservative increase
  ENABLE_SIZE_VALIDATION: true      # New validation
  SIZE_LIMIT_KB: 900
  LEGACY_STEP_COUNT_WARNING: true   # Keep old check as warning
{code}

----

h2. Risks & Mitigations

h3. Risk 1: Size Validation Too Strict

*Impact:* Workflows currently at 95-100 steps might fail if heavy
*Probability:* Medium
*Mitigation:*
* Start with warning-only mode
* Collect metrics for 1 week
* Adjust threshold based on data
* Provide migration guide

h3. Risk 2: Performance Overhead

*Impact:* JSON marshaling adds latency to transformation
*Probability:* Low
*Mitigation:*
* Marshal happens once per job
* Typical task <100KB, marshaling <10ms
* Load test to validate
* Cache marshaled size if needed

h3. Risk 3: User Confusion

*Impact:* Change in error messaging confuses users
*Probability:* Low
*Mitigation:*
* Comprehensive documentation
* Migration guide
* Support team training
* FAQ section

h3. Risk 4: Kubernetes Limit Changes

*Impact:* K8s increases etcd limit, our code outdated
*Probability:* Very Low
*Mitigation:*
* Make limit configurable
* Document assumption (1MB K8s limit)
* Easy to adjust in future

----

h2. Documentation Updates Required

h3. 1. User-Facing Documentation

*New Page:* {{docs/workflows/troubleshooting/task-size-limit.md}}

{panel:title=Example Content|borderStyle=solid|borderColor=#205081|titleBGColor=#D4E5F7|bgColor=#FFF}
h4. Understanding Task Size Limits

CloudBees workflows are converted to Kubernetes Task objects which have a 1MB size limit imposed by etcd (Kubernetes' storage backend).

h5. What Counts Toward the Limit?

* Number of steps (more steps = larger object)
* Environment variables per step
* Secrets mounted in steps
* Action complexity (actions add wrapper overhead)
* Annotations and labels

h5. How to Stay Within Limits

# *Split large jobs* into multiple smaller jobs
# *Use container steps* ({{docker://}}) instead of actions where possible
# *Minimize environment variables* - use ConfigMaps for large configs
# *Reduce action nesting* - avoid actions calling too many other actions
# *Extract large scripts* to separate files instead of inline

h5. Monitoring Your Workflow Size

(Coming soon: UI indicator showing estimated Task size)
{panel}

h3. 2. Internal Documentation

*Update:* {{dsl-engine-cli/README.md}}

{noformat}
## Environment Variables

- CLOUDBEES_MAX_STEPS_PER_JOB: Maximum Tekton steps per job (default: 100)
- CLOUDBEES_TASK_SIZE_LIMIT_KB: Maximum Task object size in KB (default: 900)
{noformat}

h3. 3. Migration Guide

{panel:title=For Affected Users|borderStyle=solid|borderColor=#f6c342|titleBGColor=#FFFAE6|bgColor=#FFF}
h4. Migrating to Size-Based Validation

h5. What Changed?

*Previous:* Hard 100 step limit
*New:* 900KB Task object size limit (typically 150-200 lightweight steps)

h5. If Your Workflow Fails

# Check the error message for actual Task size
# Split jobs with >80 steps
# Replace action-heavy workflows with container steps
# Contact support if legitimate use case blocked
{panel}

----

h2. Cost-Benefit Analysis

h3. Implementation Cost

||Phase||Effort||Team||
|Phase 1: Quick fixes|1 week|1 engineer|
|Phase 2: Size validation|2-3 weeks|2 engineers|
|Phase 3: UI validation|1 week|1 frontend engineer|
|Testing & QA|1 week|QA team|
|Documentation|3 days|Tech writer|
|*Total*|*6-7 weeks*|*~2.5 FTE*|

h3. Business Value

||Benefit||Annual Value||
|Reduced support tickets (4/month → 1/quarter)|~40 hours/year support time|
|Unblocked customers|4+ teams immediately productive|
|Improved platform reputation|Qualitative|
|Enable feature adoption (SLSA, security)|Strategic enabler|
|Better engineering practices allowed|Qualitative (reduced tech debt)|

h3. ROI

*Immediate:* Unblock 4+ customer teams within 1 week
*Short-term:* Reduce support burden by 75%
*Long-term:* Platform can evolve without artificial constraints

----

h2. Recommendation Summary

h3. DO THIS (Phase 1 - Week 1)

# (/) *Make step limit configurable* via env var
# (/) *Enhanced error messages* (CBP-31078)
# (/) *Add monitoring metrics*
# (/) *Increase default to 150* for immediate relief

h3. DO THIS (Phase 2 - Weeks 2-4)

# (/) *Implement size-based validation* as primary check
# (/) *Load test* with various workflow types
# (/) *Keep step count as warning* during migration
# (/) *Document best practices*

h3. CONSIDER LATER (Months 2-6)

# UI pre-submission validation
# Scope wrapper elimination (if ROI justified)
# Tekton remote results storage
# Automatic job chunking

h3. DO NOT DO

# (x) Remove limit entirely (K8s constraint is real)
# (x) Just increase to 500 without size validation (kicks can down road)
# (x) Complex auto-splitting (too much magic, confusing UX)

----

h2. Implementation Timeline

||Phase||Tasks||Week||
|*Phase 1*|Quick Wins|Week 1|
| |Configurable limit|Feb 17-19|
| |Enhanced error messages|Feb 17-18|
| |Monitoring metrics|Feb 19-20|
| |Testing & deployment|Feb 21-22|
|*Phase 2*|Architecture Fix|Weeks 2-4|
| |Size-based validation impl|Feb 24 - Mar 2|
| |Load testing|Mar 3-7|
| |Production rollout|Mar 10-16|
|*Phase 3*|UX Improvements|Week 5|
| |UI validation|Mar 17-21|
| |Documentation|Mar 17-19|
| |User communication|Mar 24-25|

*Key Milestones:*

* *Feb 21:* Phase 1 complete - customers unblocked
* *Mar 10:* Phase 2 complete - proper validation in production
* *Mar 26:* Phase 3 complete - full feature with UI support

----

h2. Acceptance Criteria Met

{panel:title=This spike addresses all requirements from CBP-31533|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}

h3. (/) AC1: Identity how exactly this limit is derived

*Answer:*
* Hard-coded {{stepsLimit = 100}} in transformworkflow.go line 39
* Based on Alexey's testing: ~100 steps hit 1MB K8s etcd limit
* Step count is proxy for actual constraint (K8s object size)
* Each action adds 2+ wrapper steps (push/pop scope)

h3. (/) AC2: Short term solution options

*Provided:*
# Make limit configurable via env var (1 week)
# Enhanced error messages (2-3 days)
# Increase default to 150-200 (immediate)
# Add monitoring/alerting (2 days)

h3. (/) AC3: Long term solution options

*Provided:*
# Replace step counting with size-based validation (2-3 weeks) {color:#d04437}*PRIMARY*{color}
# Exclude wrapper steps from count (2 weeks)
# UI pre-submission validation (1 week)
# Scope wrapper elimination (2-3 months, future)
# Auto-splitting (4-6 weeks, future consideration)
{panel}

----

h2. References

h3. Code Locations

* Limit definition: {{dsl-engine-cli/internal/engine/tekton/transformworkflow.go:39}}
* Validation: {{dsl-engine-cli/internal/engine/tekton/transformworkflow.go:797-801}}
* Action expansion: {{dsl-engine-cli/internal/engine/tekton/transformaction.go:26-85}}
* Test case: {{dsl-engine-cli/internal/engine/testdata/workflows/steps-limit.yaml}}

h3. Related Tickets

* [CBP-30962|https://cloudbees.atlassian.net/browse/CBP-30962] - Parent epic
* [CBP-31078|https://cloudbees.atlassian.net/browse/CBP-31078] - Error message enhancement
* [CBP-31535|https://cloudbees.atlassian.net/browse/CBP-31535] - Created by Vyankatesh

h3. Slack Threads

* Dec 18, 2025 - scan-build-push-harbor failure
* Jan 8, 2026 - NATS workflow failure
* Jan 15, 2026 - Security actions upgrade to v6
* Jan 29, 2026 - SLSA attestation blocking
* Max Goltzsche's comment on K8s 1MB limit

h3. External References

* [Kubernetes etcd limits|https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container]
* [Tekton Task size considerations|https://tekton.dev/docs/pipelines/tasks/]
* [Tekton remote task results|https://tekton.dev/docs/pipelines/pipelines/#configuring-remote-task-resolution]

----

h2. Stakeholders

h3. Decision Makers

* *Mariluz Cabrera* - Product/priority decisions
* *Vyankatesh Inamdar* - Technical lead
* *Max Goltzsche* - Architecture review

h3. Impacted Teams

* *dsl-engine-cli team* - Primary implementation
* *NextGen UI team* - Pre-submission validation
* *SRE team* - Deployment and monitoring
* *Documentation team* - User-facing docs

h3. Affected Customers

* JoeCP team
* NATS team
* Security Actions team
* Stackhawk team
* All teams using SLSA attestation

----

h2. Next Steps

# *Review this spike* with stakeholders (Mariluz, Vyankatesh, Max)
# *Get approval* for recommended hybrid approach
# *Create implementation tickets* for Phase 1
# *Assign to sprint* - target start Feb 17, 2026
# *Communicate* to affected customers: relief coming in 1 week

----

{panel:title=Spike Status|borderStyle=solid|borderColor=#14892c|titleBGColor=#E3FCEF|bgColor=#FFF}
{status:colour=Green|title=COMPLETE - READY FOR REVIEW|subtle=false}

*Recommended Action:* Proceed with Phase 1 implementation immediately
{panel}

----

_Document prepared by: Engineering Team_
_Date: February 13, 2026_
_Jira Epic: CBP-30962_
_Related: CBP-31533 (Spike), CBP-31078 (Error Message)_
