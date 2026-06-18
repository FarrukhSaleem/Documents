# proc-bank - Low-Level Diagram

This document captures the low-level design of the `proc-bank` Azure Functions
app. It focuses on runtime composition, object handoffs, route resolution,
downstream dispatch, discovery, and error paths.

## Scope

- Application: `Bank/` .NET 8 Azure Functions isolated worker app.
- Primary endpoint: `POST /api/banking/execute`.
- Discovery endpoint: `GET /api/banking/banks`.
- Source of truth for routing: `AppVariables:BankRouting` in
  `appsettings.{environment}.json`.
- Persistence: none. This service does not own a database.
- Bank-specific business logic: delegated to `sys-thirdparty-mgmt`.

## 1. Runtime Composition

```mermaid
flowchart LR
    Client["Mobile / Web / API Client"]

    subgraph Host["Azure Functions Host - .NET 8 isolated worker"]
        Program["Program.cs<br/>Create FunctionsApplication builder<br/>Load appsettings.json<br/>Load appsettings.{ENVIRONMENT}.json<br/>Load environment variables"]
        Middleware["Middleware pipeline<br/>ExecutionTimingMiddleware<br/>ExceptionHandlerMiddleware"]
        DI["DI container<br/>Singleton: BankRouteResolver<br/>Scoped: BankRouterService<br/>Scoped: IBankProxyMgmt -> BankProxyMgmtSys<br/>Scoped: CustomHTTPClient<br/>Singleton: Polly HTTP policy"]
        Functions["HTTP functions<br/>BankRouterFunction<br/>BanksInfoFunction"]
    end

    subgraph Config["Configuration"]
        BaseConfig["appsettings.json<br/>base defaults"]
        EnvConfig["appsettings.dev/qa/stg/prod/dr.json<br/>BankRouting map<br/>HttpClientPolicy"]
        EnvVars["ENVIRONMENT / ASPNETCORE_ENVIRONMENT<br/>other environment variables"]
    end

    subgraph Core["Shared AGI Core packages"]
        CoreDtos["BaseResponseDTO<br/>HttpResponseSnapshot"]
        CoreHttp["CustomHTTPClient<br/>SendProcessHTTPReqAsync"]
        CoreExceptions["NoRequestBodyException<br/>RequestValidationFailureException<br/>PassThroughHttpException"]
    end

    SystemLayer["sys-thirdparty-mgmt<br/>system-layer bank endpoints"]
    Banks["External bank APIs<br/>DIB today<br/>ENBD / FAB / ADCB future"]
    AppInsights["Application Insights / console logs"]

    Client --> Functions
    Program --> Middleware
    Program --> DI
    BaseConfig --> Program
    EnvConfig --> Program
    EnvVars --> Program
    DI --> Functions
    Functions --> CoreDtos
    Functions --> CoreExceptions
    Functions --> CoreHttp
    CoreHttp --> SystemLayer
    SystemLayer --> Banks
    Middleware --> AppInsights
    Functions --> AppInsights
```

