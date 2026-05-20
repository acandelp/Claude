---
name: etcdperformance
description: Comprehensive etcd health and performance analysis in OpenShift with Confidence Scoring. Detects overload, performance issues, compaction issues, leader changes, and other critical problems. Presents detailed evidence + confidence score. Use when the user asks about etcd, overload, performance, etcd logs, heartbeat failures, compaction, or etcd storage issues.
---

# ETCD Comprehensive Health Check

Performs a comprehensive health and performance analysis of etcd in OpenShift, including detection of overload, compaction issues, leader changes, and other anomalies.

**Output**: The skill generates TWO sections:
1. Detailed evidence analysis (organized by severity: CRITICAL, WARNING, HEALTHY)
2. Confidence Score (diagnostic confidence evaluation with 0.0-1.0 scoring)

## Steps

### 1. Initial Verification

Verify active session and cluster status:
```bash
oc whoami
```
If it fails, indicate that they must log in with `oc login` before continuing.

Get basic information about etcd pods:
```bash
echo "=== ETCD PODS STATUS ==="
oc get pods -n openshift-etcd -l app=etcd -o wide
echo ""
echo "=== ETCD POD AGES ==="
oc get pods -n openshift-etcd -l app=etcd -o custom-columns=NAME:.metadata.name,AGE:.metadata.creationTimestamp
```

### 2. Log Analysis - Critical Patterns

Analyze all known problem patterns in etcd:

```bash
echo "=== ETCD DIAGNOSTIC SUMMARY ==="
echo ""

# Count different types of problematic messages
OVERLOAD_NETWORK=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "overloaded network" 2>/dev/null || echo "0")
OVERLOAD_SERVER=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "server is likely overloaded" 2>/dev/null || echo "0")
HEARTBEAT_FAIL=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "failed to send out heartbeat" 2>/dev/null || echo "0")
TOOK_TOO_LONG=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "took too long" 2>/dev/null || echo "0")
LEADER_CHANGES=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "leader changed" 2>/dev/null || echo "0")
DB_SPACE=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "database space exceeded" 2>/dev/null || echo "0")
CLOCK_DRIFT=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -E -c "clock difference|clock-drift" 2>/dev/null || echo "0")
COMPACTION=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "compaction" 2>/dev/null || echo "0")

echo "Issue Type                          | Count"
echo "----------------------------------- | --------"
echo "Overloaded Network (buffer full)    | $OVERLOAD_NETWORK"
echo "Server Likely Overloaded (slow disk)| $OVERLOAD_SERVER"
echo "Heartbeat Failures                  | $HEARTBEAT_FAIL"
echo "Took Too Long (>50ms expected)      | $TOOK_TOO_LONG"
echo "Leader Changes                      | $LEADER_CHANGES"
echo "Database Space Exceeded             | $DB_SPACE"
echo "Clock Drift / NTP Issues            | $CLOCK_DRIFT"
echo "Compaction Messages                 | $COMPACTION"
```

### 3. Distribution by Pod

Identify which specific pods have issues:
```bash
echo ""
echo "=== DISTRIBUTION BY POD ==="
echo ""
for pod in $(oc get pods -n openshift-etcd -l app=etcd -o name); do
  echo "Pod: $pod"
  overload_net=$(oc logs -n openshift-etcd "$pod" --all-containers --tail=-1 2>/dev/null | grep -c "overloaded network" 2>/dev/null || echo "0")
  overload_srv=$(oc logs -n openshift-etcd "$pod" --all-containers --tail=-1 2>/dev/null | grep -c "server is likely overloaded" 2>/dev/null || echo "0")
  heartbeat=$(oc logs -n openshift-etcd "$pod" --all-containers --tail=-1 2>/dev/null | grep -c "failed to send out heartbeat" 2>/dev/null || echo "0")
  leader=$(oc logs -n openshift-etcd "$pod" --all-containers --tail=-1 2>/dev/null | grep -c "leader changed" 2>/dev/null || echo "0")
  echo "  - Overloaded Network: $overload_net"
  echo "  - Server Overloaded: $overload_srv"
  echo "  - Heartbeat Failures: $heartbeat"
  echo "  - Leader Changes: $leader"
  echo ""
done
```

