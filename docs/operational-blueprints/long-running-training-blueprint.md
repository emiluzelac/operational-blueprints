# Long-Running Model Training Operations Blueprint

## 1. Core Mission

Operate long-running model training, fine-tuning, evaluation, conversion, and data-processing jobs safely, methodically, and reproducibly.

Launching a process is not completion.

A job is complete only when it reaches a verified terminal state:

1. Successful completion with expected outputs verified.
2. Safe failure with logs and diagnostic evidence preserved.
3. Controlled stop caused by a predefined safety threshold.
4. Escalation because human judgment or authorization is required.

Do not claim that a job is being monitored unless an active agent loop, scheduler, watchdog, or process supervisor is actually continuing to inspect it.

Do not claim that work will continue after the current agent session ends unless an independent process has been installed and verified to continue that work.

---

# 2. Operating Principles

## 2.1 Verify Before Acting

Never rely solely on assumptions about the machine, training environment, dataset, checkpoint, or previous process state.

Before launching a job, verify:

- Hostname and intended machine.
- System uptime.
- Available RAM.
- Current swap use.
- Available disk space.
- GPU identity.
- GPU utilization.
- GPU memory use.
- Running training, inference, evaluation, or model-server processes.
- Existing output directories.
- Python environment.
- Required package versions.
- Base-model path.
- Dataset path.
- Dataset integrity.
- Checkpoint path.
- Log path.
- Command-line arguments.
- Whether an earlier run survived a reboot, disconnect, or failed session.

Do not terminate an existing process merely because it resembles the target process. Identify the exact process, owner, command, process ID, and start time first.

---

## 2.2 Start Small Before Scaling

Never launch a full training sweep immediately when the configuration has not been validated on the target hardware.

Use the following progression:

1. Static configuration audit.
2. Dataset and token-length audit.
3. One-batch or one-microbatch test.
4. One complete optimizer-step smoke test.
5. Checkpoint-write test.
6. Short bounded training run.
7. One full candidate.
8. Additional candidates only after the previous stage passes.

A smoke test must exercise the real:

- Model.
- Tokenizer.
- Dataset format.
- Collator.
- Precision mode.
- Maximum sequence length.
- Batch size.
- Gradient accumulation.
- Optimizer.
- Checkpoint writer.
- Output filesystem.

A script that only loads the model is not a sufficient training smoke test.

---

## 2.3 Use the Minimum Necessary Scope

Operate only on the intended host, repository, model, dataset, process, and output path.

Never use broad commands such as:

- Killing every Python process.
- Killing every GPU process.
- Killing all processes matching an imprecise name.
- Removing an entire shared temporary directory.
- Deleting all checkpoints.
- Restarting unrelated services.
- Rebooting the host without explicit authorization.

Target the exact process ID or process group belonging to the current job.

Preserve unrelated model servers, agents, containers, notebooks, and user services.

---

## 2.4 Keep the Active Run Immutable

Once a real training candidate has started, record and freeze:

- Dataset hash.
- Training-script hash.
- Configuration.
- Base-model hash or revision.
- Tokenizer revision.
- Seed.
- Output directory.
- Environment information.
- Dependency versions.
- Launch command.
- Start time.

Do not silently replace or edit files consumed by the active process.

Improvements discovered during the run may be prepared for future candidates, but they must not be represented as part of the currently running candidate.

Clearly distinguish:

- Active-run configuration.
- Future-run configuration.
- Evaluation-only changes.
- Infrastructure changes.
- Unrelated repository changes.

---

## 2.5 Evidence Over Reassurance

Do not say that a process is healthy merely because it still exists.

Health must be supported by evidence such as:

- Current step increasing.
- Epoch increasing.
- Recent log activity.
- GPU utilization.
- CPU activity.
- Stable memory behavior.
- Valid numerical values.
- Expected checkpoint production.
- No repeated exceptions.
- No prolonged retry loop.
- No stalled data loader.
- No runaway disk growth.

Avoid vague statements such as:

- "Everything looks fine."
- "It should be okay."
- "The model is still working."
- "I am keeping an eye on it."

Instead report measurable evidence:

- Current step and total steps.
- Current epoch.
- Latest loss.
- Learning rate.
- GPU utilization.
- GPU memory.
- Available system memory.
- Swap use.
- Process ID.
- Time since last completed step.
- Watchdog state.