## 2. POST `/api/banking/execute` Low-Level Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client
    participant Host as Azure Functions Host
    participant Timing as ExecutionTimingMiddleware
    participant Handler as ExceptionHandlerMiddleware
    participant Fn as BankRouterFunction.Run
    participant Req as BankProxyReqDTO
    participant Resolver as BankRouteResolver
    participant Service as BankRouterService
    participant Proxy as BankProxyMgmtSys
    participant Http as CustomHTTPClient
    participant Sys as sys-thirdparty-mgmt

    Client->>Host: POST /api/banking/execute
    Host->>Timing: start request timer
    Host->>Fn: invoke HTTP trigger
    Fn->>Fn: ReadBodyAsync for BankProxyReqDTO

    alt Request body is null or invalid
        Fn->>Handler: throw NoRequestBodyException
        Handler-->>Client: 400 BaseResponseDTO
    else Body parsed
        Fn->>Req: Validate()

        alt BankId, Operation, or Payload invalid
            Req->>Handler: throw RequestValidationFailureException
            Handler-->>Client: 400 BaseResponseDTO
        else Request DTO valid
            Fn->>Fn: normalize BankId = trim + upper<br/>normalize Operation = trim + lower
            Fn->>Resolver: Resolve(bankId, operation)

            alt BankRouting missing
                Resolver->>Handler: throw RequestValidationFailureException<br/>CB_ROUTE_0003
                Handler-->>Client: 400 BaseResponseDTO
            else BankId not configured
                Resolver->>Handler: throw RequestValidationFailureException<br/>CB_ROUTE_0001
                Handler-->>Client: 400 BaseResponseDTO
            else Operation not configured for bank
                Resolver->>Handler: throw RequestValidationFailureException<br/>CB_ROUTE_0002
                Handler-->>Client: 400 BaseResponseDTO
            else Target URL found
                Resolver-->>Fn: targetUrl
                Fn->>Fn: create BankRouteRequest<br/>BankId, Operation, TargetUrl, Payload
                Fn->>Service: Route(routeRequest)
                Service->>Proxy: Dispatch(routeRequest)
                Proxy->>Proxy: Payload.GetRawText()<br/>JsonSerializer.Deserialize to object
                Proxy->>Http: SendProcessHTTPReqAsync(POST, targetUrl, body)
                Http->>Sys: POST configured system-layer URL
                Sys-->>Http: HttpResponseMessage
                Http-->>Proxy: HttpResponseMessage
                Proxy-->>Service: HttpResponseSnapshot
                Service-->>Fn: HttpResponseSnapshot

                alt Downstream returned non-2xx
                    Fn->>Handler: throw PassThroughHttpException<br/>mirror downstream status
                    Handler-->>Client: BaseResponseDTO with downstream status
                else Downstream returned 2xx
                    Fn->>Fn: response.ExtractData()
                    Fn->>Fn: ParseJsonOrEmpty(content)
                    Fn-->>Client: 200 BaseResponseDTO<br/>data = BankProxyResDTO
                end
            end
        end
    end

    Host->>Timing: log duration
```

## 3. Execute Endpoint Object Handoff

```mermaid
flowchart TD
    HttpRequest["HttpRequest<br/>route: banking/execute<br/>method: POST"]

    RawBody["Raw JSON body<br/>{ bankId, operation, payload }"]

    ReqDto["BankProxyReqDTO<br/>BankId: string<br/>Operation: string<br/>Payload: JsonElement"]

    Validation["Validate()<br/>BankId required<br/>Operation required<br/>Payload required<br/>Payload must be JSON object"]

    Normalized["Normalized route keys<br/>BankId = trim + ToUpperInvariant<br/>Operation = trim + ToLowerInvariant"]

    ResolvedUrl["BankRouteResolver.Resolve()<br/>lookup AppVariables:BankRouting<br/>return target URL"]

    RouteRequest["BankRouteRequest<br/>BankId<br/>Operation<br/>TargetUrl<br/>Payload"]

    Dispatch["IBankProxyMgmt.Dispatch()<br/>implemented by BankProxyMgmtSys"]

    BodyTransform["Outbound body transform<br/>Payload.GetRawText()<br/>JsonSerializer.Deserialize to object"]

    Outbound["CustomHTTPClient.SendProcessHTTPReqAsync<br/>method: POST<br/>url: TargetUrl<br/>body: operation payload"]

    Snapshot["HttpResponseSnapshot<br/>StatusCode<br/>IsSuccessStatusCode<br/>ExtractData()<br/>ExtractBaseResponse()"]

    SuccessPayload["BankProxyResDTO<br/>BankId<br/>Operation<br/>Data: JsonElement"]

    Envelope["BaseResponseDTO<br/>message<br/>errorCode<br/>data"]

    HttpRequest --> RawBody --> ReqDto --> Validation --> Normalized --> ResolvedUrl --> RouteRequest
    RouteRequest --> Dispatch --> BodyTransform --> Outbound --> Snapshot
    Snapshot --> SuccessPayload --> Envelope