### 4. Temporal Analysis

Show event timeline to identify patterns:
```bash
echo "=== TIMELINE ANALYSIS ==="
echo ""
echo "Overload messages by date:"
oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
  | grep -i "overload" \
  | jq -r '.ts' 2>/dev/null \
  | awk -F'T' '{print $1}' \
  | sort | uniq -c | sort -rn | head -10

echo ""
echo "Leader changes by date:"
oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
  | grep "leader changed" \
  | jq -r '.ts' 2>/dev/null \
  | awk -F'T' '{print $1}' \
  | sort | uniq -c | sort -rn | head -10
```

### 5. Compaction Performance Analysis

Analyze compaction times (critical for performance):
```bash
echo ""
echo "=== COMPACTION PERFORMANCE ==="
echo ""
echo "Extracting compaction times (expected: <100ms for HDD, <10ms for SSD/NVMe)..."
echo ""

# Extraer tiempos de compaction y mostrar estadísticas
oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
  | grep "compaction" \
  | grep -v "downgrade" \
  | jq -r 'select(.msg | contains("finished scheduled compaction")) | .took' 2>/dev/null \
  | sed 's/"//g' \
  | awk '{
      if ($0 ~ /ms$/) { 
        gsub(/ms$/, "", $0); 
        ms = $0 + 0 
      } else if ($0 ~ /s$/) { 
        gsub(/s$/, "", $0); 
        ms = $0 * 1000 
      } else { 
        next 
      }
      sum += ms; 
      count++; 
      if (ms > max) max = ms; 
      if (min == 0 || ms < min) min = ms;
      if (ms > 300) critical++;
      if (ms > 100 && ms <= 300) warning++;
    } 
    END {
      if (count > 0) {
        avg = sum/count;
        print "Total compactions: " count;
        print "Average time: " avg " ms";
        print "Min time: " min " ms";
        print "Max time: " max " ms";
        print "WARNING (>100ms): " warning;
        print "CRITICAL (>300ms): " critical;
      } else {
        print "No compaction data found in logs"
      }
    }'
```

### 6. Heartbeat Failures with Exceeded Duration

If there are heartbeat failures, show the worst cases:
```bash
echo ""
echo "=== TOP 10 HEARTBEAT FAILURES (by exceeded-duration) ==="
echo ""

exceeded_count=$(oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep -c "exceeded-duration" 2>/dev/null || echo "0")

if [ "$exceeded_count" -gt "0" ]; then
  echo "Timestamp                        | Destino           | Expected | Exceeded"
  echo "-------------------------------- | ----------------- | -------- | ----------"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
    | grep "exceeded-duration" \
    | jq -r '[.ts, .to, ."expected-duration", ."exceeded-duration"] | @tsv' \
    | awk -F'\t' '{
        d = $4
        if (d ~ /ms$/) { gsub(/ms$/, "", d); ms = d + 0 }
        else if (d ~ /s$/)  { gsub(/s$/, "", d); ms = d * 1000 }
        else { ms = 0 }
        print ms "\t" $0
      }' \
    | sort -rn \
    | head -10 \
    | cut -f2- \
    | awk -F'\t' '{printf "%s | %s | %s | %s\n", $1, $2, $3, $4}'
else
  echo "No heartbeat failures with exceeded-duration found"
fi
```

### 7. Database Space Issues

If there are space issues, show details:
```bash
echo ""
if [ "$DB_SPACE" -gt "0" ]; then
  echo "=== DATABASE SPACE EXCEEDED DETAILS ==="
  echo ""
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
    | grep "database space exceeded" \
    | jq -r '[.ts, .msg] | @tsv' \
    | head -10
  echo ""
  echo "⚠️  CRITICAL: Database space exceeded requires immediate defragmentation!"
fi
```

