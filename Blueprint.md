# Blueprint - Portfolio Health Report

## 1. Data Ingestion & Initial Processing
- **Input:** Multiple threads of raw text files containing emails can be ingested into a cloud object storage like Amazon S3.
- **Parsing:** 
    - **Email Threads:** Parse each email individually and store them in separate JSON objects adding a thread_id and a message_id to each record.
    - **Emails:** The emails will be parsed with key-value header detection followed by the body. The reason is that the headers can be out of order. The headers to detect are `From`, `To`, `Date`, `Subject`, `Cc` (optional: so add it as blank/empty if it doesn't exist). We should consider forwarded emails as emails within emails that contain the same format. Also, emails usually have signatures that are useless for the AI model like names, roles, legal disclaimers that should be truncated to not waste tokens.
    - **Colleagues:** The colleagues will be parsed so we know the role, name and email of each worker and we can assign an ID to them. Since some workers have multiple emails we will map their emails to a single person's ID so that they don't appear twice. This way we can avoid problems with ownership assignment.
- **Scalability:** To make this scalable, instead of reading from text files, ingestion would directly happen from email servers and logged in a project management system like Jira which can be used to track issues, then we will store them in a Amazon S3 object (scalable), and finally, the messages will be sent for distributed processing via Spark for parsing, cleaning & normalizing, and AI inference.

### Diagram
```
[Email Servers] -> [Jira] -> [S3 object] -> [Spark Cluster] -> [Parsing] -> [Cleaning & Normalizing] -> [AI inference]
```

---

## 2. The Analytical Engine (Multi-Step AI Logic)

### Attention Flags
We will focus on two categories of issues that demand the Director's immediate attention:
1. **Unresolved High-Priority Issues**
    - Issues that have risen but have not been solved.
        - Can be crucial issues that pose a threat to the system.
        - Can be issues unresolved for a long period of time.
        - Can be issues that were identified by high role members of the company.

2. **Emerging Risks or Blockers**  
    - Messages which indicate no clear path to resolution.
    - Missing ownership of the task or issue that needs to be solved.
    - Examples: "I'm stuck until DevOps fixes it", "Not sure who this task belongs to", "somebody address this".

By leveraging Jira we can find the issues that demand the Director's attention better than solely relying on emails.
Jira can help keep track of how long has it been since the last update on each issue, assign issues to colleagues, and find unresolved issues.

### AI Process
1. **Preprocessing:**
    - Ensure that formatting is consistent accross emails (`Thread ID: [thread_id], Message ID [message_id], From: [sender_names] [sender_roles], To: [receiver_names] [receiver_roles], Cc: [cc_names] [cc_roles], Date: [date], Subject: [subject], Message: [body]`).
    - Aggregate thread emails into a list of emails following each other separated by two blank lines.

2. **Classification:**
    - Use an AI model to classify threads (e.g. gpt-4o-mini) based on if the emails contain any attention flags that haven't been resolved.
    - Fallback: keyword-based + time-based heuristic rules.
3. **Report Generation:** 
    - Group flagged emails by thread.
    - Sort by severity and recency.
    - Summarize into a concise report (Portfolio Health Report).

### Prompt engineering
```
You are an assistant that analyzes the full email thread and flags if the thread contains any attention flags that haven't been resolved.

Here are the attention flags:
- Unresolved High-Priority Issues:
    - Crucial issue that pose a threat to the system
    - Issue unresolved for a long period of time
    - Issue that were identified by high role members
- Emerging Risks or Blockers:
    - Issue which indicate no clear path to resolution
    - Issue missing ownership

For each flagged issue output JSON object:
{
    "title": str,
    "attention_flag": str,
    "priority": "low" | "medium" | "high" | null,
    "owner": str | null,
    "days_since_last_update": int,
    "evidence_quote": str,
    "evidence_location": {"thread_id": str, "message_id": str},
    "confidence": 0..1
}

- Use ONLY the provided text.
- If uncertain, set fields to null, never invent.
- Consider the entire conversation to decide if an issue is unresolved.
- If no issues are found, return an empty list [].

Thread: "{thread_text}"
```

### Minimizing Misinformation/Hallucination
Include a referee prompt to verify the claims to reduce misinformation and hallucinations.

**Referee Prompt:**
```
Given the thread of emails and flagged issues (with evidence and location) verify for each issue:
    1. The evidence_quote is word for word in the thread.
    2. The flagged issue has not been resolved within the thread.
    3. The output format follows the expected format.

Expected format:
{
    "title": str,
    "attention_flag": str,
    "priority": "low" | "medium" | "high" | null,
    "owner": str | null,
    "days_since_last_update": int,
    "evidence_quote": str,
    "evidence_location": {{"thread_id": str, "message_id": str}},
    "confidence": 0..1
}

Return the same list of issues, but add a field {"verified": True | False} to each issue.

Thread: "{thread_text}"
Flagged issues: "{flagged_issues}"
```

### Security Considerations
- **Data Privacy:** 
    - Anonymize names/emails in the preprocessing phase by attributing a alias for each name/email before sending to the AI via API.
    - Run local models instead of external APIs like `gpt-oss-120b` or `gpt-oss-20b` to keep the data private.

---

## 3. Cost & Robustness Considerations

- **Robustness:**
    - By avoiding using fixed keyword searches and instead using AI to handle informal language.
    - By setting the temperature of the AI to 0, to get consistent results.
    - By adding the referee prompt which double checks that the claims are valid.
    - By adding a human in the loop to revise low confidence AI claims.

- **Cost Management:**
    - By using lightweight models such as `gpt-4o-mini` and scale up to `gpt-4o` if results are poor.
    - By sending threads of emails into the model we reduce API calls which reduces cost.
    - By chunking long emails, we can reduce the amount of tokens which reduces cost.

---

## 4. Monitoring & Trust

- **Trustworthiness & Accuracy:**
    - By testing the system through mockup email threads to check for accuracy.
    - By artificially generating email treads to test the system's accuracy.

- **Metrics:**
    - Percentage of threads correctly flagged as having a critical attention flag (using accuracy)..
    - False positives/negatives checked by a human to improve the system.
    - Cost and latency per classification.

---

## 5. Architectural Risk & Mitigation

- **Biggest Risk:** Too long of a thread that cannot fit in the context window.
- **Mitigation:**
    - Generate intermediate summaries of early emails (while identifying the issues) and keep the last K messages intact to check if the issues have been resolved or not.
    - Pick K based on context window capacity.