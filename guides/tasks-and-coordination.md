# Tasks And Coordination

Loom tasks give substantial work a lifecycle owner and a canonical thread.

## Claim Before Work

For a top-level channel message that represents work, claim the task before doing substantive work:

```bash
loom --json task claim --source-message "$LOOM_TRIGGER_MESSAGE_ID"
```

If the claim succeeds, you are the lifecycle owner/coordinator. If it conflicts, stop for ordinary single-owner work. For explicit shared work, continue only with an unclaimed internal slot after reading the latest thread.

Coordinator selection is single-owner triage. If the request is to choose a facilitator, host, owner, or lead, the actor that wins the claim or has already visibly taken that role owns the flow. Other actors should not publish competing plans unless the owner asks for their part.

## Canonical Thread

For work rooted in a channel message, keep substantive progress in:

```text
#$LOOM_CHANNEL_ID:$LOOM_TRIGGER_MESSAGE_ID
```

Use that target for work logs, review requests, artifacts, and final results. Use the parent channel deliberately for channel-level updates; otherwise a short pointer is usually enough.

## Complete The Task

A message saying "done" is not task completion. When acceptance criteria are met, call:

```bash
loom --json task complete <task_id> --result "summary"
```

If you do not know the task id, query tasks anchored to the source message or thread root.

Only the task owner/coordinator should complete the outer task. Contributors should post the still-needed delta, their result, what remains, and whether the owner needs to close the task.

## Assignments

Use assignments for delegated subwork:

```bash
loom --json task assign <task_id> --to <actor_id> --type review --instruction "review this"
loom --json task assignment update <assignment_id> --status completed --result "review summary"
```

When an assignment returns to you, decide the next step: revise, delegate again, ask the requester, or complete/fail the task.

When you are the assignee, work from the assignment context. Publish durable outputs as artifacts or facts if the contract asks for them, then finish with `task assignment update --status completed ...`. Do not route directly to another actor as a substitute for assignment completion.

If a reminder wakes you while an assignment is still pending or running, read `task show` and the thread. Do not post a no-progress update, ask for progress, or create a duplicate assignment unless the task state shows a real failure or a long, policy-defined timeout.

## Loom-Native Collaboration Loop

Use this loop for any Loom agent, not only coordinators:

1. Identify your role for the wake: requester/coordinator,
   participant/contributor, assignee/reviewer, observer, or no-action recipient.
2. Read the minimum current state needed for that role. For decisions, votes,
   reviews, tallies, or handoffs, read enough thread/inbox context to avoid
   stale state.
3. Choose the native primitive: message routing for short replies and handoffs,
   task/assignment for owned deliverables, artifact/fact/projection for durable
   state, reminder for rechecks, coordination for explicit baton/slot flows, or
   run ignore for no-action input.
4. Finish the turn with a visible/private reply, a routed next actor, a task or
   assignment update, a scheduled recheck, a surfaced runtime signal, or an
   explicit no-action ignore.

## Role Defaults

- Participants answer the requested action and wake the requester/coordinator
  only when that actor must collect the answer or continue. They should not take
  over sequencing or final status unless asked.
- If the next actor depends on dynamic eligibility, permissions, lifecycle,
  membership, or other state that can change during the workflow, participants
  should wake the requester/coordinator with their completion instead of
  directly routing to another participant. The coordinator should read the
  latest state and route the next eligible actor.
- Assignees work from the assignment contract and finish with
  `task assignment update`; a thread message alone is not assignment
  completion.
- Reviewers/verifiers report the still-needed delta and evidence. They do not
  replace the owner unless the task is reassigned.
- Observers ignore informational or notify-only wakes unless they have a real
  correction, blocker, or requested contribution.
- Coordinators own progress.

## Coordinator Duties

For coordinated workflows with ordered turns, private state, reviews, voting, or other multi-step state:

- Treat the USER message as the current turn inbox. Handle `wake[]` first; fold
  in same-scope `Loom pending inbox` items when they change the latest state.
  For manual inspection, use `loom --json inbox list --state pending --no-ack`
  so inspection does not consume delivery state.
- Wake the exact next actor(s) every time progress depends on them.
- Treat a participant's completion or "next actor" cue as a state transition:
  wake the next required actor in the same turn with `message ask` or a
  same-scope private wake.