### 8. Leader Changes Analysis

If there are multiple leader changes, investigate:
```bash
echo ""
if [ "$LEADER_CHANGES" -gt "10" ]; then
  echo "=== LEADER CHANGES ANALYSIS ==="
  echo ""
  echo "⚠️  WARNING: $LEADER_CHANGES leader changes detected (expected: minimal)"
  echo ""
  echo "Recent leader changes:"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
    | grep "leader changed" \
    | jq -r '[.ts, .msg] | @tsv' \
    | tail -5
  echo ""
  echo "Frequent leader changes indicate performance or network issues between etcd members"
fi
```

### 9. Clock Drift / NTP Issues

If there are clock issues, alert:
```bash
echo ""
if [ "$CLOCK_DRIFT" -gt "0" ]; then
  echo "=== CLOCK DRIFT / NTP ISSUES ==="
  echo ""
  echo "⚠️  CRITICAL: Clock synchronization issues detected!"
  echo ""
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
    | grep -E "clock difference|clock-drift" \
    | jq -r '[.ts, .msg] | @tsv' \
    | head -10
  echo ""
  echo "Recommendation: Verify NTP synchronization on all master nodes with 'chronyc sources' and 'chronyc tracking'"
fi
```

### 10. Last Occurrences

Show the latest messages for each problem type:
```bash
echo ""
echo "=== LAST OCCURRENCES ==="
echo ""

if [ "$OVERLOAD_NETWORK" -gt "0" ]; then
  echo "Last Overloaded Network:"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep "overloaded network" | tail -1 | jq -r '[.ts, .msg] | @tsv'
  echo ""
fi

if [ "$OVERLOAD_SERVER" -gt "0" ]; then
  echo "Last Server Overloaded:"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep "server is likely overloaded" | tail -1 | jq -r '[.ts, .msg] | @tsv'
  echo ""
fi

if [ "$LEADER_CHANGES" -gt "0" ]; then
  echo "Last Leader Change:"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep "leader changed" | tail -1 | jq -r '[.ts, .msg] | @tsv'
  echo ""
fi

if [ "$TOOK_TOO_LONG" -gt "0" ]; then
  echo "Last 'Took Too Long' message:"
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null | grep "took too long" | tail -1 | jq -r '[.ts, .msg, .took] | @tsv'
  echo ""
fi
```

### 11. Detailed "Took Too Long" Analysis

If there are "took too long" messages, provide detailed analysis:
```bash
if [ "$TOOK_TOO_LONG" -gt "1000" ]; then
  echo ""
  echo "=== 'TOOK TOO LONG' DETAILED ANALYSIS ==="
  echo ""
  echo "Analyzing apply request times ($TOOK_TOO_LONG messages found)..."
  echo ""

  # Analyze "took too long" times
  oc logs -n openshift-etcd -l app=etcd --all-containers --tail=-1 2>/dev/null \
    | grep "took too long" \
    | jq -r 'select(.took != null) | .took' 2>/dev/null \
    | sed 's/"//g' \
    | awk '{
        if ($0 ~ /ms$/) { 
          gsub(/ms$/, "", $0); 
          ms = $0 + 0 
        } else if ($0 ~ /s$/) { 
          gsub(/s$/, "", $0); 
          ms = $0 * 1000 
        } else { 
          next 
        }
        sum += ms; 
        count++; 
        if (ms > max) max = ms; 
        if (min == 0 || ms < min) min = ms;
        if (ms > 1000) critical++;
        if (ms > 500 && ms <= 1000) warning++;
      } 
      END {
        if (count > 0) {
          avg = sum/count;
          print "Total messages with timing: " count;
          print "Average time: " avg " ms";
          print "Min time: " min " ms";
          print "Max time: " max " ms";
          print "WARNING (>500ms): " warning;
          print "CRITICAL (>1000ms): " critical;
        } else {
          print "No timing data found"
        }
      }'
fi
```

## Results Interpretation

