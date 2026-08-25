\# Observations



\## Run 1 — Initial system prompt



Testing the bug-report flow with:



> "On Safari 17 on my iPhone, the product images never load on category pages.

> I open the shop, tap any category, and every image is a grey box."



correctly triggered `create\_bug\_report` and returned a ticket ID

(`356baf47-8a18-41a1-b33a-4dba71932525`), later confirmed present in the

DynamoDB table with status `OPEN`. Routing logic worked as intended for this

complete, single-message bug report.



However, the model's reply included its raw reasoning wrapped in `<thinking>`

tags before the customer-facing message:



> `<thinking> This message describes a problem with the website or app...</thinking>`



This matches a known behavior flagged in the project runbook's

troubleshooting table: Nova's chain-of-thought output leaking into the final

response.



\## Change made before Run 2



Added an explicit instruction to `system\_prompt.txt`'s GENERAL RULES section:



> "Never include your reasoning, thinking, or internal analysis in your reply

> to the customer. Do not use `<thinking>` tags or any similar

> meta-commentary of any kind."

>

> "Every turn must end with exactly one clear, customer-facing message and

> nothing else."



The harness was rebuilt (`create\_harness.py`) to pick up the updated prompt

and confirmed `READY`. The full 7-case test suite was then regenerated and

evaluated as Run 2.



\## Run 2 evaluation results



Run 2 produced an overall Correctness score of \*\*0.71\*\*, unchanged from

Run 1. This confirms the prompt change did not move the aggregate score,

though it did partially address the original issue: `<thinking>` tags no

longer appeared in the FAQ, handoff, and edge-case responses, but they

persisted in the two bug-report test cases — suggesting the leak is tied

specifically to responses that involve field-collection/tool-call reasoning,

not a blanket instruction-following failure.



\### Successful results



\- The incomplete bug report correctly triggered a request for the missing

&#x20; environment detail and did not file a ticket prematurely.

\- The Klarna question was correctly identified as not covered by the FAQ,

&#x20; and the chatbot handed off to human support rather than guessing.

\- The request to speak with a human agent was handled with a polite handoff

&#x20; and no unnecessary tool call.



\### Issues identified



\- \*\*Complete bug report scored 0.\*\* The Android/Pixel 8 message already

&#x20; contained the description, reproduction steps, and environment in a single

&#x20; message, but the chatbot incorrectly asked for reproduction steps instead

&#x20; of filing the ticket immediately. The prompt's definition of

&#x20; "stepsToReproduce" needs to more explicitly state that a natural-language

&#x20; description of the actions that trigger the fault counts as valid

&#x20; reproduction steps, even without an explicit numbered list.

\- \*\*Same-day dispatch question scored 0.\*\* The chatbot handed off, stating

&#x20; the FAQ didn't cover cutoff times, when the reference answer expected a

&#x20; direct figure from the FAQ. This needs to be checked against the actual

&#x20; `online\_shop\_faq.md` content to determine whether this is a retrieval/

&#x20; interpretation issue in the prompt, or whether the FAQ genuinely doesn't

&#x20; contain this information (in which case the reference expectation itself

&#x20; may need revisiting).

\- \*\*Aggregate score unchanged (0.71 → 0.71).\*\* The `<thinking>`-suppression

&#x20; change improved output cleanliness on 5 of 7 test cases but did not affect

&#x20; the two lowest-scoring cases, since their failures were unrelated to the

&#x20; `<thinking>` leak (they were classification/field-recognition and

&#x20; FAQ-retrieval issues instead).



\## Conclusion



Across both runs, the chatbot performs reliably on straightforward handoffs

and incomplete bug reports, and the `<thinking>`-suppression fix

successfully cleaned up 5 of 7 responses. Two concrete gaps remain:

(1) recognizing when all three bug-report fields are already present in a

single natural-language message, and (2) reliably extracting FAQ figures

that are technically present in the source document. Further prompt

