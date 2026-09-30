---
layout: post
title: "Writing my own custom docintelligence"
date: 2025-07-30 00:00:00 +0530
categories: [azure, data engineering]
permalink: /blogs/docintelligence
---

New challenge! Since I work in a bank, we do a lot of document reviews. I cannot reveal all the details, so let me explain the problem with an example based on HDFC Bank (my salary account, haha).

Think of John. He has just started a new job as an AI engineer, but he does not have a bank account for his salary. He visits the bank with his Aadhaar card, passport, photograph, signatures, and several forms.

The teller tells him that his account details will arrive after verification, and the process could take 10 to 15 days.

John asks, "Why? You already have all my details. Just give me the account number!"

The teller explains, "First, we need to verify that you are who you say you are and that you live at the declared address. The KYC team checks your documents against the national database before approving the account."

John asks, "So they just run a query?"

"Not exactly," says the teller. "The documents are uploaded as scanned images. The KYC team has to read them, compare the details across documents, verify them, and then approve or reject the application. They already have a huge backlog."

John looks at the pile of forms and says, "Hmm. I am an AI engineer. Let me automate this process!"

## The actual problem

The data already exists, but it is trapped inside scanned forms, ID cards, and handwritten documents. Before any database query can run, someone has to extract and understand that data.

Traditional image processing can solve some of this, but it takes time to build and still struggles with skewed pages, poor lighting, handwriting, and low-quality scans. Passing an image to an LLM can help, but it is not automatically a reliable extraction pipeline. Results vary with image quality, document layout, and context. A model can also misread a field or return data in an unexpected format.

There are better tools for this job. OCR can extract text, while Azure AI Document Intelligence can analyze layout and return structured fields. Document Intelligence provides prebuilt models, and it also supports custom extraction models for organization-specific forms.

## Training a custom model

Suppose the bank wants to extract a customer's name, address, date of birth, PAN number, and account opening form ID. The basic process is:

1. Collect representative documents, including clean scans, poor scans, handwritten forms, blank fields, and different document variations.
2. Label the fields in the documents so the service knows what each value represents.
3. Train a custom extraction model using the labeled documents.
4. Test it on documents that were not used for training.
5. Improve the labels and training set when the model makes mistakes.

Good training data matters more than having a large collection of perfect examples. If production documents are noisy, the training set should contain noisy documents too. Otherwise, the model may perform well in a demo and fail in the real workflow.

After training, the model can be called from an application. This example sends a document to a custom model and reads the extracted fields:

```python
import os

from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.core.credentials import AzureKeyCredential


endpoint = os.environ["DOCUMENT_INTELLIGENCE_ENDPOINT"]
key = os.environ["DOCUMENT_INTELLIGENCE_KEY"]
model_id = os.environ["CUSTOM_MODEL_ID"]

client = DocumentIntelligenceClient(
	endpoint=endpoint,
	credential=AzureKeyCredential(key),
)

with open("account-opening-form.jpg", "rb") as document:
	poller = client.begin_analyze_document(
		model_id=model_id,
		body=document,
	)

result = poller.result()

for document in result.documents:
	for field_name, field in document.fields.items():
		print(field_name, field.value, field.confidence)
```

The model output is not the final KYC decision. It is structured input for the next step. The application still needs business rules to check required fields, compare names across documents, validate dates, and decide whether a case should be approved or sent for human review.

For example:

```python
def needs_manual_review(fields: dict) -> bool:
	required_fields = ["CustomerName", "DateOfBirth", "Address"]

	for field_name in required_fields:
		field = fields.get(field_name)
		if field is None or field.value in (None, ""):
			return True
		if field.confidence is not None and field.confidence < 0.85:
			return True

	return False
```

The confidence threshold above is only an example. In a real banking system, it should be chosen using representative test data and business risk. Sensitive fields may need stricter checks, and low-confidence results should usually go to a human rather than being rejected automatically.

## What John learned

John realized that the delay was not simply a database problem. The real bottleneck was turning inconsistent document images into trustworthy, structured data.

Document Intelligence does not replace the KYC process, and it does not decide whether a customer should receive an account. It helps with the document-understanding step. The database checks, business rules, fraud controls, privacy requirements, and human review process still belong around the model.

That is why I find Document Intelligence useful. It is more than OCR: it combines text extraction, layout understanding, and custom field extraction in a workflow that can be tested and improved.

My advice is simple: start with real document examples, define the fields that matter, label them carefully, test on unseen documents, and validate every important output. Once the extraction becomes consistent, the rest of the KYC workflow becomes much easier to automate.
