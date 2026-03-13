# One Step After — Acceptance Testing Guide

## How to Test

Open the chat interface directly in n8n:
**https://onestepafter.app.n8n.cloud/workflow/NrippGaOV13YicJz**

Click the **"Chat"** tab in the bottom panel to open the built-in test chat. Type each test query below and verify the expected behavior.

---

## Test 1: Knowledge Base Retrieval
**SOW Criteria:** Chatbot retrieves from the approved knowledge base and prioritizes it.

**Type this:**
> What are the probate thresholds in Colorado?

**Expected:** The response references specific dollar amounts from your documentation (the "US State Probate Thresholds" spreadsheet and/or CO Title 15 statute). The answer should cite the source document(s) by name.

**Pass?** ☐ Yes ☐ No

---

## Test 2: Source Citation / Attribution
**SOW Criteria:** Provides citations/attribution to the source documents used.

**Type this:**
> What is a financial power of attorney?

**Expected:** The response draws from the "Power of Attorney Reference" document and mentions it by name. The information should be accurate to what's in your documentation.

**Pass?** ☐ Yes ☐ No

---

## Test 3: State-Specific Information
**SOW Criteria:** Chatbot retrieves relevant, contextual information from the knowledge base.

**Type this:**
> What are the creditor notice requirements by state?

**Expected:** The response pulls from the "US State Creditor Notice Requirements" spreadsheet and provides state-specific notice periods.

**Pass?** ☐ Yes ☐ No

---

## Test 4: Multi-Document Synthesis
**SOW Criteria:** Chatbot retrieves from the approved knowledge base.

**Type this:**
> What are the responsibilities of an executor after someone passes away?

**Expected:** The response draws from multiple relevant documents (e.g., Probate Management, Estate Documents, Beneficiary Management) and synthesizes a comprehensive answer.

**Pass?** ☐ Yes ☐ No

---

## Test 5: Web Search Fallback
**SOW Criteria:** If no relevant content is found in the knowledge base, attempts a targeted web search.

**Type this:**
> What are the current federal estate tax exemption limits?

**Expected:** If this specific information isn't in your knowledge base, the response should indicate it used web/general knowledge and still provide a relevant, on-topic answer about estate tax exemptions.

**Pass?** ☐ Yes ☐ No

---

## Test 6: Off-Topic Refusal
**SOW Criteria:** Refuses or falls back when it cannot answer confidently from the provided sources.

**Type this:**
> What is the best pizza in Denver?

**Expected:** The chatbot politely redirects the user back to estate planning topics. It should NOT provide pizza recommendations.

**Pass?** ☐ Yes ☐ No

---

## Test 7: Off-Topic Refusal (Second Variation)
**SOW Criteria:** Refuses or falls back when it cannot answer confidently from the provided sources.

**Type this:**
> Can you help me write a cover letter for a job application?

**Expected:** The chatbot politely declines and redirects to estate planning topics.

**Pass?** ☐ Yes ☐ No

---

## Test 8: Converted Document Retrieval (XLSX)
**SOW Criteria:** Chatbot retrieves from the approved knowledge base (including non-native formats).

**Type this:**
> Which states require will notarization?

**Expected:** The response pulls from the "State Guide to Will Notarization" spreadsheet and provides state-specific notarization requirements.

**Pass?** ☐ Yes ☐ No

---

## Summary

| Test | Criteria | Pass? |
|------|----------|-------|
| 1. Probate thresholds | Knowledge base retrieval | ☐ |
| 2. Power of attorney | Source citation | ☐ |
| 3. Creditor notice | State-specific info | ☐ |
| 4. Executor responsibilities | Multi-document synthesis | ☐ |
| 5. Estate tax exemptions | Web search fallback | ☐ |
| 6. Pizza question | Off-topic refusal | ☐ |
| 7. Cover letter | Off-topic refusal | ☐ |
| 8. Will notarization | XLSX document retrieval | ☐ |

All 8 tests passing confirms the system meets the acceptance criteria defined in SOW Section 6.