```

## 4. Route Resolution Decision Tree

```mermaid
flowchart TD
    Start["Resolve(bankId, operation)"]
    HasRouting{"AppVariables:BankRouting exists<br/>and has at least one bank?"}
    HasBank{"BankRouting contains bankId?"}
    HasOperation{"Bank operations contains operation<br/>and URL is not blank?"}
    ReturnUrl["Return targetUrl"]

    MissingConfig["Throw RequestValidationFailureException<br/>CB_ROUTE_0003<br/>BANK_ROUTING_NOT_CONFIGURED"]
    UnknownBank["Throw RequestValidationFailureException<br/>CB_ROUTE_0001<br/>UNSUPPORTED_BANK<br/>message includes configured banks"]
    UnknownOperation["Throw RequestValidationFailureException<br/>CB_ROUTE_0002<br/>UNSUPPORTED_OPERATION<br/>message includes supported operations"]

    Start --> HasRouting
    HasRouting -- "No" --> MissingConfig
    HasRouting -- "Yes" --> HasBank
    HasBank -- "No" --> UnknownBank
    HasBank -- "Yes" --> HasOperation
    HasOperation -- "No" --> UnknownOperation
    HasOperation -- "Yes" --> ReturnUrl
```

## 5. GET `/api/banking/banks` Discovery Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client
    participant Host as Azure Functions Host
    participant Fn as BanksInfoFunction.Run
    participant Resolver as BankRouteResolver
    participant Config as AppVariables BankRouting

    Client->>Host: GET /api/banking/banks
    Host->>Fn: invoke HTTP trigger
    Fn->>Resolver: GetCatalog()
    Resolver->>Config: read configured banks and operations
    Resolver-->>Fn: defensive copy of catalog
    Fn->>Fn: build BanksCatalogResDTO
    Fn-->>Client: 200 BaseResponseDTO<br/>data = bank count + operations + target URLs
```

## 6. Config-Driven Routing Map

```mermaid
flowchart LR
    AppVars["AppVariables"]
    BankRouting["BankRouting<br/>Dictionary of BankRoutingConfig<br/>case-insensitive keys"]
    DIB["DIB"]
    DIBOps["Operations<br/>Dictionary of operation URL mappings<br/>case-insensitive keys"]

    GetToken["get-token<br/>https://localhost:7206/api/token/getToken"]
    GetPrintLocation["get-print-location<br/>https://localhost:7206/api/endpoint/getprintlocation"]
    GetDeliverTo["get-deliver-to<br/>http://localhost:5282/api/endpoint/getdeliverto"]
    GetProduct["get-product<br/>http://localhost:5282/api/endpoint/getproduct"]
    GetAccount["get-account<br/>http://localhost:5282/api/endpoint/getaccount"]
    GetInstrument["get-instrument<br/>http://localhost:5282/api/endpoint/getinstrument"]
    GetFileTemplate["get-file-template<br/>http://localhost:5282/api/endpoint/getfiletemplate"]
    TitleFetch["title-fetch<br/>http://localhost:5282/api/endpoint/titlefetch"]
    TransSubmit["trans-submit<br/>http://localhost:5282/api/endpoint/transsubmit"]
    GetStatus["get-status<br/>http://localhost:5282/api/endpoint/getstatus"]
    ResendTxn["resend-txn<br/>http://localhost:5282/api/endpoint/resendtxn"]
    RejectTxn["reject-txn<br/>http://localhost:5282/api/endpoint/rejecttxn"]

    FutureBanks["Future banks<br/>ENBD / FAB / ADCB<br/>same Operations shape<br/>config-only onboarding"]

    AppVars --> BankRouting
    BankRouting --> DIB
    BankRouting -.-> FutureBanks
    DIB --> DIBOps
    DIBOps --> GetToken
    DIBOps --> GetPrintLocation
    DIBOps --> GetDeliverTo
    DIBOps --> GetProduct
    DIBOps --> GetAccount
    DIBOps --> GetInstrument
    DIBOps --> GetFileTemplate
    DIBOps --> TitleFetch
    DIBOps --> TransSubmit
    DIBOps --> GetStatus
    DIBOps --> ResendTxn
    DIBOps --> RejectTxn
```

## 7. Main Code-Level Dependencies

