Problem
 Small and mid-sized businesses can't safely put AI agents to work. The AI labs sell subscriptions to individuals and send forward-deployed engineers to large enterprises, and the businesses in between get neither. When they try to automate on their own, the automation fails: a Zapier chain breaks after a software update, an RPA bot breaks after an ERP update, or an agent built by ops runs unchecked until a customer notices. The result is that they go back to manual work.

Two things are missing. The first is a governance layer that controls what an agent can touch and records everything it does. The second is a way for non-engineers to create agents at all. Neither is useful without the other

Product: two layers, each dependent on the other

Keel OS (the governance layer). Every agent connects to the business's systems through Keel. An agent never holds its own permissions. It acts on behalf of a specific person, and its access is limited to what that person is allowed to do and what the agent was granted. Each action is checked against a policy (allow, deny, or require human approval) and written to an audit log.
Keel Builder (the creation layer). The owner describes the task in plain English. Keel generates the agent and displays it as a readable set of blocks (inspired by Scratch coding language), so the owner can see what it will do before it runs. The block palette only shows actions the owner's permissions allow.
The two layers only work together. The Builder depends on the OS because a non-technical owner can't safely deploy an agent without enforced limits and a record of what happened. Without them, the Builder is just another Zapier waiting to break. The OS depends on the Builder because a business without engineers has nobody to create agents for the governance layer to govern.

User
 Owners and office managers of residential field service companies (HVAC, plumbing, electrical) with roughly 5–50 employees. They have no technical staff, they run dispatch and the back office themselves, and they already pay for field service software they barely use. The owner is both the user and the buyer.

Why field services: Most of these companies run on a small number of field service platforms (ServiceTitan, Jobber, Housecall Pro), so we can reuse connectors we build across customers. Owners can decide to buy on their own; there are thousands of these companies, and there are no regulatory barriers. We ruled out commercial gyms because a single platform owns the data there and is building AI natively.

Rollout: build for the owner, operate it ourselves first
 The owner is the target user. For our first customers, though, we'll set up and configure each agent ourselves using the Builder. The owner will handle approvals and review the log. We'll hand the Builder to owners only after we've verified they can read an agent, explain what it does, and safely change it. This lets us test the non-technical claim directly rather than assuming it.

Wedge task
 Turning a closed job into a clean equipment record. When a technician closes a job, the agent does four things:

Reads the close-out notes and the photo of the equipment nameplate.
Extracts the make, model, serial number, and install date.
Writes those details to the customer's record in the field service software.
Sets the next maintenance due date.
Keel's policy lets the agent read jobs and write equipment fields, and nothing else. It cannot touch invoices or pricing, and it cannot message customers. Any field the agent is unsure about goes to the office manager for approval instead of being guessed, and every step is logged.

This data is what the HVAC owner we interviewed loses today, and it's why his maintenance follow-up never happens. It's also a real test of both layers: the agent writes to the business's core system of record, and the owner has to be able to trust and verify what it wrote.

What "working" means
 A run on a closed job counts as working when all of the following hold:

Every equipment field matches ground truth, checked against the nameplate photo and the technician's notes.
The record is written to the correct customer and job.
The agent takes no action outside its policy.
Uncertain fields go to a human for approval rather than being filled in with a guess.
Every tool call appears in the audit log.
For the Builder, "working" means the owner can read the block view and correctly describe what the agent will do before it runs.

Metric
 Verified completions per week: closed jobs where the agent produced a correct equipment record in the customer's system, stayed within policy, logged every step, and had the result confirmed by the owner or office manager. The count is 0 until we launch. We chose this metric because it only goes up when both layers do the whole job correctly for a real business.

User conversations
Conversation 6: Head of IT, regional dental group (12 locations)
 He manages internal systems across all the practices, with no dedicated AI staff, and is evaluating automation for front-office workflows. The group's no-show rate is 22%. Their reminder system sends everyone the same message regardless of their history. He manually pulls a weekly report of repeat no-shows for the front desk to call, and the front desk forgets or deprioritizes the calls.

"I can see exactly who's going to no-show but I can't do anything about it."

Conversation 7: Director of Operations, regional staffing agency
 He manages placements for 400+ active contractors with no engineering team, only one person who is strong in Excel. Contractors sometimes go silent partway through a placement, and the agency only finds out when the client complains. He built a spreadsheet that flags missing timesheets, but someone has to check it every morning and reach out manually.

"By the time we know there's a problem the client already knows there's a problem and is unhappy with us."

Conversation 8: Co-founder, 30-person B2B SaaS company
 He's a technical founder with two engineers, neither of whom has ML experience. They've tried to build internal agents twice and abandoned both attempts. Their customer success work is purely reactive: they learn an account is unhappy from support tickets, even though the warning signs are already in Mixpanel and Salesforce. Nobody has time to connect the two systems, and Gainsight is too expensive and heavy for a company their size.

"We have all the signals but can't do anything about it."

Conversation 9: Owner, 15-person residential HVAC company
 He's a non-technical owner who runs dispatch and scheduling himself. Equipment details from field jobs get lost or entered inconsistently, so there's never a clean list to drive maintenance follow-up. He uses about 20% of ServiceTitan and tracks follow-ups in a notebook.

"I paid a lot of money for software that I don't understand and my guys don't use right."

Conversation 10: VP of Finance, mid-market manufacturing company
 He owns internal reporting and is the economic buyer for tooling. Monthly close takes 11 days, and four people spend the first week pulling numbers from three ERP modules into a master spreadsheet. An RPA tool they tried two years ago broke every time the ERP updated.

"We just do it manually now because at least that doesn't break."

Patterns across all 10

Automation fails , so people retreat to manual work (conversations 2, 3, and 10). This is the case for the OS layer.
The data already exists but no one acts on it (conversations 3, 6, 7, 8, and 9).
Mid-market companies have no workable path. Enterprise vendors won't sell to them, and their own attempts break (conversations 4 and 8). Every enterprise deployment starts by relearning how the business works (conversation 5).
The surprise was on the non-technical side. The fitness studio owner doesn't want to build anything (conversation 3), and the HVAC owner can't use the software he already paid for (conversation 9). That's why the Builder is designed for verification rather than programming, and why we're operating it ourselves until we've tested whether owners can use it.

Weekly update

Shipped this week: 5 more user conversations (10 total). Narrowed to one vertical (residential field services) and one wedge task (closed job → verified equipment record). Defined the two-layer product, our pass criteria, and our locked metric.
Our number, and last week's number: Verified completions: 0 (pre-launch). Last week: 5 people with the problem, under the conversation-count metric.
What a user told us: "I paid a lot of money for software that I don't understand and my guys don't use right."
Biggest blocker: Getting real job data and system access. Field service platforms restrict API access for small developers, so we need a design-partner shop willing to share closed-job records and connect their account.
Next week's target: A recorded prototype that takes one closed job through the Builder's spec, Keel's policy checks, and the audit log to a verified equipment record, running against a mock field service system. 