---

# 3. Required Job State Machine

Every long-running job must be treated as a state machine.

## State 1: DISCOVER

Inspect the environment and identify:

- Intended host.
- Relevant repository.
- Base model.
- Dataset.
- Training entry point.
- Existing processes.
- Available resources.
- Previous failed or incomplete runs.

Do not modify the system during discovery unless necessary to obtain read-only information.

Transition to PREFLIGHT only after the target environment is understood.

---

## State 2: PREFLIGHT

Validate:

- Required files exist.
- Output path is new or intentionally resumable.
- Dataset passes schema checks.
- Dataset does not overlap protected holdouts.
- Token lengths fit within the intended context.
- Available memory is sufficient.
- Disk space is sufficient.
- Required packages import successfully.
- GPU is visible.
- Training command parses correctly.
- Checkpoint and log directories are writable.
- No conflicting process is using the same output path.

Generate and record hashes for critical inputs where practical.

If preflight fails, stop and report the failed condition. Do not continue by guessing.

---

## State 3: SMOKE_TEST

Run a bounded test using the intended training configuration.

The smoke test should complete at least:

- One full forward pass.
- One backward pass.
- All configured gradient-accumulation microbatches.
- One optimizer step.
- One checkpoint or model-save operation.

During the smoke test, record:

- Peak GPU memory.
- Peak host memory.
- Swap change.
- Runtime.
- Any warnings.
- Whether the process exited cleanly.
- Whether the saved output can be reopened.

If the smoke test fails, do not launch the real candidate.

Investigate the failure, adjust only what is justified, and repeat the bounded test.

---

## State 4: LAUNCH

Launch one candidate at a time unless parallel execution has been explicitly authorized and resource isolation has been verified.

The process must be launched so it can be independently identified and monitored.

Record:

- Process ID.
- Parent process ID.
- Process group ID when available.
- Command.
- Working directory.
- Start timestamp.
- Log paths.
- Output directory.
- Dataset hash.
- Configuration hash.
- Base-model revision.
- Watchdog process ID.

Do not interpret successful process creation as successful training.

---

## State 5: MONITOR

Continue monitoring until the process reaches a terminal state.

At each monitoring cycle:

1. Confirm the exact trainer process still exists.
2. Verify that the command still matches the intended job.
3. Read only new log output when possible.
4. Record the current optimizer step.
5. Record the current epoch.
6. Record training and validation loss when available.
7. Record learning rate.
8. Check for NaN or Inf values.
9. Check GPU utilization.
10. Check GPU memory use.
11. Check host available memory.
12. Check swap use.
13. Check disk space.
14. Check time since the last completed step.
15. Check checkpoint creation.
16. Check the watchdog process.
17. Check for repeated errors or retries.
18. Save monitoring state persistently.

Use a reasonable agent-level reporting interval, such as every several minutes or after meaningful progress.

Use a much shorter deterministic watchdog interval for critical safety checks.

Do not repeatedly print unchanged metrics unless needed to confirm continued stability.

---

## State 6: INVESTIGATE

Enter INVESTIGATE when any of the following occurs:

- No new optimizer step within the configured stall window.
- GPU utilization remains unexpectedly low.
- Available memory trends continuously downward.
- Swap grows toward the safety threshold.
- Disk space is approaching the minimum.
- Checkpoints are missing.
- Numerical instability appears.
- Repeated exceptions occur.
- The process exits unexpectedly.
- Output files cannot be read.
- Monitoring loses access to the host.
- The watchdog stops unexpectedly.

During investigation:

- Preserve logs.
- Preserve process details.
- Preserve the active configuration.
- Inspect the exact failure.
- Avoid broad cleanup.
- Do not immediately restart.
- Determine whether the issue is transient, recoverable, configuration-related, data-related, or unsafe.

---

## State 7: RECOVER

Automatic recovery is allowed only when explicitly authorized.

A restart may occur automatically only when all of the following are true:

- The failure is categorized as recoverable.
- The model and dataset configuration remain unchanged.
- A valid checkpoint exists.
- The checkpoint passes an integrity check.
- The output path will not be corrupted.
- The restart command has been recorded.
- The restart-count limit has not been exceeded.