### Problem Severity

**CRITICAL (Requires immediate action):**
- `database space exceeded` → Execute defragmentation immediately
- `clock drift` > 0 → Verify and correct NTP on all masters
- Compaction times > 300ms → Serious storage problems, investigate disk I/O

**WARNING (Requires investigation):**
- Overload messages > 1000 → Investigate root cause (CPU, disk, network)
- Leader changes > 10 → Performance or network issues between members
- Compaction times > 100ms → Slow storage, consider upgrade to SSD/NVMe
- Heartbeat failures > 50 → Slow disk or CPU overload

**INFO (Monitor):**
- Overload messages < 100 → Temporary spikes, monitor trend
- Leader changes < 10 → Normal in some maintenance scenarios

### Performance Thresholds

- **Compaction time**: <10ms (SSD/NVMe), <100ms (HDD), >300ms (CRITICAL)
- **Heartbeat expected**: <50ms (normal), >50ms indicates slow disk
- **API Latency**: 2-5ms expected
- **Object count in etcd**: >8000 objects can cause degradation
- **Secrets per namespace**: >20 should be cleaned up

## Additional Useful Commands

### Count objects in etcd (requires rsh to pod):
```bash
# IMPORTANT: Only execute if the user explicitly requests it
# oc rsh -n openshift-etcd etcd-<pod-name>
# etcdctl get / --prefix --keys-only | sed '/^$/d' | cut -d/ -f3 | sort | uniq -c | sort -rn
```

### View secrets by namespace:
```bash
oc get secrets -A --no-headers | awk '{ns[$1]++}END{for (i in ns) print i,ns[i]}' | sort -k2 -rn | head -20
```

## Results Presentation

**CRITICAL**: After running ALL diagnostic commands (steps 1-11), present the results in TWO main sections:

1. **First**: Detailed Evidence Analysis (organized by severity)
2. **Second**: Confidence Score (with detailed calculation)

DO NOT present only the confidence score. Both sections are mandatory.

## Final Output Format

**IMPORTANT**: The analysis must be presented in TWO main sections:

### SECTION 1: Detailed Evidence Analysis

First present a complete analysis with evidence organized by severity.

**IMPORTANT**: Use this EXACT format (without long decorative lines):

```
📊 ETCD COMPREHENSIVE HEALTH ANALYSIS - CLUSTER <cluster-name>

🔴 CRITICAL FINDINGS

1. SEVERE STORAGE PERFORMANCE ISSUES
   - Apply Request Latency: Average Xms (expected <100ms)
     - MAX latency: X SECONDS 🔴
     - X requests took >1 second (CRITICAL)
     - X requests took >500ms (WARNING)
     - X total "took too long" messages
   
2. COMPACTION PERFORMANCE DEGRADED
   - Average compaction time: Xms (expected <10ms for SSD, <100ms for HDD)
     - MIN: Xms (even best case is slow!)
     - MAX: Xms
     - X compactions >100ms (WARNING)
     - X compactions >300ms (CRITICAL)

3. PROBLEM CONCENTRATED ON MASTER-X
   - ALL issues are on pod etcd-<cluster>-master-X:
     - X "overloaded network" messages
     - X heartbeat failures
     - X leader changes
   - Master-1 and Master-2 are healthy

⚠️  WARNING FINDINGS

4. LEADER CHANGES
   - X leader changes detected (expected: minimal)
   - All leader changes due to "apply request took too long"
   - Pattern correlates with storage performance issues

5. RECENT AND ONGOING ISSUE
   Timeline of "took too long" messages:
   May XX: X,XXX messages
   May XX: X,XXX messages
   May XX: X,XXX messages (TODAY - ongoing!)
   
   Last occurrence: Today at HH:MM (took X.XXX seconds)

✓ HEALTHY METRICS

- No database space issues
- No clock drift / NTP issues
- Secrets distribution reasonable (max X per namespace)
- Pods stable (X days old, only expected restarts)

---
🎯 ROOT CAUSE ANALYSIS

Primary Issue: [Primary diagnosis based on evidence]

Evidence:
1. [Specific quantitative evidence]
2. [Log/message evidence]
3. [Temporal correlation evidence]
4. [Problem localization evidence]
5. [Any other relevant evidence]

---
🔧 IMMEDIATE ACTIONS REQUIRED

Priority 1: [Main action title]

[Brief action description]

# Exact commands to execute
<commands with comments>

Priority 2: [Second action title]

[Procedure or considerations]
# Commands if applicable
<commands>
```