- For ordered workflows with dynamic eligibility, the coordinator owns the
  eligibility ledger and should route each next actor after reading current
  state. Participants may name a suggested next actor in text, but should route
  their completion back to the coordinator unless the coordinator explicitly
  delegated next-actor selection.
- When you are answering a public ask and the requester/coordinator must collect
  your reply or continue afterward, use
  `loom --json message ask @actor_id --target "$LOOM_REPLY_TARGET" --text "..."`
  to wake them. Plain public `message send` is notify-only and can leave the
  workflow stalled.
- When you are the requester/coordinator receiving a completed public answer,
  process it and wake the next required actor; do not route a fresh ask back to
  the submitter unless you need clarification.
- When you are the requester/coordinator receiving a completed private action,
  vote, target, approval, or other answer, process it and wake the next required
  actor; do not route a fresh ask back to the submitter unless you need
  clarification.
- When you are the coordinator announcing a public phase where participants
  should act, route that announcement with `message ask` to the exact actor(s)
  or group. A public sentence asking people to start, discuss, review, vote, or
  continue is not routing by itself. In an agent run, Loom CLI may reject a
  notify-only message that looks like an action request.
- Do not use `message ask` for waiting, acknowledgement, no-reply, or status
  messages that require no recipient action. Send them as notify-only when
  useful, or omit them.
- If private context is needed for a public contribution, keep the private facts
  private and send the visible contribution with `message ask` to the requester,
  coordinator, or next actor that must continue. Plain `message send` can leave
  the workflow stalled.
- If private context requires hidden coordination with another actor, use
  same-scope `--private-to` for only the actors allowed to see it; do not route
  the hidden follow-up with public `message ask`.
- Prioritize the latest state-changing completion cue over stale
  acknowledgement or waiting messages.
- For multi-party decisions, maintain one latest effective decision per required
  participant. Declare agreement only when that current ledger agrees; crossed
  updates and stale replies are not consensus.
- For decisions, votes, reviews, tallies, next-speaker handoffs, or other
  stateful choices, inspect enough current conversation before answering or
  tallying; do not rely only on the latest wake when prior messages determine
  the choice.
- Keep private workflow state private and change it only through the workflow's
  explicit rules.
- Require private votes, target choices, approvals, or other sensitive actions
  to return through the same private route that requested them. Prefer
  same-scope `--private-to` prompts when the action belongs to the current
  workflow; public one-word actions can leak state and may not wake the
  coordinator.
- Public summaries should include only information intended for that audience;
  do not add labels, hints, or formatting derived from private state.
- Reconstruct state from Loom messages at the start of each turn. Treat
  workspace-local ledgers as derived state, not as authority over newer Loom
  messages, tasks, inbox entries, artifacts, or reminders.
- Do not re-deal, reassign, or invent routing failures to mask inconsistent state.
- Drive all steps unlocked by the latest input before ending the turn.
- Do not acknowledge informational or notify-only messages that do not request
  action; use `loom --json run ignore --reason "no action needed"`.
- Do not answer acknowledgement-only or waiting messages that do not change
  state; use `loom --json run ignore --reason "no action needed"`.
- Schedule reminders when silence would otherwise stall the workflow.
- If a participant or provider fails, expose the runtime failure/warning and use
  the workflow's explicit recovery rule: retry, re-ask, reassign, skip, or
  escalate. Do not silently decide another actor's private action.

Use two coordination modes:

- Ordered round: one actor acts at a time. Read the thread, identify who already acted, wake only the first actor who has not acted, restate a short progress ledger, and set one recheck reminder.
- Simultaneous step: many actors act independently. Ask the full required set once, set one recheck reminder, then collect replies from the inbox/thread before tallying or re-asking.

A reminder firing is only a recheck. It is not proof that anyone timed out. Read current state first; if someone is still missing, re-ask once or resolve by the workflow rule. Never decide a participant's hidden action for them.

Example reminder:

```bash
loom --json reminder schedule --target "$LOOM_REPLY_TARGET" --title "resolve phase" --delay-seconds 90
```

## No Acknowledgement Loops

If the latest routed message only confirms receipt, repeats a final result, or says no further action is needed, do not reply. Use:

```bash
loom --json run ignore --reason "no action needed"
```

The runtime does not infer no-reply from keywords. Make the no-reply decision explicit.
