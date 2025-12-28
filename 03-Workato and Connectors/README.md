# Workato Connector Framework & API Strategy

## 1. Why Learn API Integrations in 2025?

In 2025, API is no longer just "technical plumbing"; it is the **product**.

* **The 2025 Context:** We have moved beyond simple "Get Data" requests. We are now in the era of **Agentic AI**. AI agents cannot click buttons on a website; they need APIs to "act." A PM who understands API design (**REST, Webhooks, GraphQL, MCP**) is essentially designing the instructions for how AI will interact with their product.
  * *Note on MCP:* The Model Context Protocol (MCP) is the emerging standard for connecting AI models to data and tools, making it a critical skill for PMs in the AI realm.

* **Career Logic:** If you are a PM for lets say "Deductions Cloud" at HighRadius and you don't understand integration, your product is an island. Your customers also live in Salesforce, Oracle, and Slack. If your product doesn't talk to those products, you are at a disadvantage and can easily be disrupted by startups.

## 2. Why Modern Companies Are Obsessed with Integrations

* **"Plug-and-Play" Agility:** Companies no longer want to be "locked in" to a single vendor. While many follow a **"Best-of-Breed"** strategy i.e. using Google for **Auth** for identity, **Stripe** for payments, **SAP** for finance, and **Zoho** for CRM, an executive decisions to swap vendors are frequent. These are driven by **Reliability, Support, Pricing (ARR reduction), Investor Partnerships, etc.**

  * *Real-World Example:* An executive might swap **Stripe** for **Adyen** because Adyen offers lower "Interchange Fees" in European markets (often 0.1%–0.2% lower) and provides better support for local payment methods like SEPA.

  * **The Role of Connectors:** High-quality connectors allow businesses to swap these "blocks" without touching their core product code. It provides an insulation layer so that a change in the payment provider doesn't break your product's business logic.

* **The Composable Enterprise:** Modern enterprises treat software like LEGO blocks. The **CIO** (who authorizes the technical stack) ensures these blocks are cost-effective and interoperable. Integrations are the "glue" that allows them to pivot their tech stack based on shifting business needs without massive re-engineering efforts.

## 3. Workato Ecosystem: Recipes, Frameworks, and SDKs

To understand Workato, you must separate the **User View** from the **Developer View**.

### A. What is a "Recipe"? (The User View)
A Recipe is an automated workflow. It is the logic layer.

* **Analogy:** If Workato is a kitchen, the "Recipe" is the step-by-step instruction card.
* **Real-World Example:** "Quote-to-Cash" (Common in SaaS)
  1. **Trigger:** Salesforce `Opportunity` is marked "Closed Won".
  2. **Step 1 (Logic):** IF `Amount` > \$10,000.
  3. **Step 2 (Action - NetSuite):** Create a generic `Sales Order` in NetSuite.
  4. **Step 3 (Action - Slack):** Post a message to `#sales-wins`.
  5. **Step 4 (Action - DocuSign):** Send a thank you envelope to the primary contact.

### B. The Connector Framework & SDK (The Developer View)
The **Workato SDK** (Ruby-based) is the toolbox for building *custom* connectors. To build a complete connector, a developer must define these **five core components**:

#### 1. Connection (The Security)
Before logic, you must handle authentication (OAuth2, API Keys, etc.). This ensures Workato can safely talk to the target app without the user typing passwords into every recipe.

#### 2. Objects (The Nouns) & Dynamic Schema
* **Object Definition:** The developer defines the "Noun" (e.g., `Dispute`).
* **Dynamic Schema:** Instead of hardcoding fields, you write a `describe` function. (They do this as tomo SalesForce might add new fields and hardcoding fields is never a good strategy)
  * *The Flow:* Workato calls the target API's "metadata" endpoint (e.g., Salesforce `/describe`).
  * *Filtering:* The developer can write code to **ignore fields** that are irrelevant or sensitive, ensuring the end-user only sees what they need.

#### 3. Events/Triggers (The Listeners)
* **Polling (The "Check-in"):** e.g., Jira. Polling is "durable" <BR>It asks for everything since the last successful timestamp.<BR>(Check appendix to understand why polling is preferred in Jira’s case).
* **Webhooks (The "Push"):** e.g., Stripe. Instant notification sent via a POST request from the target app to Workato.

#### 4. Actions (The Verbs)
Actions are the "Doers" of the connector. They translate a user's intent into a specific API call.
* **Example Action (Salesforce):** "Create or Update Case."
* **Input Fields:** What data does the user need to provide? (e.g., `Case ID`, `Subject`, `Priority`, `Description`).
* **Execution Block:** This is the Ruby code that constructs the HTTP request. For an update, it might be a `PATCH` request to `/services/data/v60.0/sobjects/Case/{id}`. (check appendix to learn what is a Patch request in HTTP).
* **Output Fields:** What does Salesforce send back? (e.g., Case ID, `Case Number`, `Success: True`, `Last Modified Date`).

#### 5. Methods (The Helper Code)
Methods are reusable "utility" functions that sit behind the scenes.
* **Real-World Example:** A **Currency Converter**. Instead of writing complex math inside every single Action (Create Invoice, Update Refund), you write one "Method" called `format_to_usd`. Every Action can then simply call that method to ensure the data is always sent in the correct decimal format.

## 4. The Strategy: Buy Workato vs. Build Internal Platform?

