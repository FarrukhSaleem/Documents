# CMS Wrapper API - Low Level Diagram

Source: `CMSWrapperAPI.pdf`

Scope: This document diagrams only the APIs explicitly listed in the supplied API document. It does not add project-specific routes, classes, handlers, database objects, queues, or APIs that are not present in the PDF.

## Documented API Set

| API | Purpose | Documented endpoint |
| --- | --- | --- |
| Token | Generate JWT bearer token | `POST /api/token/getToken` |
| GetPrintLocation | Get print locations for cheque and pay order | `POST /api/endpoint/getprintlocation` |
| GetDeliverTo | Get delivery types | `POST /api/endpoint/getdeliverto` |
| GetProduct | Get product list | `POST /api/endpoint/getproduct` |
| GetAccount | Get account list for product | `POST /api/endpoint/getaccount` |
| GetInstrument | Get instrument number | `POST /api/endpoint/getinstrument` |
| GetFileTemplate | Get file template list | `POST /api/endpoint/getfiletemplate` |
| TitleFetch | Fetch account title | `POST /api/endpoint/titlefetch` |
| TransSubmit | Submit transaction | `POST /api/endpoint/transsubmit` |
| GetStatus | Get transaction status | `POST /api/endpoint/getstatus` |
| ResendTxn | Resend transaction | `POST /api/endpoint/resendtxn` |
| RejectTxn | Reject transaction | `POST /api/endpoint/rejecttxn` |

Document notes:

- The Web Service URL table lists `TitleFetch` as `/api/endpoint/titlefetch`, while section `4.4.8` shows `/api/endpoint/getfiletemplate`. The diagram uses `/api/endpoint/titlefetch` from the API list.
- The Web Service URL table lists `ResendTxn` as `/api/endpoint/resendtxn`, while section `4.4.11` shows `/api/endpoint/getstatus`. The diagram uses `/api/endpoint/resendtxn` from the API list.
- Section `4.4.12` labels `RejectTxn` purpose as "Resend Transaction", but the API name, endpoint, request, and response describe rejection. The diagram treats it as reject transaction.

## API Boundary Diagram

```mermaid
flowchart LR
    Client["Client application"]

    subgraph CMS["CMS Wrapper API"]
        Token["Token<br/>POST /api/token/getToken<br/>ID, Password -> accessToken"]
        Jwt["JWT bearer token verification"]

        subgraph Lookups["Lookup APIs"]
            PrintLocation["GetPrintLocation<br/>/api/endpoint/getprintlocation"]
            DeliverTo["GetDeliverTo<br/>/api/endpoint/getdeliverto"]
            Product["GetProduct<br/>/api/endpoint/getproduct"]
            Account["GetAccount<br/>/api/endpoint/getaccount"]
            Instrument["GetInstrument<br/>/api/endpoint/getinstrument"]
            FileTemplate["GetFileTemplate<br/>/api/endpoint/getfiletemplate"]
            TitleFetch["TitleFetch<br/>/api/endpoint/titlefetch"]
        end

        subgraph Txn["Transaction APIs"]
            TransSubmit["TransSubmit<br/>/api/endpoint/transsubmit"]
            GetStatus["GetStatus<br/>/api/endpoint/getstatus"]
            ResendTxn["ResendTxn<br/>/api/endpoint/resendtxn"]
            RejectTxn["RejectTxn<br/>/api/endpoint/rejecttxn"]
        end
    end

    Vendor["TransPaymentAPI vendor endpoints"]
    CoreDmz["Core / DMZ systems"]

    Client -->|"POST credentials"| Token
    Token -->|"JWT accessToken"| Client
    Client -->|"Bearer token + JSON request"| Jwt

    Jwt --> PrintLocation
    Jwt --> DeliverTo
    Jwt --> Product
    Jwt --> Account
    Jwt --> Instrument
    Jwt --> FileTemplate
    Jwt --> TitleFetch
    Jwt --> TransSubmit
    Jwt --> GetStatus
    Jwt --> ResendTxn
    Jwt --> RejectTxn

    PrintLocation --> Vendor
    DeliverTo --> Vendor
    Product --> Vendor
    Account --> Vendor
    Instrument --> Vendor
    FileTemplate --> Vendor
    TitleFetch --> Vendor
    TransSubmit --> Vendor
    GetStatus --> Vendor
    ResendTxn --> Vendor
    RejectTxn --> Vendor

    Vendor --> CoreDmz
```