**Format rules**:
- DO NOT use long decorative lines (━━━━)
- Use simple separators with --- between main sections
- Indentation with spaces (2 spaces per level)
- Emojis at the start of each main section
- Commands with inline comments (#)
- Readable metrics format (commas in thousands: 4,264 not 4264)

### SECTION 2: Confidence Score

After the evidence analysis, ALWAYS include the complete confidence scoring.

**IMPORTANT**: This section DOES use long decorative lines (━━━━) to give formal formatting to the scoring.

Exact format for Section 2:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 ETCD DIAGNOSTIC CONFIDENCE SCORE CALCULATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

COLLECTED EVIDENCE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[List of collected evidence with metrics]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONFIDENCE SCORE BREAKDOWN:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Factor 1: Root Cause Clarity                              +0.XX
[Justification]

Factor 2: Quantitative Evidence                           +0.XX
[Justification]

Factor 3: Temporal Correlation                            +0.XX
[Justification]

Factor 4: Fix Documentation Available                     +0.XX
[Justification]
                                                          ------
TOTAL CONFIDENCE SCORE:                                    X.XX

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 ASSESSMENT: X.XX (Level Description) 🟢/🟡/🟠/🔴
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

INTERPRETATION:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Diagnostic and confidence level interpretation]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 RECOMMENDED ACTIONS (Based on Confidence Level):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Recommended actions based on the score]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONFIDENCE LEVEL JUSTIFICATION:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Score justification, why this value and how to improve it]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 REFERENCES:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[References and links]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Recommendations Based on Findings

When generating recommendations, follow these guidelines based on findings:

1. **If there are >100 overload messages**: Investigate disk I/O, CPU, and network on the affected node
2. **If there are frequent leader changes**: Review network connectivity between masters
3. **If compaction >300ms**: Critical storage, consider migrating to SSD/NVMe or investigate I/O bottleneck
4. **If database space exceeded**: Execute defragmentation following official procedure
5. **If clock drift detected**: Synchronize NTP on all masters immediately
6. **If problems concentrated on one pod**: Investigate the specific node (hardware, network, storage)

### Severity Classification for Analysis

**CRITICAL (🔴 - Requires immediate action):**
- `database space exceeded` → Execute defragmentation immediately
- `clock drift` > 0 → Verify and correct NTP on all masters
- Compaction times > 300ms → Serious storage problems, investigate disk I/O
- Apply request latency > 1000ms (1s) → Severe storage bottleneck
- Heartbeat failures with explicit "slow disk" message

**WARNING (⚠️ - Requires investigation):**
- Overload messages > 1000 → Investigate root cause (CPU, disk, network)
- Leader changes > 10 → Performance or network issues between members
- Compaction times > 100ms → Slow storage, consider upgrade to SSD/NVMe
- Heartbeat failures > 50 → Slow disk or CPU overload
- Apply request latency > 500ms

**INFO (✓ - Monitor):**
- Overload messages < 100 → Temporary spikes, monitor trend
- Leader changes < 10 → Normal in some maintenance scenarios
- Compaction times < 100ms → Acceptable performance
- Stable pods without abnormal restarts

## Confidence Scoring

**IMPORTANT**: At the end of the analysis, ALWAYS calculate and present a Confidence Score based on the collected evidence.

### Diagnostic Confidence Scale