Do not automatically recover from:

- NaN or Inf training.
- Suspected data corruption.
- Repeated out-of-memory failures.
- Unexplained checkpoint corruption.
- Unexpected loss divergence.
- Evaluation contamination.
- Repository state changes.
- Unapproved configuration differences.

These conditions require investigation or human review.

---

## State 8: VERIFY_COMPLETION

A zero exit code is necessary but not sufficient.

Before declaring success, verify:

- The trainer exited normally.
- The final checkpoint exists.
- Required tokenizer files exist.
- Required configuration files exist.
- Model weights exist.
- Files are nonempty.
- Checkpoint files can be loaded.
- Training state is internally consistent.
- Expected metrics were written.
- No fatal error appeared near shutdown.
- Output hashes or manifest were recorded.
- The watchdog stopped or detached cleanly.
- Temporary resources were cleaned without affecting unrelated services.

Only then may the job be reported as successfully completed.

---

## State 9: EVALUATE

Evaluation must remain isolated from training.

Do not evaluate on the protected final release set while still selecting among candidates.

Use separate data partitions for:

- Training.
- Validation.
- Candidate selection.
- Final release evaluation.

Candidate and baseline evaluations must use identical:

- Task IDs.
- Snapshot contents.
- Snapshot hashes.
- Evaluator version.
- Validator version.
- Decoding settings.
- Maximum generation length.
- Temperature.
- Seed where applicable.
- Tool permissions.
- Sandbox policy.
- Timeout policy.
- Retry policy.

Do not compare results generated under materially different evaluator conditions.

---

## State 10: RELEASE_GATE

A model may proceed toward release only if the declared gate passes.

The gate must reject results when:

- Evaluation task count is incomplete.
- Expected tasks are missing.
- Unexpected tasks are present.
- Candidate and baseline use different snapshots.
- Snapshot hashes differ.
- Evaluator versions differ.
- Validator versions differ.
- Decoding configurations differ.
- Results contain unresolved execution failures.
- The final checkpoint is missing.
- Required model files are missing.
- Dataset provenance is incomplete.
- Holdout contamination is detected.
- Release restrictions are violated.

Do not weaken the release gate to accommodate a preferred result.

---

# 4. Deterministic Watchdog Requirements

The watchdog must be an external process independent of individual model responses.

It must monitor the exact trainer PID or process group.

Minimum watchdog inputs:

- Trainer PID.
- Check interval.
- Minimum available RAM.
- Maximum permitted swap use.
- Minimum available disk space.
- Maximum stall duration.
- Maximum number of consecutive threshold breaches.
- Log path.
- Stop signal.
- Escalation behavior.

Example policy:

- Check every 5 seconds.
- Trigger only after 2 consecutive violations.
- Stop if available RAM remains below 12 GiB.
- Stop if swap use exceeds 4 GiB.
- Stop if free disk falls below the configured reserve.
- Send SIGINT first to allow graceful checkpointing.
- Wait for the configured grace period.
- Escalate to SIGTERM only if required.
- Never send a broad signal to unrelated processes.
- Record every check and action.

Thresholds must be adjusted to the actual hardware and workload. They are not universal constants.

The watchdog must be verified after launch:

- Confirm the watchdog PID exists.
- Confirm it targets the correct trainer PID.
- Confirm its log file is updating.
- Confirm its thresholds match the requested configuration.
- Confirm it does not target unrelated services.

---

# 5. Resource Interpretation Rules

## Memory

On Linux, use available memory as the primary system-headroom signal rather than relying only on the raw free-memory column.

Track:

- Available memory.
- Swap use.
- Trainer resident memory.
- GPU memory.
- Direction and rate of change.

Differentiate between:

- Stable high-water allocation.
- Reclaimable filesystem cache.
- Slow memory leak.
- Sudden allocation spike.
- Normal one-time graph or kernel initialization.

Do not stop a healthy job solely because the free-memory column is low while available memory remains stable.

---

## GPU

Track:

- GPU utilization.
- GPU memory allocated to the trainer.
- Other GPU processes.
- Temperature when available.
- Power or throttling signals when relevant.

Do not assume low utilization means failure without checking:

- Data loading.
- Checkpoint writing.
- Evaluation.
- Gradient synchronization.
- CPU preprocessing.
- Long sequence variation.

---

## Loss

Do not overreact to a single loss spike.

Evaluate:

- Numerical validity.
- Multi-step trend.
- Evaluation trend.
- Dataset heterogeneity.
- Learning-rate schedule.
- Gradient norm.
- Whether loss behavior correlates with a particular data class.

Automatically stop for NaN or Inf.

Do not automatically alter hyperparameters because of one high-loss batch.

---

# 6. Change-Control Policy

## Changes Allowed During an Active Run

Without additional authorization, the agent may:

- Inspect logs.
- Inspect system metrics.
- Inspect process state.
- Activate an external watchdog.
- Fix monitoring scripts that do not affect the trainer.
- Prepare isolated future-run configurations.
- Run isolated tests that do not compete for critical resources.
- Improve documentation.
- Preserve diagnostic evidence.

## Changes Requiring Explicit Authorization

The agent must not silently change:

- Dataset.
- Dataset sampling.
- Base model.
- Model architecture.
- Learning rate.
- Optimizer.
- Batch size.
- Gradient accumulation.
- Maximum sequence length.
- Precision.
- Epoch count.
- Seed.
- Loss function.
- Checkpoint contents.
- Current output directory.
- Evaluation split.
- Release criteria.

## Changes Prohibited During a Critical Run

Do not:

- Replace scripts currently used by the active process.
- Overwrite active checkpoints.
- Move the active output directory.
- Launch a competing GPU-heavy process without verified headroom.
- Run final release evaluation before candidate selection is frozen.
- Modify protected holdout data.
- Reset system services unnecessarily.
- perform broad process cleanup.

---

# 7. Scope-Control Rule

While training is active, remain focused on:

- Job safety.
- Job progress.
- Reproducibility.
- Evaluation readiness.
- Directly blocking defects.

Do not expand into unrelated refactoring merely because additional issues are discovered.

Classify discoveries as:

1. Critical to the active run.
2. Required before evaluation.
3. Required before release.
4. Future improvement.
5. Unrelated.

Address categories 1 and 2 immediately when safe.

Record categories 3 and 4 for later.

Do not interrupt the active run for category 5.

---

# 8. Reporting Standard

Provide meaningful progress reports when:

- Preflight completes.
- Smoke test succeeds or fails.
- Real training begins.
- Watchdog becomes active.
- A safety threshold changes materially.
- A major step or epoch boundary is reached.
- A problem is discovered.
- Recovery is attempted.
- Training finishes.
- Evaluation starts.
- Release gate passes or fails.

A progress report should normally include:

- Job state.
- Current step and total.
- Current epoch.
- Latest relevant loss.
- GPU utilization.
- GPU memory.
- Available system memory.
- Swap use.
- Watchdog status.
- Any action taken.
- What happens next.

Example:

"Candidate c20 is at step 70 of 143. GPU utilization is 96%, trainer GPU memory is stable at 53.2 GiB, system available memory is 17 GiB, and swap use is 0.97 GiB. The PID-scoped watchdog remains active and has not triggered. No configuration changes have been made to the live run."

Do not imply continuous monitoring after the agent execution ends unless the independent watchdog or supervisor has been verified.

---

# 9. Failure Classification

Classify failures before taking action.

## Infrastructure Failure

Examples:

- SSH disconnect.
- Temporary filesystem problem.
- Model server unavailable.
- Container startup failure.
- Transient network problem.

Possible response:

- Retry within limits.
- Preserve logs.
- Resume from a valid checkpoint when authorized.

## Resource Failure

Examples:

- Out of memory.
- Swap escalation.
- Disk exhaustion.
- Thermal threshold.
- Excessive process contention.

Possible response:

- Gracefully stop.
- Preserve the checkpoint.
- Reduce resource demand only with authorization.
- Re-run a smoke test before restarting.

## Numerical Failure

Examples:

- NaN.
- Inf.
- Exploding gradients.
- Persistent loss divergence.

Response:

- Stop.
- Preserve the last stable checkpoint.
- Do not automatically restart unchanged.
- Investigate configuration and data.

## Data Failure

Examples:

- Invalid schema.
- Corrupted record.
- Holdout overlap.
- Unintended class omission.
- Token-length overflow.

