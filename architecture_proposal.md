# Document Extractor Architecture Proposal

## Document Service
The document service keeps the state of all documents. This is the service that all the other components communicate with.
It exposes a simple CRUD api for `Document`s and `Document Definition`s that the other components and external actors use.

External systems can add new documents for extraction by creating a `Document` with the state `INGESTED`. 
The document contains urls to the original files and metadata about the document, like the `source_system`, which indicates what system the document is coming from. And `domain` which can be used to vertically separate different use cases in the same system. Like `sbc`and `sbc_hk` 
If the type is not known at ingestion time, it should dbe set to `unknown` and a separate could be responsible for assigning the type at a later time.
The `type` field of the document normally determines what happens next.   
In the [invoice example sequence diagram](invoice_example.mermaid), the `Invoice Extractor` service will handle documents of type `invoice`, domain `sbc|sbc_hk` and state `INGESTED`.
It will run extraction logic and update document with extracted fields and other data that might be useful. If the `Invoice Extractor` can't with high confidence extract and validate all required fields of the invoice it will update the state of the document to `HUMAN_NEEDED` which indicated that it need manual processing mby a human.
The `Extractor UI` backend service will then include the document in the lists for employees working with manual extraction and validation of documents. When the `Extractor UI` backend service shows the invoice to the employee it will also present the editor controls configured in the `Document Definition` for type `ìnvoice`. The `Document Definition` contains the fields that are needed and the `editor id` which lets the Extractor UI present custom editors (UI widgets) for specific fields in specific document types. The implementation of the editors can be shared across document types or be specific for an document or an OpCo. For example, the `clientid` field in the example, has a editor configured with id `dynamic_dropdown` that is configured to fetch its data from a custom integration service that has data about SBCs client names and ids and will show them in a dropdown ui control.
When the employee is finished the Document is updated with manually extracted fields and the state is set to `COMPLETED`.
An external system (SBC ERP), will then query for the documents it is interested in. In this case documents with type `ìnvoice` and state `COMPLETED`. Optionally it can request a templated version of the document for easier integration.     


Webhooks with filters (like `type=invoice`, `state=INGESTED`) can also be configured for external systems so that they can be notified when document changes happen. The webhooks
are of the fire-and-forget kind, and if unsuccessful, will not be retried by the Document Service. The consuming services are supposed to poll the REST API for changes at regular intervals and the webhooks should ideally just trigger or short-circuit the already existing poll behaviour.

#### Implementation ideas

Simple json CRUD REST API backed by a postgres database. System fields are materialized in the db table but the all of the dynamic fields are simply stored as a json field in postgres. Updates to a document (by any system) is done with simple Json Merge Patch requests (https://zuplo.com/blog/2024/10/11/what-is-json-merge-patch). Validation logic on create/update makes sure that relevant fields are updated according to the lifecycle of the Document. External systems access the API through a reverse proxy (requiring API keys or SSO creds) and roles are set in a http header by the proxy. Role mappings are configured in reverse proxy (or authentication provider) and determine what is allowed in the API.  


### Document
This is an example of the model of a `Document`. It is shown in yaml fo readability. 

```yaml
documents:
  - id: 7a13efe4-651e-4038-8361-e7b8d87542a8
    displayName: 20241015 Faktura Brf Gråsparven - Inköp av bergvärmepump från Polarpumpen AB # optional pretty name 
    type: invoice
    domain: sbc-invoices # sbc or sbc-hk
    originalUrls:
      - https://blobstore/7a13efe4-651e-4038-8361-e7b8d87542a8/1.pdf
    isScanned: true # true if the originals are scanned, false if received through email or similar
    sourceSystem: pdu-scan-abby # identifies the system that the document arrived from, like invoices@sbc.se
    ingestionTs: #timestamp of ingestion
    state: HUMAN_NEEDED # the documents current state in the Document lifecycle
    lockedBy:
      - user: foobar@sbc.se
        ts:
    humanIntervention:
      - user: foobar@sbc.se
        startedTs:
        finishedTs:
      
    # these are added by extraction components, like the invoice-data-extractor service 
    extractionResults:
      - field: clientId
        value: S-12122
        confidence: 0
        meta:
          extractedBy: invoice-data-extractor
          extractedTs:
          bounding_box: 
          description: Could not match the extracted client id to a known client id
          alternative_values:
      - field: amount
        value: "145000.00"
        confidence: 100
        meta:
          extractedBy: invoice-data-extractor
          extractedTs:
          bounding_box:
      - field: currency
        confidence: 100
        value: SEK
        meta:
          extractedBy: invoice-data-extractor
          extractedTs:
          bounding_box:

    fields:
      clientId:
      amount: "145000.00"
      currency: SEK
      priority: high

    tags:
      - priority

    humanEditorComments:
      - user: foobar@sbc.se
        comment: this client id looks strange
        ts:

    temp:
      invoiceExtractorService:
        documentIntelligenceCache:
          query1: ["https://blobstore/7a13efe4-651e-4038-8361-e7b8d87542a8/1.pdf"]
```

### Document Definition
```yaml
documentDefinitions:
  - id: 7a13efe4-651e-4038-8361-e7b8d87542a8
    name: { en: "Invoice", sv: "Faktura" }
    type: invoice
    version: 1
    domain: sbc-invoices
    enabled: true
    fields:
      - id: 44ed7175-1140-4761-8ed3-7d05c7fcaae4
        name: client_id
        label: { en: "Client ID", sv: "Klientid" }
        order: 10
        required: true
        editor:
          id: dynamic_dropdown
          dropdownDataUrl: http://sbc-enricher-service:8080/client_ids
      - id: 44ed7175-1140-4761-8ed3-7d05c7fcaae4
        name: amount
        label: { en: "Amount", sv: "Belopp" }
        order: 20
        required: true
        editor:
          id: amount
        validators:
          - name: is_amount
            message: { en: "Must be an amount with zero or two decimals", sv: "Måste vara ett belopp med noll eller två decimaler" }
            regex: "^\d+(\.\d{2})?$"
      - id: 44ed7175-1140-4761-8ed3-7d05c7fcaae4
        name: currency
        label: { en: "Currency", sv: "Valuta" }
        order: 30
        required: true
        editor:
          id: dropdown
          label: 
          values: 
            - { id: sek, value: { en: "SEK", sv: "SEK" } }
            - { id: usd, value: { en: "USD", sv: "USD" } } 
    outputTemplates:
      - name: erp
        mimeType: application/json
        jinja: >
          {
            "id": "{{ document.id }}",
            "name": "{{ document.name }}",
            "type": "{{ document.type }}",
            "fields": [
              {% for field in document.fields %}
              {
                "type": "{{ field.type }}",
                "name": "{{ field.name }}",
                "value": "{{ field.value }}",
                "is_human_extracted": {{ field.human | lower }}{% if not loop.last %},{% endif %}
              }
              {% endfor %}
            ]
          }
      - name: csv
        mimeType: text/csv
        jinja: >
          id,name,type,client_id,amount,currency
          {{ document.id }},{{ document.name }},{{ document.type }},{{ document.fields["client_id"].value }},{{ document.fields["amount"].value }},{{ document.fields["currency"].value }}
          {% endfor %}
``` 


### Document Lifecycle
State can be assigned at any point but ideally maps to a lifecycle
INGESTED -> HUMAN_NEEDED -> COMPLETED or FAILED

## Document Extractor UI Backend
Talks to Document Service 

## Document Ingestor Service
Watches for files on external SFTP servers and ingests them to the Document Service

## Invoice Extraction Service
Custom service that does extraction of `Document`s with type `ìnvoice`.  