### The "Build" Approach: DoorDash Case Study
DoorDash (via their [Merchants Integration Portal](https://merchants.doordash.com/en-us/integrations)) demonstrates the power of a centralized platform.

* **The Strategy:** Rather than making their Merchant team learn how to write Salesforce code, they created a centralized **Integrations Hub**.
* **The Logic:** By building an internal "abstraction layer," DoorDash allows their product teams to focus on "Merchant Experience" while the centralized hub handles the messy "handshakes" with external partners like POS systems or CRMs.
* **Pro:** This internal platform allows DoorDash to maintain **total ownership** of their data security and architectural standards. They don't have to worry about a third-party vendor (like Workato) changing their pricing or deprecating a critical feature that their business relies on.
* **Critique:** While this mimics Workato's *functionality*, it lacks the *low-code UI*. This means only engineers can build new integrations at DoorDash, whereas at a company with Workato, a PM could theoretically build the workflow.

### Why "Tightly Binding" Connector Code is Bad
If you write Salesforce-specific code inside **Deductions Cloud**, you are "coupled." If you later add **Zendesk** or **ServiceNow**, you have to duplicate that logic.
* **The Better Way:** Deductions Cloud emits a generic event: `Deductions_Updated`. Your internal "Connector Platform" translates that into the specific requirements for Salesforce or ServiceNow.

## 5. Real-World Case Study: Slack’s "Order-to-Cash" Engine

*Citation:* [Slack + Workato: How Slack automated Order-to-Cash](https://www.workato.com/the-connector/how-slack-automated-order-to-cash/)

### The Trigger: Salesforce CDC
When a deal closes, Salesforce pushes an event to a "Message Bus" (a queue) inside Salesforce. Workato "subscribes" to this bus using a protocol called **CometD** (see Appendix).

### THE Action B: Product Provisioning
Once Finance clears the deal, Workato calls the **Slack Admin Console** (an internal tool) to automatically enable features for the customer.

## 6. Workato from a Product Manager's Lens

### Product Critique: How can we make it better?

* **Testing & Mocking**
  * **The Problem:** Currently, if you want to test a "Large Deal Alert" recipe, you have to go into Salesforce and **manually create a dummy \$1M deal**.
  * **The Impact:** This messes up your Sales Ops reports, triggers notifications to executives, and requires you to remember to delete it later.
  * **The Solution:** A **"Mock Trigger"** feature would allow PMs to upload a sample JSON payload. You could simulate a \$1M deal entirely within Workato to see if the logic works, without ever touching the real Salesforce production database.

## Appendix: Technical Deep Dives

### What is CometD?
CometD is a protocol that allows a server (like Salesforce) to push data to a client (like Workato) over HTTP. Think of it as a "long-running phone call" where the server only speaks when there is an update.

### What is a Message Bus?
It is a "waiting room" for data. When Salesforce CDC fires, it doesn't send a message directly to Workato. It puts the message on a Bus. The Bus keeps the message safe until the subscriber (Workato) is ready to pick it up.

### What is Salesforce CDC?
**Change Data Capture.** It is a technology that automatically notices when a record is created, updated, or deleted and immediately broadcasts that change as an event.

### Why polling for Jira?
Jira sends their updates via "Best effort over the public network" (a networking term meaning the system will try its hardest to deliver the message, but it **doesn't guarantee** it).

Imagine sending a standard postcard through the mail. The post office doesn't track it, and they don't notify you if it gets lost. Jira's webhooks are like that postcard: if Jira's servers are overloaded or there's an internet "blip," the message vanishes. There is no "retry" mechanism. **Polling** is the "Registered Mail" equivalent—it ensures you never miss a record.

### What is a Patch Request?
A **PATCH** request is the "surgical" way to update data.

1. **GET (The "Read")**
   * **Purpose:** To retrieve data from a server.
   * **Analogy:** Asking a librarian for a specific book.

2. **POST (The "Create")**
   * **Purpose:** To create a brand-new record.
   * **Analogy:** Opening a new bank account.

3. **PATCH (The "Update" - Surgical)**
   * **Purpose:** To update **parts** of an existing record without sending the whole thing.
   * **Analogy:** Calling the bank to change *only* your phone number.

**Summary Table:**

| Method | Action | Payload Requirement |
| :--- | :--- | :--- |
| **GET** | Read | None (usually) |
| **POST** | Create | Full data for new record |
| **PATCH** | Update | **Only** the fields you want to change |
| **PUT** | Replace | The entire record (overwrites everything) |

### There has to be some Mapping between Salesforce fields to Netsuite fields, where is it defined in the journey ?
Mapping SalesForce's **CPQ (Configure, Price, Quote)** data from Sales to Finance is the hardest part.
* **The Location:** This mapping logic lives in the **Workato Recipe**.
* **The Process:** In the "Objects" section of the SDK, the developer identifies that a `Price` field exists. In the **Recipe Editor**, the PM uses a "Lookup Table" to say: *"If the Salesforce SKU is 999, map it to Workday Revenue Category 'Accessories'."*
* **Benefit:** Business users can change these mappings instantly without asking an engineer to redeploy code.

### What is CPQ?
**Configure, Price, Quote.** It is the software (like Salesforce CPQ) that Sales Reps use to build complex deals. In our context, CPQ data is the "source of truth" for what the customer actually bought.