Response:

- Stop or quarantine the affected run.
- Preserve provenance.
- Correct future data.
- Do not rewrite the history of the active candidate.

## Evaluation Failure

Examples:

- Candidate and baseline use different snapshots.
- Evaluator imports the wrong installed package.
- Incomplete task subset passes incorrectly.
- Sandbox permissions differ.
- Transient server failures become model decisions.

Response:

- Invalidate affected evaluation results.
- Correct the evaluator.
- Rerun candidate and baseline under identical conditions.

---

# 10. Required Run Configuration

Before each run, populate this block.

## Job Identity

- Job name:
- Candidate name:
- Host:
- Repository:
- Working directory:
- Base model:
- Base-model revision:
- Dataset:
- Dataset SHA-256:
- Output directory:
- Log directory:
- Seed:

## Training Configuration

- Objective:
- Epochs:
- Learning rate:
- Optimizer:
- Batch size:
- Gradient accumulation:
- Maximum sequence length:
- Precision:
- Checkpoint interval:
- Maximum checkpoints:
- Expected optimizer steps:

## Monitoring Configuration

- Agent polling interval:
- Watchdog polling interval:
- Stall timeout:
- Minimum available RAM:
- Maximum swap use:
- Minimum disk reserve:
- Maximum GPU temperature:
- Consecutive violations required:
- Graceful stop signal:
- Termination grace period:
- Automatic restart authorized: yes/no
- Maximum restart count:

## Evaluation Configuration

- Validation split:
- Protected final split:
- Evaluator version:
- Validator version:
- Sandbox configuration:
- Temperature:
- Maximum generation length:
- Retry limit:
- Expected task count:
- Snapshot provenance file:

## Change Authority

- Changes allowed without approval:
- Changes requiring approval:
- Conditions requiring immediate stop:
- Conditions requiring human review:

---

# 11. Launch Procedure

Follow this exact sequence:

1. Complete discovery.
2. Complete preflight.
3. Hash critical inputs.
4. Confirm the output path.
5. Run the optimizer-step smoke test.
6. Verify checkpoint creation.
7. Compare peak resource use with safety margins.
8. Launch one real candidate.
9. Capture the exact trainer PID.
10. Launch the watchdog against that PID.
11. Verify the watchdog.
12. Record the run manifest.
13. Begin active monitoring.
14. Do not launch another candidate until authorized or until the current candidate reaches a terminal state.

---

# 12. Completion Procedure

When training exits:

1. Record the exit code.
2. Preserve the final logs.
3. Verify final model files.
4. Verify checkpoint readability.
5. Record final hashes.
6. Verify the final step count.
7. Verify the watchdog state.
8. Stop only job-specific auxiliary processes.
9. Leave unrelated services untouched.
10. Record the final resource state.
11. Run validation only after training artifacts are frozen.
12. Compare candidates only under identical evaluation conditions.
13. Keep the final release set sealed until the candidate is selected.
14. Report the result with evidence.

---

# 13. Non-Negotiable Rules

- Never treat process launch as task completion.
- Never claim background monitoring without a real background mechanism.
- Never kill unrelated GPU or Python processes.
- Never silently modify a live run.
- Never alter hyperparameters based on one noisy metric.
- Never expose the protected release set during candidate selection.
- Never compare evaluations performed under different conditions.
- Never declare success without verifying the produced artifacts.
- Never hide a failed or superseded run.
- Never rewrite provenance to make a result appear cleaner.
- Never trade reproducibility for a preferred benchmark result.
- Never continue after a hard safety condition merely to avoid losing progress.

---

# 14. Final Operating Directive

Be conservative with the machine, precise with process identity, skeptical of assumptions, and explicit about evidence.

Use the model for interpretation and decision-making.

Use deterministic code for:

- Repeated checks.
- Safety thresholds.
- Timeouts.
- Process supervision.
- Signal delivery.
- Logging.
- State persistence.
- Restart limits.

The agent must remain actively involved while its execution session is open, but it must not pretend that language-model reasoning alone creates continuous supervision.

When the agent session may end before the job does, ensure that the external watchdog and process supervisor are independently active, verified, and capable of placing the job into a safe state.

The job is finished only when the outcome is verified.
