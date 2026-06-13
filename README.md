# End-to-End Automated Course Delivery & AI Grading Engine

A complete, zero-touch backend infrastructure for online academies and digital info-products. This system handles the entire student lifecycle: from form-based onboarding and course routing, to timed email drip campaigns, automated exam distribution, and instantaneous LLM-powered test grading.

---

## System Architecture

The ecosystem consists of two distinct, interacting sub-workflows designed to maintain database states and handle async scheduling.

### Phase 1: Onboarding, Provisioning & Drip Lifecycle

```
[Form Trigger] ➔ [Route Course] ➔ [Format Data] ➔ ┌─► [Welcome Email]
                                                 ├─► [Log Student to GSheets]
                                                 ├─► [Notify Admin via Telegram]
                                                 └─► [Wait 25 Days] ➔ [Send Reminder Email] ➔ [Wait 5 Days] ➔ [Route & Send Exam Link]

```

### Phase 2: Autonomous Grading & Certification

```
[GSheets Exam Submission] ➔ [AI Agent (Gemini Grade)] ➔ [Update Score in Sheet] ➔ [Pass/Fail Switch] ──► (True)  ➔ [Send Congrats Email]
                                                                                                  └──► (False) ➔ [Send Retake/Sorry Email]

```

---

## Detailed Component Breakdown

### 1. Registration & Provisioning Pipeline

* **Trigger (`Form`):** Captures incoming student sign-ups via an external form POST request (or directly from checkout events like Stripe/Lemon Squeezy).
* **Course Router (`Course A/B/C`):** A native n8n Rules node evaluates the selected product ID or course selection and passes it to dedicated formatting nodes to personalize course assets dynamically.
* **Fulfillment Stack:** Simultaneously triggers three actions upon data normalization:
* **Gmail (`Welcome Email`):** Delivers instantaneous student credentials and portal links.
* **Google Sheets (`Data Saved`):** Appends student profile, timestamp, and course variation to the primary database.
* **Telegram (`Admin Notified`):** Pushes real-time conversions or registration notices straight to the system administrator's channel.



### 2. The 30-Day Automated Exam Drip

* **Pre-Exam Cushion (`25 Days Wait`):** Pauses execution context for 25 days before firing an automated milestone/preparation check-in email via the `Reminder` node.
* **Exam Deployment (`5 Days Wait` + `Course A/B/C1` Router):** Exactly 30 days post-registration, the system wakes up, checks the user’s course variant via a final Rules router, and dispatches the corresponding custom exam link (`Exam Link`, `Link1`, or `Link2`).

### 3. AI Evaluator & Feedback Loop

* **Database Trigger (`G.Sheets Update`):** Listens for new rows on the exam response sheet when a student completes their evaluation.
* **Cognitive Layer (`Check Sheets` AI Agent):** Passes the student's raw text answers directly into **Google Gemini**. The model grades the text responses programmatically against a provided answer key embedded in its system prompt.
* **Database Sync (`Update Score`):** Writes the derived numeric score and short-form AI feedback string directly back into the student's database row.
* **Branching Logic (`Pass/Fail`):** An If Node assesses if the AI-calculated score hits your established passing threshold (e.g., $\ge 80\%$).
* **True Path (`Congrats`):** Triggers a success email enclosing certification or next-step instructions.
* **False Path (`Sorry`):** Sends a polite failure notice outlining areas of improvement based on the AI feedback string, with an option to retake the test.



---

## Prerequisites & Setup Strategy

### Google Sheets Configuration

Your database workbook needs two separate sheets structured as follows:

* **Sheet 1 (Roster):** `Student Name`, `Email`, `Course Selected`, `Registration Date`.
* **Sheet 2 (Exams):** `Email`, `Course Type`, `Raw Question Answers`, `AI Score`, `AI Feedback`. (Configure the Phase 2 trigger node to watch this second sheet for `rowAdded` events).

### AI Grading Prompt Guidelines

For deterministic evaluation results, configure the Gemini Chat Model with a zero-temperature setup and a strict grading rubric structure:

```text
You are an objective academic evaluator. Your task is to grade the incoming student exam answers against the following reference key:
[Insert Questions & Answer Rubric Here]

Output your evaluation exactly in this format for database consumption:
Score: [Calculated percentage without % sign, e.g., 85]
Feedback: [A 2-sentence summary detailing what answers were right or wrong]

```
