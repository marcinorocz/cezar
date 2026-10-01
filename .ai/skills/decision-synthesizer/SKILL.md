---
name: decision-synthesizer
description: Compares code variants using the Immutable Evidence Snapshot Pattern to prevent TOCTOU race conditions and output structured 4-Act Decision Cards.
---

# 4-Act Decision Synthesizer (Immutable Evidence Snapshot)

## 1. Problem Statement
When arbitrating parallel variants or validating code outputs, background processes (formatters, compilers, concurrent worktree cleanups, or user "Pick Winner" clicks in `server.ts:5000`) continuously alter files on disk. Evaluating live files directly introduces severe Time-of-Check to Time-of-Use (TOCTOU) races and leads to inconsistent, corrupted recommendations or fatal `ENOENT` errors.

---

## 2. Hard Policy & Invariants
- **No Live-to-Live Comparisons:** The synthesizer NEVER compares "live file vs live file". All evaluations execute strictly as "snapshot vs frozen baseline" or "snapshot A vs snapshot B".
- **Immutable Evidence Artifact:** Before analysis begins, an atomic evidence snapshot is locked and persisted to disk (e.g. `.tmp/evidence-snapshot.json` or `.ai/cezar/groups/<groupId>/evidence-snapshot.json`).
- **Required Snapshot Schema:**
  - File path relative to workspace root
  - SHA-256 blob checksum
  - File size in bytes
  - ISO timestamp and mtime
  - List of input files and command execution records (exit codes, captured stdout)
- **"Evidence Stale" State:** If any document, contract, or repository HEAD changes after snapshot generation, the evaluation state transitions immediately to `EVIDENCE_STALE`. A decision card is valid ONLY when based on an uncorrupted, locked snapshot.
- **Safety Veto Invariant:** A security, PII, or data-integrity objection raised by any evaluator step CANNOT be overridden by majority vote; it triggers an immediate `SAFETY_ESCALATION` to human tech leads.

---

## 3. Decision Table (Snapshot Integrity & Actions)

| Comparison Condition | File State on Disk | Snapshot Hash Match | Engine Action | Decision Validity |
|---|---|:---:|:---:|---|
| **Clean Baseline** | Unchanged since snapshot | Exact match (`100%`) | **PROCEED** | Valid & Decisive |
| **Minor Non-Code Drift** | Ignored files (`.git/`, logs) changed | Target blobs match | **ALLOW WITH AUDIT** | Valid & Decisive |
| **Contract / Code Drift** | Touched files modified post-snapshot | Mismatch detected | **WARN & SET STALE** | `EVIDENCE_STALE` (Invalidated) |
| **Losing Worktree Deleted** | Directory unlinked via Pick/GC | Target file missing on live disk | **PROCEED FROM SNAPSHOT** | Valid (Snapshot isolated from FS deletion) |
| **Snapshot Hash Corrupted** | Snapshot file tampered or partial write | Checksum failure | **HARD BLOCK (`EVIDENCE_DRIFT`)** | Aborted immediately |
| **Safety Objection Raised** | Variant introduces security/PII hazard | Any | **ESCALATE TO LEAD** | Human sign-off required |

---

## 4. Operational Checklist & Pseudocode

### Synthesis Checklist
- [ ] 1. Assemble candidate files into a staging buffer in `.tmp/`.
- [ ] 2. Compute SHA-256 hashes, file sizes, and record test exit codes.
- [ ] 3. Atomically write and freeze snapshot to `.tmp/evidence-snapshot.json`.
- [ ] 4. Check for any drift during snapshot assembly. If drift occurred -> mark `EVIDENCE_STALE` and abort.
- [ ] 5. Run synthesizer evaluator tool-free against the frozen snapshot JSON payload.
- [ ] 6. If any blocking safety objections are detected -> flag `SAFETY_ESCALATION`.
- [ ] 7. Render 4-Act Decision Card with grounded metrics.

### Snapshot Generation Pattern
```ts
export interface FileEvidence {
  path: string;
  sha256: string;
  sizeBytes: number;
  mtime: number;
}

export interface EvidenceSnapshot {
  snapshotId: string;
  groupId: string;
  createdAt: string;
  inputRevisionId: string;
  files: readonly FileEvidence[];
  verificationRecords: readonly { command: string; exitCode: number }[];
}

export async function createImmutableSnapshot(
  groupId: string,
  targetPaths: string[]
): Promise<EvidenceSnapshot> {
  const fileEntries: FileEvidence[] = [];
  for (const relPath of targetPaths) {
    const content = await fs.promises.readFile(relPath);
    const hash = crypto.createHash('sha256').update(content).digest('hex');
    const stats = await fs.promises.stat(relPath);
    fileEntries.push({
      path: relPath,
      sha256: hash,
      sizeBytes: stats.size,
      mtime: stats.mtimeMs
    });
  }

  const snapshot: EvidenceSnapshot = {
    snapshotId: crypto.randomUUID(),
    groupId,
    createdAt: new Date().toISOString(),
    inputRevisionId: await getGitRevision(),
    files: Object.freeze(fileEntries),
    verificationRecords: await extractTestTranscripts(groupId)
  };

  const tmpPath = `.tmp/snapshot-${snapshot.snapshotId}.json`;
  const finalPath = `.ai/cezar/groups/${groupId}/evidence-snapshot.json`;
  await fs.promises.writeFile(tmpPath, JSON.stringify(snapshot, null, 2), 'utf-8');
  await fs.promises.rename(tmpPath, finalPath); // Atomic publish

  return snapshot;
}
```

---

## 5. The 4-Act Output Structure

### Act 1: Concrete Scenario
Problem context, affected module paths, and user story.

### Act 2: Value at Risk / Cost of Inaction
Architectural debt, maintenance risks, regressions, or safety hazards.

### Act 3: Trade-off Matrix
| Dimension | Variant A | Variant B |
|---|---|---|
| **Complexity & Maintainability** | Clean / Low | High |
| **Verification Evidence** | [EXECUTED: exit 0] | [REASONED only] |
| **Integrity & Drift Status** | Snapshot Verified | Mismatch (`EVIDENCE_STALE`) |

### Act 4: Decisive Recommendation with Grounded Metrics
- **Winner:** Winning Run ID (or **ESCALATE** if safety objection active).
- **Decisive Argument:** Clear, empirical reasoning.
- **Verification Metric:** Verified execution command output from snapshot transcript.
- **Provenance:** `snapshotId` and SHA-256 digest.
