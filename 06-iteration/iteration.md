# Iteration: Analytics, Sprint, Final Recommendation

> Module 6 · Evals & Iteration. Read the analytics, run an iteration sprint, present with evidence.

## What the evidence says

_What real usage showed: numbers if your tool has analytics, counted behaviour if it does not. Put the signal that matters on screen._

- **Primary signal:** Users were able to explore the product and understand the overall interface, but some workflow actions and permissions were not immediately clear.
- **What moved:** Testers navigated beyond the dashboard into Action Details, Reviewer Queue, Create Action and Review flows. Feedback described the interface as clear and intuitive, and the dashboard as useful for quickly understanding priorities and status.
- **What didn't:** Some users were unclear about why an action could not be sent for review or who had permission to perform certain actions. One tester also reported being unable to download evidence files, and another expected to be able to create an account.


## Iteration sprint

| Change | Hypothesis | Result |
| --- | --- | --- |
| Make the requirements for sending an action to review more visible. | If the requirements are more visible, Action Owners will understand why they cannot send an action for review and what they need to complete next. | The submission requirements are now displayed directly at the point of action, showing why Send for Review is unavailable and what evidence remains to be completed. The existing submission workflow and permissions remain unchanged. |

## Peer feedback

Pascal:
"Hi Elisabeth, I like the very clear to-do list with priorities etc. I tested OK the ability to comment on some actions. You may add a 'Journal' view with the history of comments."

Jomkwan:
"I tested with Marcus bell and left comments and tried changing the status I see it in audit trial but it would be nice to have be notified of changes since I moved from medium to high priority"

Nnanna:
"I really like the clean intuitive interface. The dashboard gives quick data insights

I was confused as to why I could not move one item to review. I later saw a text that showed that only a certain person could move it. It would be nice to have a pop-up to make it more obvious"

Péter:
"Hi Elisabeth,
I really liked the overall look, and in the action page the help around the Send button (what is missing, or who can do that if I don't have the right to do) is super.
I noticed that I cannot download the evidence files."

Ila:
"1. I could not find a create account option for vantage Controls, I only see sign-in option
2. I tried som random login credentials, it did display a valid error message saying the email and password combination didn’t work. Try again"

## The recommendation

**Decision:** ☐ Go  ☑ Iterate  ☐ Kill

_The evidence that justifies the call:_

Peer testing showed that users understood the overall product and found the interface clear and intuitive, but identified friction around workflow permissions and progressing actions to review. One tester also reported difficulty downloading evidence files. This provides enough signal to continue iterating rather than considering the hypothesis fully validated.

## Final showcase

- **Demo link:** https://vantage-controls.lovable.app/
- **The one-sentence story:** Vantage Controls helps teams manage control actions from assignment through evidence submission and review.
- **Where it landed on the Confidence Line (M2 → now):** Early positive signal, with the core workflow understood but some usability friction still to resolve.
