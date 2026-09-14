# PROMPTS.md: Living Prompt Pack

> Module 3 · Prompt Chaining. Re-architect the build with prompt chains; capture the reusable ones here.

## How to use this pack

_Each prompt is a reusable step. Chain them: the output of one becomes the input to the next._

## Prompt chain: Control Action Workflow Chain

### Step 1: Expand, build new screens in a strict sequence
```
Build the next phase of this app in a strict sequence:

1. Add a screen "{{Create Action}}".
This screen allows a Controls/Audit user to create and assign a new control action.

Include:
- Action title
- Finding
- Recommendation
- Required Evidence, supporting multiple evidence requirements
- Action Owner
- Reviewer
- Control Area
- Severity
- Due Date
- Create Action and Cancel actions

Match the layout, spacing, form hierarchy and visual language of the attached {{Action Detail reference}} screenshot.

2. Add a screen "{{Reviewer Queue}}".
This screen allows a Reviewer to see the actions that have been submitted for their review.

For each action show:
- Action reference
- Title
- Action Owner
- Control Area
- Severity
- Due Date
- Evidence completion
- Review status

Match the layout, spacing and data density of the attached {{Dashboard reference}} screenshot.

3. Add a screen "{{Review Action}}".
This screen allows a Reviewer to review an action submitted by an Action Owner.

Preserve the existing structure of:
- Finding
- Recommendation
- Required Evidence
- Comments
- Audit Trail

Allow the Reviewer to:
- Inspect the evidence provided
- Approve the action
- Return the action to the Action Owner with a required comment

Match the layout, spacing and information hierarchy of the attached {{Action Detail reference}} screenshot.

4. Navigation: write the logic so:

{{Create Action}} creates and assigns an action to an Action Owner.

The Action Owner can work on the action through the existing Action Detail flow and Send for Review.

Send for Review makes the action available in {{Reviewer Queue}}.

Selecting an action in {{Reviewer Queue}} opens {{Review Action}}.

Approving the action marks it as completed.

Returning the action makes it available to the Action Owner again and records the review comment.

5. Add a lightweight prototype-level role/user switch so the workflow can be experienced as:
- Controls/Audit/Reviewer
- Action Owner

Do not add authentication, passwords, or backend logic.

Build these in order so {{Create Action}} anchors the creation flow, {{Reviewer Queue}} anchors the reviewer flow, and {{Review Action}} completes the review workflow.

Preserve all existing Vantage Controls functionality and design patterns. Do not add loading, empty, or error states in this step and do not redesign existing screens.
```

### Step 2: Behavior, hard-code the states
```
Apply the following logic constraints to the {{Reviewer Queue}} flow:

- Use skeleton rows for the {{actions awaiting review list}} loading state.

- If no actions are awaiting review, show the empty state:
  "No actions are waiting for your review."

- On fetch failure, trigger the error state:
  "We couldn't load the actions awaiting review. Try again."

Maintain the same Vantage Controls design language throughout and tether all behavior strictly to the existing reviewer workflow.

Do not add new screens, features, or navigation in this step.
```

### Step 3: Refine, one surgical polish
```
The {{Action Detail}} screen needs a professional visual hierarchy and spacing polish.

1. Start by listing the 3 biggest gaps in typography and spacing in the {{Required Evidence}} section compared to the established design language used across the rest of Vantage Controls.

2. Once you've identified those, resize the headers and update the {{Required Evidence cards}} to create a clearer hierarchy between the evidence requirement, its status, and its available actions.

Keep the existing Vantage Controls visual language and components consistent.

Don't change anything else in the project or touch the underlying logic.
```

## Reusable techniques learned

- Using a visual reference kept new screens consistent with the existing product

- Separating Expand, Behavior and Refine made it easier to test one type of change at a time

- Explicitly limiting what the AI could change helped preserve working parts of the flow

- Testing the full workflow exposed permission and navigation issues that were not obvious from the prompt alone

## What broke (and the fix)

_Where a single mega-prompt failed and chaining fixed it._

The first Expand output created the new workflow, but testing showed several issues: Action Owners could still see reviewer-only navigation, the user switcher was unclear, reviewers could not edit the fields they owned, evidence could not be downloaded, and approving an action completed it immediately without a final review comment.

Instead of rebuilding the flow, I tried to fix only those specific issues while preserving the working Create Action → Action Owner → Send for Review → Reviewer Queue → Approve / Return to Owner workflow.

Breaking the work into chained prompts made it easier to correct these issues without introducing unnecessary changes elsewhere in the prototype.
