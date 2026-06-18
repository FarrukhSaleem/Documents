# CMS Wrapper API - Updated Low Level Diagram

Source: `CMSWrapperAPI.pdf`

Scope: This document diagrams only the APIs explicitly listed in the supplied API document and the authentication path required to call those APIs. It does not add Chargemod, Emirates, vehicle recovery, station, database, queue, or any other non-CMS Wrapper API flow.

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

- The Web Service URL table lists `TitleFetch` as `/api/endpoint/titlefetch`, while section `4.4.8` shows `/api/endpoint/getfiletemplate`. This diagram uses `/api/endpoint/titlefetch` from the API list.
- The Web Service URL table lists `ResendTxn` as `/api/endpoint/resendtxn`, while section `4.4.11` shows `/api/endpoint/getstatus`. This diagram uses `/api/endpoint/resendtxn` from the API list.
- Section `4.4.12` labels `RejectTxn` purpose as "Resend Transaction", but the API name, endpoint, request, and response describe rejection. This diagram treats it as reject transaction.

## Updated Authentication Boundary

The CMS Wrapper `Token` API still receives `ID` and `Password` as required by the API document. The updated implementation resolves those credentials from Key Vault before invoking `POST /api/token/getToken`.

```mermaid
flowchart LR
    Client["Client application"]

    subgraph App["ThirdPartyMgmtSystem - CMS Wrapper APIs only"]
        DibAuth["DIB authentication middleware<br/>applies only to documented CMS Wrapper functions"]
        Cache["Redis cache<br/>DIB_ACCESS_TOKEN"]
        CredentialProvider["DibTokenCredentialProvider"]
        SecretReader["KeyVaultSecretReader"]
        RequestContext["RequestContext<br/>DIB bearer token slot"]
        FunctionApi["Documented CMS Wrapper function<br/>GetPrintLocation ... RejectTxn"]
        AtomicHandler["CMS Wrapper atomic handler<br/>calls one documented endpoint"]
    end

    KeyVault["Azure Key Vault<br/>AppVariables--BankApi--ID<br/>AppVariables--BankApi--Password"]

    subgraph Cms["CMS Wrapper downstream API document"]
        Token["Token<br/>POST /api/token/getToken<br/>ID, Password -> accessToken"]
        LookupApis["Lookup APIs<br/>GetPrintLocation, GetDeliverTo, GetProduct,<br/>GetAccount, GetInstrument, GetFileTemplate, TitleFetch"]
        TransactionApis["Transaction APIs<br/>TransSubmit, GetStatus, ResendTxn, RejectTxn"]
    end

    Vendor["TransPaymentAPI / Core-DMZ"]

    Client -->|"POST JSON request"| DibAuth
    DibAuth -->|"read cached token"| Cache
    DibAuth -->|"cache miss"| CredentialProvider
    CredentialProvider --> SecretReader
    SecretReader -->|"read ID and Password secrets"| KeyVault
    CredentialProvider -->|"ID, Password"| Token
    Token -->|"accessToken"| DibAuth
    DibAuth -->|"cache token"| Cache
    DibAuth -->|"set bearer token"| RequestContext
    RequestContext --> FunctionApi
    FunctionApi --> AtomicHandler
    AtomicHandler -->|"Bearer token + request payload"| LookupApis
    AtomicHandler -->|"Bearer token + request payload"| TransactionApis
    LookupApis --> Vendor
    TransactionApis --> Vendor
```