## Common Runtime Sequence

```mermaid
sequenceDiagram
    autonumber
    participant C as Client application
    participant T as Token API
    participant W as CMS Wrapper endpoint
    participant V as TransPaymentAPI / Core-DMZ

    C->>T: POST /api/token/getToken {ID, Password}
    T-->>C: {message, accessToken}

    C->>W: POST one documented /api/endpoint/* API with Bearer JWT
    W->>W: Verify JWT token
    W->>W: Validate documented mandatory fields
    W->>V: Forward mapped request to vendor endpoint
    V-->>W: ResponseCode, ResponseDescription, optional TransDetail
    W-->>C: Return documented response shape
```

## Transaction Lifecycle Using Documented APIs Only

```mermaid
flowchart TD
    Start(["Start"])
    TokenCall["Token<br/>Get JWT accessToken"]
    Product["GetProduct<br/>Read PRODUCT_CODE values"]
    Account["GetAccount<br/>Requires ProductCode<br/>Returns ACCOUNTNO, ACCOUNTNAME"]
    Instrument["GetInstrument<br/>Requires ProductCode<br/>Returns INSTRUMENTNUMBER"]
    FileTemplate["GetFileTemplate<br/>Requires ProductCode<br/>Returns CONF_DEF_ID, CONF_DEF_DESC"]
    PrintLocation["GetPrintLocation<br/>Returns LOCATIONCODE, LOCATIONNAME"]
    DeliverTo["GetDeliverTo<br/>Returns delivery type fields"]
    TitleFetch["TitleFetch<br/>Requires Bank, AccountNo<br/>Returns account title in ResponseDescription"]
    Prepare["Prepare TransSubmit payload<br/>Header + Transactions[] + Invoice[]"]
    Submit["TransSubmit<br/>Submit transaction batch"]
    Status["GetStatus<br/>Requires MakerID, REFNO"]
    Decision{"Post-submit action needed?"}
    Resend["ResendTxn<br/>Requires MakerID, REFNO"]
    Reject["RejectTxn<br/>Requires MakerID, REFNO, Remarks"]
    Done(["End"])

    Start --> TokenCall
    TokenCall --> Product
    Product --> Account
    Product --> Instrument
    Product --> FileTemplate
    TokenCall --> PrintLocation
    TokenCall --> DeliverTo
    TokenCall --> TitleFetch
    Account --> Prepare
    Instrument --> Prepare
    FileTemplate --> Prepare
    PrintLocation --> Prepare
    DeliverTo --> Prepare
    TitleFetch --> Prepare
    Prepare --> Submit
    Submit --> Status
    Status --> Decision
    Decision -->|"No"| Done
    Decision -->|"Resend"| Resend
    Decision -->|"Reject"| Reject
    Resend --> Done
    Reject --> Done
```

## Request Shape Diagram

