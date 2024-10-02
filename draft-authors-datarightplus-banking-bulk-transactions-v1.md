%%%
title = "DataRight+: Bulk Transaction Detail"
area = "Internet"
workgroup = "datarightplus"
submissionType = "independent"

[seriesInfo]
name = "Internet-Draft"
value = "draft-authors-datarightplus-banking-bulk-transactions-v1-latest"
stream = "independent"
status = "experimental"

date = 2024-07-22T00:00:00Z

[[author]]
initials="S."
surname="Low"
fullname="Stuart Low"
organization="Biza.io"
  [author.address]
  email = "stuart@biza.io"


%%%

.# Abstract

This specification outlines the DataRight+ extension functionality for an asynchronous bulk banking transaction detail capability within the existing constructs of the profile described in [@!PROFILE-AU-CDR].

The functionality exposed by the Provider allows the Initiator to request, using the same filter parameters of the [@!CDS] List Banking Transactions, a full set of Banking Transaction Detail via a job queue, poll and retrieval mechanism.

.# Notational Conventions

The keywords  "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",  "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [@!RFC2119].

{mainmatter}

# Scope

The scope of this specification is focused on describing an asynchronous capability for downloading bulk banking transactions from a specific Provider supplied resource server. This specification does not seek to provide further definition as this is left to what is specified in [@!PROFILE-AU-CDR].

# Terms & Definitions

This specification uses the terms "Provider" and "Initiator" as defined by [@!DATARIGHTPLUS-ROSETTA].

# High Level Process

This document, as extended further in OpenAPI format within [@!DATARIGHTPLUS-REDOCLY-ID2], describes the endpoints to deliver the Bulk Banking Transaction Detail capability. An identified challenge faced by participants within the Australian CDR is a desire of Initiators to obtain Banking Transaction Details for all transactions while balancing the performance constraints of Providers. On the one hand Initiators are often needing to iterate over lists of transactions to obtain, on an individual basis, transaction details, sometimes hitting the upper bounds of the prescribed Non-Functional Requirements while on the other hand Providers often must interact with many different sources of data to provide all the attributes contained within the `BankingTransactionDetailV1` schema.

At a high level the process expected to be followed through the implementation of the prescribed endpoints is as follows:

1. Initiator obtains a Consumer approved token, containing the `bank:transaction:read` and `bank:accounts.basic:read` scopes from the Provider through normal processes as prescribed within [@!PROFILE-AU-CDR], [@!CDS] and [@!CDR-RULES];
2. Initiator obtains a list of banking accounts, using the `listBankingAccounts` operation, as described in [@!DATARIGHTPLUS-REDOCLY-ID2]. Further elaboration regarding this endpoint is outside the scope of this document;
3. Initiator lodges a `requestBankingTransactionDetailList` request, including account identifier and desired filters, from the Provider Resource Server;
4. The Provider responds with metadata including the Action Identifier to be used to address the action by the Initiator;
5. Periodically the Initiator polls the Provider Resource Server using the `getBankingTransactionDetailListStatus` operation until the status returned indicates readiness;
6. When the Provider indicates readiness, Initiator calls the `retrieveBankingTransactionDetailList`, the Provider responds with a continuous list of `BankingTransactionDetailV1` objects;

# Provider Resource Server Endpoints

The Provider **SHALL** make available, as described further in the following subsections and specified in OpenAPI format within [@!DATARIGHTPLUS-REDOCLY-ID2], the following endpoints where the token is granted the `bank:transactions:read` scope value:

| Resource Server Endpoint                                     | `x-v` | Description                                                                                                               |
|--------------------------------------------------------------|-------|---------------------------------------------------------------------------------------------------------------------------|
| `POST /actions/bulk-banking-transactions`                    | `1`   | Used to lodge Banking Transaction Detail List requests.                                                                   |
|                                                              |       | Expects a wrapped [Request Banking Transaction Detail List](#request-banking-transaction-detail-list) object to           |
|                                                              |       | be submitted and returns a payload containing [Action Status Metadata](#action-status-metadata)                           |
| `GET /actions/bulk-banking-transactions/{actionId}`          | `1`   | Returns the current status of request. Accepts an `actionId` as a path parameter and returns a payload containing         |
|                                                              |       | [Action Status Metadata](#action-status-metadata) as well as the original                                                 |
|                                                              |       | [Request Banking Transaction Detail List](#request-banking-transaction-detail-list) request content                       |
| `GET /actions/bulk-banking-transactions/{actionId}/retrieve` | `1`   | Provides the result payload. Accepts an `actionId` as a path parameter and returns a payload containing a list of         |
|                                                              |       | [Banking Transaction Detail](#banking-transaction-detail). <br/><br/>If the action is not in `COMPLETE` status returns    |
|                                                              |       | `urn:au-cds:error:cds-all:Resource/Unavailable` error as described in [@!CDS]. <br/><br/>If the action is in `EXPIRED` or |
|                                                              |       | `FAILED` status returns `urn:au-cds:error:cds-all:Resource/Invalid` error as described in [@!CDS]                         |


Note: Each endpoint supports a number of other error behaviours, these are documented within [@!DATARIGHTPLUS-REDOCLY-ID2] and are intended to align with the behaviours described within [@!CDS].


# Request Detail Endpoint

The endpoint for Request Detail Endpoint is advertised according to [@!DATARIGHTPLUS-DISCOVERY] as the `requestBankingTransactionDetailList` operation. A request is made using HTTP `POST` method containing a JSON encoded payload according to the schema defined within [@!DATARIGHTPLUS-REDOCLY-ID2] as `RequestBankingTransactionDetailListDataV1` and containing the following attributes:

| Attribute    | Requirement  | Type                    | Description                                                                                                                                                 |
|--------------|--------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `accountId`  | **REQUIRED** | String                  | ID of a specific account to obtain data for. This is a tokenised ID previously obtained from the List Banking Accounts endpoint.                            |
| `oldestTime` | **REQUIRED** | String (DateTimeString) | Constrain the transaction history request to transactions with effective time at or after this date/time. Format is aligned to DateTimeString common type.  |
| `newestTime` | **REQUIRED** | String (DateTimeString) | Constrain the transaction history request to transactions with effective time at or before this date/time. Format is aligned to DateTimeString common type. |
| `minAmount`  | **OPTIONAL** | String (AmountString)   | Filter transactions to only transactions with amounts higher or equal to than this amount                                                                   |
| `maxAmount`  | **OPTIONAL** | String (AmountString)   | Filter transactions to only transactions with amounts less than or equal to than this amount                                                                |

**Note:** For requests related to this specification the Initiator **SHALL NOT** include any other attributes than those documented above.

The Provider responds to this request with a JSON encoded payload according to the schema defined within [@!DATARIGHTPLUS-REDOCLY-ID2] as `ResponseGetBankingTransactionDetailListStatusV1` and containing the following attributes:

| Attribute | Requirement  | Type                                        | Description                                                                                  |
|-----------|--------------|---------------------------------------------|----------------------------------------------------------------------------------------------|
| `version` | **REQUIRED** | String                                      | The version of the payload, currently only `V1` is supported                                 |
| `data`    | **REQUIRED** | `RequestBankingTransactionDetailListDataV1` | Contains the original request data specified as `RequestBankingTransactionDetailListDataV1`  |
| `links`   | **REQUIRED** | `LinksV1`                                   | [@!CDS] aligned `LinksV1` object                                                             |
| `meta`    | **REQUIRED** | `MetaRequestBankingTransactionDetailListV1` | Contains the job status information described as `MetaRequestBankingTransactionDetailListV1` |

## Request Detail Metadata

Defined within [@!DATARIGHTPLUS-REDOCLY-ID2] as `MetaRequestBankingTransactionDetailListV1` this schema provides a high level status of the detail request containing the following attributes:

| Attribute           | Requirement  | Type               | Description                                                                                            |
|---------------------|--------------|--------------------|--------------------------------------------------------------------------------------------------------|
| `actionId`          | **REQUIRED** | String             | Globally unique identifier for this action. **MUST** not be reused.                                    |
| `status`            | **REQUIRED** | String (Enum)      | Status of the action based on one of the enumerations described in [Action Statuses](#action-statuses) |
| `statusDescription` | **REQUIRED** | String             | Extended description of the status containing content at the discretion of the Provider                |
| `creationDateTime`  | **REQUIRED** | String (Date-Time) | Date/Time in RFC3339 representing the creation time of the action                                      |
| `lastUpdated`       | **REQUIRED** | String (Date-Time) | Date/Time in RFC3339 representing the last update time of the action                                   |

### Action Statuses

The following `status` values are defined for this specification.

| Status       | Description                                                                               |
|--------------|-------------------------------------------------------------------------------------------|
| `INITIALISE` | Initialised but processing has not been started.                                          |
| `PROCESSING` | Currently processing and being prepared for download.                                     |
| `COMPLETE`   | Completed and available for download at the Retrieve Detail List endpoint.                |
| `EXPIRED`    | Completed successfully but no longer available for download.                              |
| `FAILED`     | Failed to complete successfully, further information available within `statusDescription` |



## Request Detail Endpoint Request

The following is a non-normative example of a request from the Initiator to the Provider:

```
POST /dio-au/actions/bulk-banking-transactions
Host: api.provider.com.au
Content-Type: application/json
Accept: application/json
x-v: V1

{
  "version": "V1",
  "data": {
    "accountId": "6ecc1b30-5889-49d4-a988-e44b4e215574",
    "oldestTime": "2022-07-01T15:43:00.123456Z",
    "newestTime": "2024-06-30T19:20:30.123456Z"
  }
}
```

## Request Detail Endpoint Successful Response

The following is a non-normative example of a response from the Provider:

```json
{
  "version": "V1",
  "data": {
    "accountId": "6ecc1b30-5889-49d4-a988-e44b4e215574",
    "oldestTime": "2022-07-01T15:43:00.123456Z",
    "newestTime": "2024-06-30T19:20:30.123456Z"
  },
  "links": {
    "self": "https://api.provider.com.au/actions/bulk-banking-transactions/9fe3f97e-c22c-4516-b6ed-05c0486db195"
  },
  "meta": {
    "actionId": "9fe3f97e-c22c-4516-b6ed-05c0486db195",
    "status": "PROCESSING",
    "statusDescription": "Job Received, contacting internal systems to formulate response."
  }
}
```

# Bulk Transaction Detail Status

The endpoint for Bulk Transaction Detail Status is advertised according to [@!DATARIGHTPLUS-DISCOVERY] as the `getBankingTransactionDetailListStatus` operation. A request is made using HTTP `GET` method and includes the unique `actionId` returned during the original request and the response matches that of the _Request Detail Endpoint_.


## Bulk Transaction Detail Request

The following is a non-normative example of a request from the Initiator to the Provider:

```
GET /dio-au/actions/bulk-banking-transactions/9fe3f97e-c22c-4516-b6ed-05c0486db195
Host: api.provider.com.au
Accept: application/json
x-v: V1
```

## Successful Bulk Transaction Detail Response

The following is a non-normative example of a response from the Provider:

```json
{
  "version": "V1",
  "data": {
    "accountId": "6ecc1b30-5889-49d4-a988-e44b4e215574",
    "oldestTime": "2022-07-01T15:43:00.123456Z",
    "newestTime": "2024-06-30T19:20:30.123456Z"
  },
  "links": {
    "self": "https://api.provider.com.au/dio-au/v1/actions/bulk-banking-transactions/9fe3f97e-c22c-4516-b6ed-05c0486db195"
  },
  "meta": {
    "actionId": "9fe3f97e-c22c-4516-b6ed-05c0486db195",
    "status": "PROCESSING",
    "statusDescription": "Job Received, contacting internal systems to formulate response."
  }
}
```

# Retrieve Detail List

The endpoint for Retrieve Detail List is advertised according to [@!DATARIGHTPLUS-DISCOVERY] as the `retrieveBankingTransactionDetailList` operation. A request is made using HTTP `GET` method and includes the unique `actionId` returned during the original request.

The Provider responds to this request with a JSON encoded payload according to the schema defined within [@!DATARIGHTPLUS-REDOCLY-ID2] as `ResponseRetrieveBankingTransactionDetailListV1` and containing the following attributes:

| Attribute | Requirement  | Type                               | Description                                                                                                       |
|-----------|--------------|------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| `version` | **REQUIRED** | String                             | The version of the payload, currently only `V1` is supported                                                      |
| `data`    | **REQUIRED** | List(`BankingTransactionDetailV1`) | Contains an array of `BankingTransactionDetailV1` objects as defined in [@!DATARIGHTPLUS-RESOURCE-SET-BANKING-00] |
| `links`   | **REQUIRED** | `LinksV1`                          | [@!CDS] aligned `LinksV1` object                                                                                  |
| `meta`    | **REQUIRED** | `MetaPaginatedV1`                  | [@!CDS] aligned `MetaPaginatedV1` object                                                                          |
The payload provided, if ready, **MUST** be a single page of unlimited records. If the result payload is not available the Provider **MUST** respond with a HTTP Code 404 containing an error payload matching the `urn:au-cds:error:cds-all:Resource/Unavailable` error code.

## Retrieve Detail Request

The following is a non-normative example of a request from the Initiator to the Provider:

```
GET /dio-au/actions/bulk-banking-transactions/9fe3f97e-c22c-4516-b6ed-05c0486db195/retrieve
Host: api.provider.com.au
Accept: application/json
x-v: V1
```

## Successful Retrieve Detail Response

The following is a non-normative example of a response from the Provider:

```json
{
  "version": "V1",
  "data": [
    {
      "accountId": "jdnmwucgrchzngrqieryhiovpvzknokkwsediggdlnxgxizegvwpwfflgavkrbay",
      "amount": "133.55",
      "apcaNumber": "484799",
      "billerCode": "75556",
      "billerName": "TAX OFFICE PAYMENTS",
      "crn": "1234123412341",
      "currency": "AUD",
      "description": "Outbound payment to ATO",
      "executionDateTime": "2022-03-03T03:03:03.3Z",
      "isDetailAvailable": true,
      "merchantCategoryCode": "5200",
      "merchantName": "Freedom Furniture",
      "postingDateTime": "2022-03-03T03:03:03.3Z",
      "reference": "Payment for Services",
      "status": "POSTED",
      "transactionId": "8cce04fbcc0fe8307e8c221b5ae497691935a368e98f5478610c60ca4ef81caf",
      "type": "PAYMENT",
      "valueDateTime": "2022-03-13T03:03:03.3Z",
      "extendedData": {
        "extensionUType": "x2p101Payload",
        "payee": "string",
        "payer": "string",
        "service": "X2P1.01",
        "x2p101Payload": {
          "endToEndId": "string",
          "extendedDescription": "An extended string description.",
          "purposeCode": "string"
        }
      }
    }
  ],
  "links": {
    "self": "https://api.provider.com.au/dio-au/v1/actions/bulk-banking-transactions/9fe3f97e-c22c-4516-b6ed-05c0486db195/retrieve"
  },
  "meta": {
    "totalPages": 1,
    "totalRecords": 9918
  }
}
```

## Retrieve Detail Not Ready Error Response

The following is a non-normative example of an error response from the Provider when the detail is not yet ready for download:

```json
{
  "errors": [
    {
      "code": "urn:au-cds:error:cds-all:Resource/Unavailable",
      "title": "Unavailable Resource",
      "detail": "The requested resource identifier not currently available.",
      "meta": {}
    }
  ]
}
```

# Implementation Considerations

The availability of this functionality is expected to be discovered using the specification outlined in [@!DATARIGHTPLUS-DISCOVERY].
The version negotiation utilised by this specification is outlined within [@!DATARIGHTPLUS-ENHANCED-VERSIONING].

`RequestBankingTransactionDetailListDataV1` is intended to be a like-for-like to the input parameters for List Transactions for Banking Account as outlined within [@!CDS].

While it is not explicitly specified within the API documentation the assigned `actionId` **SHOULD** be no more than 512 characters.

Some implementations **MAY** encounter hard limits with respect to maximum response size, for instance AWS infrastructure is known to be limited to 10MB. The behaviour of the Provider in this situation is not currently defined but various error codes exist within the OpenAPI specification which **MAY** be useful.

{backmatter}

<reference anchor="CDS" target="https://consumerdatastandardsaustralia.github.io/standards"><front><title>Consumer Data Standards (CDS)</title><author><organization>Data Standards Body (Treasury)</organization></author></front> </reference>

<reference anchor="CDR-RULES" target="https://www.legislation.gov.au/F2020L00094/2023-07-22/text"> <front><title>Competition and Consumer (Consumer Data Right) Rules 2020</title><author><organization>Department of the Treasury</organization></author></front> </reference>

<reference anchor="PROFILE-AU-CDR" target="https://datarightplus.github.io/datarightplus-cdr-profile/draft-authors-datarightplus-cdr-profile.html"> <front><title>DataRight+: Australian CDR Profile</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-ROSETTA" target="https://datarightplus.github.io/datarightplus-rosetta/draft-authors-datarightplus-rosetta.html"> <front><title>DataRight+ Rosetta Stone</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-REDOCLY-ID2" target="https://datarightplus.github.io/datarightplus-redocly/?v=ID2"> <front><title>DataRight+: Redocly (ID2)</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author><author initials="B." surname="Kolera" fullname="Ben Kolera"><organization>Biza.io</organization></author>
<author initials="W." surname="Cai" fullname="Wei Cai"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-DISCOVERY" target="https://datarightplus.github.io/datarightplus-discovery/draft-authors-datarightplus-discovery.html"> <front><title>DataRight+: Discovery</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>

<reference anchor="DATARIGHTPLUS-ENHANCED-VERSIONING" target="https://datarightplus.github.io/datarightplus-enhanced-versioning/draft-authors-datarightplus-enhanced-versioning.html"> <front><title>DataRight+: Enhanced Endpoint Versioning</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>


<reference anchor="DATARIGHTPLUS-RESOURCE-SET-BANKING-00" target="https://datarightplus.github.io/datarightplus-resource-set-banking/draft-authors-datarightplus-resource-set-banking-00/draft-authors-datarightplus-resource-set-banking.html"> <front><title>DataRight+: Banking Resource Set</title><author initials="S." surname="Low" fullname="Stuart Low"><organization>Biza.io</organization></author></front> </reference>