iteration should focus on tightening the definition of "stepsToReproduce"

with clear positive examples, and explicitly instructing the model to

re-scan the full FAQ content before concluding information is missing,

rather than reinforcing the `<thinking>`-suppression instruction further —

that part of the fix already worked as intended for non-bug-report cases.



\## Other design decisions



\- Ambiguous routing cases (late delivery, declined card) were explicitly

&#x20; classified as PLATFORM QUESTION rather than BUG REPORT per the routing

&#x20; rules defined in the prompt, avoiding false-positive bug tickets for

&#x20; issues that are actually account/payment/FAQ matters.

\- Memory was explicitly disabled on the harness (`memory={"disabled": {}}`)

&#x20; before creating it, per the runbook's guidance, to prevent conversation

&#x20; state from leaking between independent test cases and skewing evaluation

&#x20; results.

## Observations and Routing Conditions

Routing Conditions Table



The chatbot classifies every customer message into exactly one of three categories before responding.



Customer situation	Route	Routing condition

Website or app is broken, crashes, freezes, or fails	BUG REPORT	The customer reports that a website or app function is not working as expected.

Checkout error or broken checkout function	BUG REPORT	A checkout function is malfunctioning or producing an error.

Customer provides a bug description, reproduction steps, and browser/OS/device	BUG REPORT	All three required fields are present, so the bug report is filed immediately.

Bug report is missing information	BUG REPORT	Ask for exactly one missing field at a time; do not file the report yet.

Late or delayed delivery	PLATFORM QUESTION	Delivery and shipping issues are platform questions, not bugs. Use the FAQ.

Card declined at checkout	PLATFORM QUESTION	Payment issues are platform questions, not bugs. Use the FAQ.

Orders, shipping, returns, refunds, payments, products, accounts, or privacy	PLATFORM QUESTION	Answer only from the FAQ.

Platform question not covered by the FAQ	ANYTHING ELSE	Do not guess; hand the customer off to human support.

Customer asks to speak to a real person	ANYTHING ELSE	Provide the human support line and do not call the bug-report tool.

Request unrelated to a bug or an FAQ-answerable platform question	ANYTHING ELSE	Politely explain that the assistant cannot help and provide the human support line.



\## Key Ambiguous Cases



The system prompt explicitly distinguishes several cases that could otherwise be confused:



Late delivery → PLATFORM QUESTION, not BUG REPORT.

Declined card → PLATFORM QUESTION, not BUG REPORT.

Checkout error caused by the website or app → BUG REPORT.



\## Bug Report Collection Rules



For a BUG REPORT, the assistant must collect:



description — the fault in the customer's own words.

stepsToReproduce — any action or sequence of actions the customer describes that triggers the fault. The steps do not need to be numbered.

environment — a browser, operating system, or device.



The assistant asks for only one missing field at a time and does not call create\_bug\_report until all three fields are present.



Once all three fields are available, the assistant files the bug immediately and returns the ticket ID to the customer.



\## FAQ Routing Rules



For PLATFORM QUESTIONs, the assistant answers only from the supplied FAQ.



The assistant must not invent or guess policies, prices, timeframes, limits, or other information. If the FAQ does not contain enough information to answer the question, the request is handed off to human support rather than answered by guessing.



\## Evaluation Observations



The evaluation demonstrated successful routing for several cases:



A partial bug report correctly resulted in a request for the missing environment detail.

A Klarna question not covered by the FAQ correctly resulted in a human-support handoff.

A request to speak with a real support agent correctly resulted in a polite handoff without a tool call.

A late-delivery question was correctly treated as a platform/FAQ question.

A declined-card question was correctly treated as a platform/FAQ question.



The evaluation also identified areas where the prompt/model behavior could be improved, particularly recognizing when natural-language actions already provide sufficient stepsToReproduce information and ensuring FAQ questions are answered using the exact figures contained in the FAQ.