```mermaid
classDiagram
    class TokenRequest {
        +ID M
        +Password M
    }

    class TokenResponse {
        +message
        +accessToken
    }

    class CommonLookupRequest {
        +ChannelID M
        +Stan M
        +DateTime M
        +MakerID M
    }

    class ProductScopedLookupRequest {
        +ChannelID M
        +Stan M
        +DateTime M
        +MakerID M
        +ProductCode M
    }

    class TitleFetchRequest {
        +Bank M
        +AccountNo M
    }

    class RefNoRequest {
        +MakerID M
        +REFNO M
    }

    class RejectTxnRequest {
        +MakerID M
        +REFNO M
        +Remarks M
    }

    class CommonResponse {
        +ResponseCode
        +ResponseDescription
    }

    class DetailResponse {
        +ResponseCode
        +ResponseDescription
        +TransDetail[]
    }

    TokenRequest --> TokenResponse : Token
    CommonLookupRequest --> DetailResponse : GetPrintLocation
    CommonLookupRequest --> DetailResponse : GetDeliverTo
    CommonLookupRequest --> DetailResponse : GetProduct
    ProductScopedLookupRequest --> DetailResponse : GetAccount
    ProductScopedLookupRequest --> DetailResponse : GetInstrument
    ProductScopedLookupRequest --> DetailResponse : GetFileTemplate
    TitleFetchRequest --> CommonResponse : TitleFetch
    RefNoRequest --> CommonResponse : GetStatus
    RefNoRequest --> CommonResponse : ResendTxn
    RejectTxnRequest --> CommonResponse : RejectTxn
```

## TransSubmit Low-Level Payload

```mermaid
classDiagram
    class TransSubmitRequest {
        +ChannelID M
        +ProductCode M
        +DRAccountNo M
        +DRAccountTitle M
        +Stan M
        +DateTime M
        +MakerID M
        +CheckerID O
        +Signatory1ID O
        +Signatory2ID O
        +Signatory3ID O
        +ReleaserID O
        +FileTemplate M
        +BatchNo M
        +Transactions[] M
    }

    class TransactionDetail {
        +TXNREFNO M
        +XPIN O
        +BENEFNAME M
        +BENEMNAME M
        +BENELNAME O
        +BENEADDR M
        +BENECELL M
        +BENEEMAIL M
        +BENEIN M
        +BeneAccTitle M
        +BENEACNO M
        +SwiftBankCode O
        +BANK M
        +BRANCH M
        +INSTRUMENTNO O
        +INSTRUMENTPrintDT O
        +INSTRUMENTDT O
        +COVERAMOUNT M
        +CURRENCYCODE M
        +EXCHANGERATE M
        +TRANSACTIONAMOUNT M
        +ADVISING O
        +PRINTLOC O
        +REF1 O
        +REF2 O
        +Invoice[] M
    }

    class InvoiceDetail {
        +DOCNO M
        +DOCDT O
        +DOCDESCR M
        +DOCAMOUNT M
        +DEDUCTAMOUNT M
        +NETAMOUNT
        +IREF1 O
        +IREF2 O
        +COLUMNORDER
        +VALUE_DATE O
    }

    class TransSubmitResponse {
        +ResponseCode
        +ResponseDescription
    }

    TransSubmitRequest "1" --> "1..*" TransactionDetail : Transactions
    TransactionDetail "1" --> "0..*" InvoiceDetail : Invoice
    TransSubmitRequest --> TransSubmitResponse : POST /api/endpoint/transsubmit
```

## Response Detail Catalog

```mermaid
flowchart TD
    Detail["TransDetail response shapes"]

    Print["GetPrintLocation<br/>LOCATIONCODE<br/>LOCATIONNAME"]
    Deliver["GetDeliverTo<br/>PARENTID<br/>FIELDDESC"]
    Product["GetProduct<br/>PRODUCT_CODE<br/>PRODUCT_NAME"]
    Account["GetAccount<br/>ACCOUNTNO<br/>ACCOUNTNAME"]
    Instrument["GetInstrument<br/>INSTRUMENTNUMBER"]
    Template["GetFileTemplate<br/>CONF_DEF_ID<br/>CONF_DEF_DESC"]

    Simple["Simple response only<br/>ResponseCode + ResponseDescription"]
    Title["TitleFetch"]
    Submit["TransSubmit"]
    Status["GetStatus"]
    Resend["ResendTxn"]
    Reject["RejectTxn"]

    Detail --> Print
    Detail --> Deliver
    Detail --> Product
    Detail --> Account
    Detail --> Instrument
    Detail --> Template

    Simple --> Title
    Simple --> Submit
    Simple --> Status
    Simple --> Resend
    Simple --> Reject
```
