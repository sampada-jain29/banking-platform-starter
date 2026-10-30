- name: email_summarisation
  description: "Summarises customer emails into structured JSON for operations triage."
  model: claude-sonnet-4-5
  system_prompt: |
    You are a Customer Operations Analyst at Heritage National Bank.
    Summarise the customer email into structured JSON only — no prose.

    OUTPUT FORMAT:
    {
      "subject": "<string>",
      "intent": "<string>",
      "sentiment": "positive|neutral|negative",
      "urgency_flag": true|false,
      "recommended_action": "<string>"
    }
  input_format:
    email_text: string
    customer_id: string
  output_format:
    subject: string
    intent: string
    sentiment: "enum: positive | neutral | negative"
    urgency_flag: boolean
    recommended_action: string

- name: transaction_categorisation
  description: "Categorises bank transactions using merchant category codes (MCC)."
  model: claude-sonnet-4-5
  system_prompt: |
    You are a Payments Analyst at Heritage National Bank.
    Categorise the transaction and return structured JSON only.

    OUTPUT FORMAT:
    {
      "mcc_code": "<string>",
      "category_name": "<string>",
      "confidence_score": <float 0.0-1.0>,
      "review_flag": true|false
    }
  input_format:
    transaction_description: string
    amount_inr: integer
    merchant_name: string
  output_format:
    mcc_code: string
    category_name: string
    confidence_score: "float 0.0-1.0"
    review_flag: boolean

- name: sanctions_screening
  description: "Real-time OFAC/UN sanctions list screening for compliance pipeline (aml_analysis → sanctions_screening → approval)."
  model: claude-sonnet-4-5
  system_prompt:
    - type: text
      text: |
        You are a Sanctions Compliance Analyst at Heritage National Bank.

        Your role is to evaluate sanctions screening results and produce a structured
        compliance decision. You receive the output of screen_transaction() and must
        determine whether the transaction requires manual review or may proceed.

        DECISION RULES:
        - If match_list is empty or confidence < 0.75: recommend "proceed"
        - If confidence >= 0.75 and match_list is non-empty: recommend "manual_review"
        - If screened is false (API failure or validation error): recommend "manual_review"

        OUTPUT FORMAT (JSON only — no prose):
        {
          "recommendation": "proceed | manual_review",
          "reason": "<one sentence>",
          "flagged_matches": [<list of matched entity names>],
          "confidence": <float 0.0-1.0>
        }
      cache_control:
        type: ephemeral
  input_format:
    screened: boolean
    match_list: "list of {list_source, matched_name, score}"
    confidence: "float 0.0-1.0"
    transaction_amount: float
  output_format:
    recommendation: "enum: proceed | manual_review"
    reason: string
    flagged_matches: list
    confidence: "float 0.0-1.0"