| Score | Level | Meaning | Typical Evidence |
|-------|-------|-------------|------------------|
| **0.95-1.0** | 🟢 Near-certain match | Nearly confirmed diagnosis | Exact symptoms + version matches verified KCS + documented fix + cluster data confirms root cause |
| **0.80-0.94** | 🟢 Strong match | Strong match | Symptoms + version match known issue + fix exists and applies + evidence passes all validations |
| **0.60-0.79** | 🟡 Probable direction | Probable direction | Related issues/KCS with good version/environment match, but with some caveats. Multiple plausible causes |
| **0.40-0.59** | 🟡 Leads to pursue | Leads to pursue | Contextually relevant information but not direct match. Plausible but unconfirmed hypotheses |
| **0.20-0.39** | 🟠 Thin evidence | Thin evidence | Minimally relevant results. Tangential matches. Requires must-gather, additional logs or cluster access |
| **0.00-0.19** | 🔴 No actionable evidence | No actionable evidence | Searches returned nothing useful. Novel issue or insufficient public documentation. Recommend escalation or deep log analysis |

### Scoring Criteria for ETCD

Calculate the score based on the following factors:

#### Factor 1: Root Cause Clarity (+0.0 to +0.40)
- **+0.40**: Explicit root cause message (e.g.: "slow disk", "database space exceeded")
- **+0.30**: Multiple correlated symptoms pointing to same cause
- **+0.20**: Symptoms present but ambiguous cause
- **+0.10**: Symptoms detected without clear pattern
- **+0.00**: No significant symptoms

#### Factor 2: Quantitative Evidence (+0.0 to +0.30)
- **+0.30**: >1000 error messages + performance metrics confirming issue
- **+0.20**: 100-1000 messages + metrics available
- **+0.10**: <100 messages or limited metrics
- **+0.00**: No quantitative evidence

#### Factor 3: Temporal Correlation (+0.0 to +0.15)
- **+0.15**: Clear temporal pattern (e.g.: concentrated on specific dates)
- **+0.10**: Identifiable trend
- **+0.05**: Scattered events
- **+0.00**: No temporal pattern

#### Factor 4: Documentation/Fix Available (+0.0 to +0.15)
- **+0.15**: KCS/official documentation with verified fix for exact symptoms
- **+0.10**: Related documentation with applicable procedures
- **+0.05**: Generic references without specific fix
- **+0.00**: No relevant documentation

### Calculation Example

```bash
# After collecting all data, calculate score

SCORE=0.00

# Factor 1: Root Cause
if (HEARTBEAT_FAIL > 0 and message contains "slow disk") or (DB_SPACE > 0); then
  SCORE += 0.40
elif (COMPACTION_CRITICAL > 10 and TOOK_TOO_LONG > 1000); then
  SCORE += 0.30
elif (OVERLOAD_NETWORK > 100 or LEADER_CHANGES > 10); then
  SCORE += 0.20
fi

# Factor 2: Quantitative Evidence
TOTAL_ISSUES=$((OVERLOAD_NETWORK + HEARTBEAT_FAIL + TOOK_TOO_LONG + LEADER_CHANGES))
if [ $TOTAL_ISSUES -gt 1000 ]; then
  SCORE += 0.30
elif [ $TOTAL_ISSUES -gt 100 ]; then
  SCORE += 0.20
elif [ $TOTAL_ISSUES -gt 10 ]; then
  SCORE += 0.10
fi

# Factor 3: Temporal Correlation
if (events concentrated in 1-2 specific dates); then
  SCORE += 0.15
elif (growing trend or identifiable pattern); then
  SCORE += 0.10
fi

# Factor 4: Fix Available
if (DB_SPACE > 0 or CLOCK_DRIFT > 0); then
  # Official documented procedures exist
  SCORE += 0.15
elif (COMPACTION_CRITICAL > 0 or HEARTBEAT_FAIL > 0); then
  # Troubleshooting guides exist
  SCORE += 0.10
fi
```

### Score Presentation

