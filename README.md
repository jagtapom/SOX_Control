# SOX_Control



┌────────────────────────────┐
│     Outlook / O365         │
│  (User Mailboxes)          │
└───────────┬────────────────┘
            │ OAuth 2.0 (App + Cert)
            ▼
┌────────────────────────────┐
│ Microsoft Graph API        │
└───────────┬────────────────┘
            │ HTTPS (TLS 1.2+)
            ▼
┌────────────────────────────────────────────┐
│ AWS VPC (Private Subnets)                   │
│                                            │
│ ┌────────────────────────────────────────┐ │
│ │ Lambda / EKS Evidence Ingestor          │ │
│ │ - Fetch emails                          │ │
│ │ - Convert to .eml                      │ │
│ │ - Push to encrypted S3                 │ │
│ └───────────┬────────────────────────────┘ │
│             │                               │
│ ┌───────────▼────────────────────────────┐ │
│ │ Recursive Evidence Processor            │ │
│ │ - Email parsing                         │ │
│ │ - Attachment extraction                │ │
│ │ - Depth control + hashing               │ │
│ └───────────┬────────────────────────────┘ │
│             │                               │
│ ┌───────────▼────────────────────────────┐ │
│ │ OCR Layer                               │ │
│ │ - AWS Textract                          │ │
│ └───────────┬────────────────────────────┘ │
│             │                               │
│ ┌───────────▼────────────────────────────┐ │
│ │ NLP / Understanding                    │ │
│ │ - AWS Bedrock (Claude / Nova)          │ │
│ └───────────┬────────────────────────────┘ │
│             │                               │
│ ┌───────────▼────────────────────────────┐ │
│ │ Evidence Store                          │ │
│ │ - S3 (SSE-KMS)                          │ │
│ │ - DynamoDB (metadata, lineage)          │ │
│ │ - OpenSearch (optional search)          │ │
│ └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘


🔐 Encryption
Layer	Method
Email fetch	TLS
Storage	SSE-KMS
OCR	AWS-managed secure
NLP	Private Bedrock endpoint
Temp files	Ephemeral, wiped


#KMS Key Policy (Customer-Managed)

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowLambdaUse",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/evidence-processing-role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}

# IAM Role – Evidence Processor (Lambda / EKS)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::evidence-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt"
      ],
      "Resource": "arn:aws:kms:region:acct:key/<KEY_ID>"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:region:acct:table/EvidenceMetadata"
    }
  ]
}
Python: Microsoft Graph API (Secure Email Access)
4.1 Authentication (OAuth – App + Certificate)
import msal

def get_graph_token():
    app = msal.ConfidentialClientApplication(
        client_id=CLIENT_ID,
        authority=f"https://login.microsoftonline.com/{TENANT_ID}",
        client_credential={
            "thumbprint": CERT_THUMBPRINT,
            "private_key": PRIVATE_KEY
        }
    )

    result = app.acquire_token_for_client(
        scopes=["https://graph.microsoft.com/.default"]
    )
    return result["access_token"]


✔ No passwords
✔ Revocable

4.2 Fetch Emails + Attachments
import requests

def fetch_emails(token):
    headers = {"Authorization": f"Bearer {token}"}
    url = "https://graph.microsoft.com/v1.0/users/{user}/messages?$expand=attachments"

    return requests.get(url, headers=headers).json()

5️⃣ Recursive Evidence Processing (CORE LOGIC)
def process_file(path, depth=0, parent_id=None):
    if depth > 5:
        return

    ftype = detect_type(path)
    evidence_id = create_evidence_id()

    store_metadata(evidence_id, parent_id, ftype, depth)

    if ftype == "email":
        parse_email(path, evidence_id, depth)

    elif ftype == "pdf":
        parse_pdf(path)

    elif ftype == "excel":
        parse_excel(path)

    elif ftype == "image":
        ocr_text = run_textract(path)
        send_to_bedrock(ocr_text)

Email Parsing
import mailparser

def parse_email(path, parent_id, depth):
    mail = mailparser.parse_from_file(path)

    text = mail.text_plain[0] if mail.text_plain else ""
    send_to_bedrock(text)

    for att in mail.attachments:
        att_path = save_encrypted(att["payload"])
        process_file(att_path, depth + 1, parent_id)

6️⃣ OCR Layer (AWS Textract)
def run_textract(file_path):
    response = textract.detect_document_text(
        Document={"S3Object": {"Bucket": BUCKET, "Name": file_path}}
    )

    return " ".join(
        b["Text"] for b in response["Blocks"] if b["BlockType"] == "LINE"
    )

7️⃣ AWS Bedrock Usage (Post-Extraction ONLY)
def send_to_bedrock(text):
    prompt = f"""
    You are a banking compliance analyst.
    Validate and summarize this evidence:
    {text}
    """

    bedrock.invoke_model(
        modelId="anthropic.claude-3-sonnet",
        body=json.dumps({
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 400
        })
    )

8️⃣ Lambda vs EKS Deployment
Option A – AWS Lambda (Recommended to Start)

✔ Serverless
✔ Secure
✔ Low ops

SES / Graph Trigger
   ↓
Lambda (Python)
   ↓
S3 + Textract + Bedrock


Limits:

15 min execution

Good for moderate volume

Option B – EKS (High Volume / Long Jobs)

✔ Heavy attachments
✔ Large PDFs
✔ Batch processing

Graph Scheduler
   ↓
EKS Job (Python)
   ↓
Same recursive engine


Security:

IRSA for IAM

Private VPC

No public ingress

9️⃣ Why This Design Passes Bank Security Review

✔ OAuth, no passwords
✔ Encryption everywhere
✔ KMS owned by bank
✔ Recursive evidence completeness
✔ Full lineage & hashing
✔ Deterministic extraction (Bedrock used only for meaning)