## Token Acquisition Sequence

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant M as DIB Auth Middleware
    participant Cache as Redis DIB_ACCESS_TOKEN
    participant P as DibTokenCredentialProvider
    participant KV as Azure Key Vault
    participant T as Token API
    participant RC as RequestContext
    participant F as Documented CMS Wrapper Function

    C->>M: POST documented CMS Wrapper API request
    M->>Cache: Read DIB_ACCESS_TOKEN
    alt Cached access token exists
        Cache-->>M: accessToken
    else Cache miss
        M->>P: GetCredentialsAsync()
        P->>KV: Get secret AppVariables--BankApi--ID
        KV-->>P: ID
        P->>KV: Get secret AppVariables--BankApi--Password
        KV-->>P: Password
        P-->>M: GenerateDibTokenHandlerReqDTO
        M->>T: POST /api/token/getToken {ID, Password}
        T-->>M: {message, accessToken}
        M->>Cache: Store DIB_ACCESS_TOKEN
    end
    M->>RC: SetDibToken(accessToken)
    M->>F: Continue function execution
```

## Common API Runtime Sequence

This sequence applies to the eleven documented business APIs after the bearer token is available.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant M as DIB Auth Middleware
    participant F as Documented API Function
    participant H as API Handler
    participant A as CMS Wrapper Atomic Handler
    participant RC as RequestContext
    participant CMS as CMS Wrapper Endpoint
    participant V as TransPaymentAPI / Core-DMZ

    C->>M: POST one documented API request
    M->>M: Resolve DIB token from cache or Key Vault + Token API
    M->>F: Execute function with token in RequestContext
    F->>F: Deserialize request body
    F->>F: Validate documented mandatory fields
    F->>H: Handle request
    H->>A: Build downstream request DTO
    A->>RC: GetDibToken()
    A->>CMS: POST documented endpoint with Bearer token
    CMS->>V: Forward request to vendor/core system
    V-->>CMS: Vendor response
    CMS-->>A: ResponseCode, ResponseDescription, optional TransDetail
    A-->>H: HttpResponseSnapshot
    H->>H: Map downstream response
    H-->>F: BaseResponseDTO
    F-->>C: API response
```

## Documented API Fan-Out

```mermaid
flowchart TD
    TokenReady["DIB accessToken ready<br/>from Redis or Token API"]

    subgraph Lookup["Lookup APIs from document"]
        PrintLocation["GetPrintLocation<br/>POST /api/endpoint/getprintlocation"]
        DeliverTo["GetDeliverTo<br/>POST /api/endpoint/getdeliverto"]
        Product["GetProduct<br/>POST /api/endpoint/getproduct"]
        Account["GetAccount<br/>POST /api/endpoint/getaccount"]
        Instrument["GetInstrument<br/>POST /api/endpoint/getinstrument"]
        FileTemplate["GetFileTemplate<br/>POST /api/endpoint/getfiletemplate"]
        TitleFetch["TitleFetch<br/>POST /api/endpoint/titlefetch"]
    end

    subgraph Transaction["Transaction APIs from document"]
        TransSubmit["TransSubmit<br/>POST /api/endpoint/transsubmit"]
        GetStatus["GetStatus<br/>POST /api/endpoint/getstatus"]
        ResendTxn["ResendTxn<br/>POST /api/endpoint/resendtxn"]
        RejectTxn["RejectTxn<br/>POST /api/endpoint/rejecttxn"]
    end

    TokenReady --> PrintLocation
    TokenReady --> DeliverTo
    TokenReady --> Product
    TokenReady --> Account
    TokenReady --> Instrument
    TokenReady --> FileTemplate
    TokenReady --> TitleFetch
    TokenReady --> TransSubmit
    TokenReady --> GetStatus
    TokenReady --> ResendTxn
    TokenReady --> RejectTxn
```

## Transaction Lifecycle Using Documented APIs Only

```mermaid
flowchart TD
    Start(["Start"])
    Credentials["Resolve Token credentials<br/>ID and Password from Key Vault"]
    TokenCall["Token<br/>POST /api/token/getToken<br/>Get JWT accessToken"]
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

    Start --> Credentials
    Credentials --> TokenCall
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
    class KeyVaultCredentialSecrets {
        +ID secret name
        +Password secret name
    }

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

    KeyVaultCredentialSecrets --> TokenRequest : populate
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