At the end of the analysis, present the score in this format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 CONFIDENCE SCORE: 0.85 (Strong Match) 🟢
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Root Cause Clarity:        +0.40  (Explicit "slow disk" message)
Quantitative Evidence:     +0.30  (14,853 messages + metrics)
Temporal Correlation:      +0.15  (Clear pattern May 16-19)
Fix Documentation:         +0.10  (Official troubleshooting guides)
                          ------
TOTAL CONFIDENCE:           0.95

INTERPRETATION:
✓ Near-certain match - Slow disk I/O on master-0 confirmed
✓ Symptoms match documented patterns for storage bottleneck
✓ Fix path clear: investigate node storage, consider replacement
✓ Live cluster data strongly supports diagnosis

RECOMMENDED ACTION:
Proceed with Priority 1 actions with high confidence. Storage 
investigation on master-0 will likely confirm root cause.
```

### Important Notes

1. **Transparency**: Always show the score breakdown, not just the final number
2. **Score Evolution**: If must-gather or additional analysis is required, indicate how the score could improve
3. **Escalation**: If score <0.40, explicitly recommend escalation or complete must-gather
4. **Validation**: If score >0.80 but live validation is missing, mention it explicitly

### Score Evolution Guidelines

Indicate to the user how to improve the score if in medium/low ranges:

- **Score 0.20-0.39**: "We need complete must-gather, node logs, or cluster access to validate hypotheses"
- **Score 0.40-0.59**: "Collecting Prometheus metrics, reviewing cluster events, or running specific diagnostics would strengthen the diagnosis"
- **Score 0.60-0.79**: "Live validation (e.g.: iostat on affected node, etcd metrics) would confirm the diagnosis"
- **Score 0.80+**: "High confidence - proceed with recommended actions"

## FINAL REMINDER - Mandatory Output

**CRITICAL**: When finishing the analysis, ALWAYS generate both sections in this order:

### 1. SECTION 1 - Detailed Evidence Analysis (SIMPLE format, without decorative lines):

Structure:
```
📊 ETCD COMPREHENSIVE HEALTH ANALYSIS - CLUSTER <name>

🔴 CRITICAL FINDINGS
[Critical findings with metrics]

⚠️ WARNING FINDINGS
[Warning findings]

✓ HEALTHY METRICS
[Healthy metrics]

---
🎯 ROOT CAUSE ANALYSIS
[Analysis with numbered evidence]

---
🔧 IMMEDIATE ACTIONS REQUIRED
Priority 1: [action]
Priority 2: [action]
```

**Characteristics**:
- DO NOT use long decorative lines (━━━━)
- Simple separators with ---
- Indentation with spaces
- Commands with inline comments

### 2. SECTION 2 - Confidence Score (FORMAL format, WITH decorative lines):

Structure:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 ETCD DIAGNOSTIC CONFIDENCE SCORE CALCULATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

COLLECTED EVIDENCE:
[Collected evidence]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONFIDENCE SCORE BREAKDOWN:
[Factors + scoring]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 ASSESSMENT: X.XX (Level) 🟢
[Interpretation]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 RECOMMENDED ACTIONS:
[Actions based on score]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONFIDENCE LEVEL JUSTIFICATION:
[Why this score]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 REFERENCES:
[Links]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Characteristics**:
- DO use long decorative lines (━━━━)
- Formal and structured format
- Detailed scoring breakdown

DO **NOT** present only one of the sections. Both are mandatory for a complete analysis.

## References

- Red Hat KB: https://access.redhat.com/articles/6271341
- Metrics Collection: https://access.redhat.com/solutions/5489721
- Based on: https://github.com/peterducai/openshift-etcd-suite

## Common Errors

- **`oc: command not found`** → The `oc` client is not installed or not in PATH
- **`Error from server (Forbidden)`** → The user doesn't have permissions to read logs in `openshift-etcd`. Needs `cluster-reader` permissions or higher
- **`jq: command not found`** → Install jq for JSON parsing: `brew install jq` (macOS) or `dnf install jq` (RHEL/Fedora)