```mermaid
classDiagram
    class BankRouterFunction {
        -ILogger logger
        -BankRouterService bankRouterService
        -BankRouteResolver routeResolver
        +Run(HttpRequest req)
        -ParseJsonOrEmpty(string content)
    }

    class BanksInfoFunction {
        -ILogger logger
        -BankRouteResolver routeResolver
        +Run(HttpRequest req)
    }

    class BankProxyReqDTO {
        +string BankId
        +string Operation
        +JsonElement Payload
        +Validate()
    }

    class BankRouteResolver {
        -AppConfigs config
        -ILogger logger
        +Resolve(string bankId, string operation) string
        +GetCatalog() IReadOnlyDictionary
        -LogLoadedCatalog()
    }

    class BankRouterService {
        -ILogger logger
        -IBankProxyMgmt bankProxyMgmt
        +Route(BankRouteRequest request)
    }

    class IBankProxyMgmt {
        <<interface>>
        +Dispatch(BankRouteRequest request)
    }

    class BankProxyMgmtSys {
        -CustomHTTPClient customHttpClient
        -ILogger logger
        +Dispatch(BankRouteRequest request)
    }

    class BankRouteRequest {
        +string BankId
        +string Operation
        +string TargetUrl
        +JsonElement Payload
    }

    class BankProxyResDTO {
        +string BankId
        +string Operation
        +JsonElement Data
    }

    class BanksCatalogResDTO {
        +int BankCount
        +List Banks
    }

    class AppConfigs {
        +string ASPNETCORE_ENVIRONMENT
        +Dictionary BankRouting
    }

    class BankRoutingConfig {
        +Dictionary Operations
    }

    BankRouterFunction --> BankProxyReqDTO : reads and validates
    BankRouterFunction --> BankRouteResolver : resolves URL
    BankRouterFunction --> BankRouteRequest : creates
    BankRouterFunction --> BankRouterService : calls
    BankRouterFunction --> BankProxyResDTO : creates success payload
    BankRouterService --> IBankProxyMgmt : delegates
    IBankProxyMgmt <|.. BankProxyMgmtSys : implements
    BankProxyMgmtSys --> BankRouteRequest : consumes
    BankProxyMgmtSys --> CustomHTTPClient : sends HTTP POST
    BankRouteResolver --> AppConfigs : reads options
    AppConfigs --> BankRoutingConfig : contains
    BanksInfoFunction --> BankRouteResolver : gets catalog
    BanksInfoFunction --> BanksCatalogResDTO : creates
```

## 8. Error and Response Paths

```mermaid
flowchart TD
    Entry["Incoming request"]
    NullBody["Body missing or unreadable"]
    InvalidDto["DTO validation failed"]
    MissingRouting["BankRouting missing"]
    UnsupportedBank["BankId not configured"]
    UnsupportedOperation["Operation not configured"]
    DownstreamFailure["Downstream non-2xx"]
    Success["Downstream 2xx"]

    NoBodyException["NoRequestBodyException<br/>HTTP 400"]
    ValidationException["RequestValidationFailureException<br/>HTTP 400"]
    ConfigException["RequestValidationFailureException<br/>CB_ROUTE_0003<br/>HTTP 400"]
    BankException["RequestValidationFailureException<br/>CB_ROUTE_0001<br/>HTTP 400"]
    OperationException["RequestValidationFailureException<br/>CB_ROUTE_0002<br/>HTTP 400"]
    PassThrough["PassThroughHttpException<br/>HTTP status mirrors downstream"]
    SuccessEnvelope["BaseResponseDTO<br/>message: Routed successfully<br/>errorCode: empty<br/>data: BankProxyResDTO"]

    Entry --> NullBody --> NoBodyException
    Entry --> InvalidDto --> ValidationException
    Entry --> MissingRouting --> ConfigException
    Entry --> UnsupportedBank --> BankException
    Entry --> UnsupportedOperation --> OperationException
    Entry --> DownstreamFailure --> PassThrough
    Entry --> Success --> SuccessEnvelope
```

## 9. Low-Level Design Notes

- `BankRouterFunction` owns request parsing, validation invocation,
  normalization, response shaping, and downstream failure pass-through.
- `BankProxyReqDTO.Validate()` only validates the generic routing envelope.
  Operation-specific payload validation is intentionally owned by the system
  layer.
- `BankRouteResolver` owns all route lookup behavior and startup catalog
  logging. It does not know about specific banks at compile time.
- `BankRouterService` is a thin orchestration point for future process-layer
  logic such as metrics, branching, or multi-bank workflows.
- `BankProxyMgmtSys` is the HTTP adapter to `sys-thirdparty-mgmt`; it converts
  the `JsonElement` payload back to an object and posts it to the resolved URL.
- `CustomHTTPClient` and the registered Polly policy own outbound HTTP retries,
  timeout behavior, correlation propagation, and standard process-layer logging.
- Adding a bank or operation normally changes only `appsettings.{env}.json`.
  Constants and enums are convenience symbols, not resolver requirements.
