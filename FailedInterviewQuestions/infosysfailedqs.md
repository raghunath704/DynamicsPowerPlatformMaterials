# Q&A: Failed Interview Questions : INFOSYS

---

**Q1. Can a plugin be triggered from a ribbon button without saving a record?**

**A.** Yes. Wrap the plugin in a **Custom API**, which defines its own message. The plugin
executes on the API's *Main Operation* rather than on `Create`/`Update`, so no data is
written unless your code explicitly does so.

---

**Q2. Why can't I just register a plugin on the ribbon button directly?**

**A.** Plugins only run in response to Dataverse **messages**. A ribbon button is pure
client-side UI — it has no message. The old workaround was setting a dummy field and
saving the form, which is an anti-pattern (creates audit noise, triggers unrelated logic,
and forces a save the user may not want).

---

**Q3. What is a Custom API?**

**A.** A first-class, custom Dataverse message with a typed contract:
- **Request parameters** — the inputs
- **Response properties** — the outputs
- **Plugin type** — the code that implements the Main Operation

It appears in the Web API `$metadata` and is callable from JavaScript, C#, Power Automate,
or any OData client. A plug-in is not strictly required — a Custom API with no plug-in can
be used as a **business event**.

---

**Q4. Custom API vs. the older Custom Process Action?**

**A.** Microsoft's official terminology is **Custom Process Action** (commonly called
"Custom Action"). Custom API is the modern, developer-focused replacement.

| Capability | Custom Process Action | Custom API |
|---|---|---|
| Declarative logic with the workflow designer | Yes | No |
| Define main operation logic in code | Yes | Yes |
| Block extension by other publishers' plug-ins | Yes | Yes |
| Require a specific privilege to call | No | Yes |
| Make the message private | No | Yes |
| Localizable names and descriptions | No | Yes |
| Create an OData **Function** (HTTP GET) | No | Yes |
| Bind to a table (entity) | Yes | Yes |
| Bind to a table collection | No | Yes |
| Compose/edit by editing solution files (ALM) | No | Yes |
| Subject to the 2-minute execution limit | No | Yes |

**Note on extension blocking:** both can restrict it, by different mechanisms —
Custom Process Action uses `IsCustomProcessingStepAllowedForOtherPublishers`; Custom API
uses `AllowedCustomProcessingStepType`.

---

**Q5. What does the plugin code look like?**

**A.**

```csharp
public class DoSomethingPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider
            .GetService(typeof(IPluginExecutionContext));
        var factory = (IOrganizationServiceFactory)serviceProvider
            .GetService(typeof(IOrganizationServiceFactory));
        var service = factory.CreateOrganizationService(context.UserId);

        var recordId = (Guid)context.InputParameters["RecordId"];

        // ...business logic; no Create/Update required...

        context.OutputParameters["Message"] = "Completed successfully";
    }
}
```

Register the **assembly only** — do *not* register an SDK message processing step.
The Custom API binds the plugin type for you.

---

**Q6. Where do I create the Custom API in the maker portal?**

**A.** `make.powerapps.com` → **Solutions** → open an **unmanaged** solution →
**+ New** → **More** → **Other** → **Custom API**.

The same menu contains **Custom API Request Parameter** and
**Custom API Response Property**.

A friendlier alternative is the **Plug-in Registration Tool → Register → Register New
Custom API**, which creates the API, its parameters, and its properties on one screen.

---

**Q7. What values go on the Custom API record?**

| Field | Value |
|---|---|
| Unique Name | `new_DoSomething` — publisher prefix, no special characters |
| Name | `new_DoSomething` (primary name shown in solution lists) |
| Display Name | `Do Something` |
| Binding Type | `Global` (0), `Entity` (1), or `EntityCollection` (2) |
| Bound Entity Logical Name | e.g. `account` — only when binding isn't Global |
| Is Function | `No` → Action (POST); `Yes` → Function (GET, needs a response property) |
| Is Private | `No` — must be No to call from client script |
| Allowed Custom Processing Step Type | `None` (0), `Async Only` (1), or `Sync and Async` (2) |
| Execute Privilege Name | Optional privilege gate |
| Enabled for Workflow (`WorkflowSdkStepEnabled`) | `Yes` to call it as a workflow action |
| Plugin Type | Set after registering the assembly |

**Choosing the processing step type**

| Option | When to use |
|---|---|
| **None** | Your plug-in is the only logic; nobody may extend it |
| **Async Only** | Others may *detect* the event asynchronously but not cancel or alter it — recommended for business events |
| **Sync and Async** | Others may fully customize or cancel the operation. Also required for the configuration-data workaround in Q17 |

---

**Q8. How are request parameters and response properties named?**

**A.** **Unique Name** is what matters — it's the parameter name used when calling the API,
and it can't contain special characters.

**Name** is only the primary-name column shown in solution lists. Microsoft *recommends*
the convention `{CustomAPI UniqueName}.{Parameter UniqueName}` to disambiguate parameters
that share a Unique Name across different APIs — but it is **not enforced**.

| | Unique Name (used in code) | Name (recommended convention) |
|---|---|---|
| Request parameter | `RecordId` | `new_DoSomething.RecordId` |
| Response property | `Message` | `new_DoSomething.Message` |

Each record has a **Custom API** lookup that must point back to the parent API.

**Available types (both parameters and properties):** Boolean, DateTime, Decimal, Entity,
EntityCollection, EntityReference, Float, Integer, Money, Picklist, String, StringArray, Guid.

If the Custom API is **bound** to a table, the entity request parameter is generated for
you — don't create one manually.

---

**Q9. How do I call it from JavaScript?**

**A.**

```javascript
var Contoso = Contoso || {};

Contoso.doSomething = async function (primaryControl) {
    var formContext = primaryControl;
    var recordId = formContext.data.entity.getId().replace(/[{}]/g, "");

    var request = {
        RecordId: recordId,
        getMetadata: function () {
            return {
                boundParameter: null,        // null = unbound / Global
                operationType: 0,            // 0 = Action, 1 = Function, 2 = CRUD
                operationName: "new_DoSomething",
                parameterTypes: {
                    "RecordId": { typeName: "Edm.Guid", structuralProperty: 1 }
                }
            };
        }
    };

    Xrm.Utility.showProgressIndicator("Working...");
    try {
        var response = await Xrm.WebApi.online.execute(request);

        // A Custom API with no response properties returns 204 No Content,
        // and calling .json() on it will throw.
        if (response.status !== 204) {
            var result = await response.json();
            await Xrm.Navigation.openAlertDialog({ text: result.Message });
        }

        // formContext.data.refresh(false); // only if the plugin changed data
    } catch (error) {
        Xrm.Navigation.openErrorDialog({ message: error.message });
    } finally {
        Xrm.Utility.hideProgressIndicator();
    }
};
```

---

**Q10. What changes for a *bound* Custom API?**

**A.** Set `boundParameter: "entity"` and supply the entity on the request:

```javascript
var request = {
    entity: { entityType: "account", id: recordId },
    getMetadata: function () {
        return {
            boundParameter: "entity",
            operationType: 0,
            operationName: "new_DoSomething",
            parameterTypes: {
                "entity": {
                    typeName: "mscrm.account",
                    structuralProperty: 5   // 5 = EntityType
                }
            }
        };
    }
};
```

`structuralProperty` values: `1` = Primitive, `2` = Complex, `3` = Enum,
`4` = Collection, `5` = EntityType.

---

**Q11. How do I wire up the ribbon button?**

**A.** Two options:

1. **Modern command designer** — open the app in the app designer, select the table's
   command bar, **+ New command**, set **Action = Run JavaScript**, pick the web resource
   and function, then add parameters.
2. **Ribbon Workbench** (XrmToolBox) — add the button, create a JavaScript command,
   add CRM Parameters.

Parameters to pass (classic ribbon names; the modern designer surfaces equivalents such as
*Primary control* and *Selected rows*):

| Location | Parameter |
|---|---|
| Form button | `PrimaryControl` |
| Grid — multi-select | `SelectedControlSelectedItemIds` |
| Grid — single record | `FirstPrimaryItemId` |
| Grid — refresh after | `SelectedControl` |

Use **Enable rules** / **Visibility rules** so the button is hidden or greyed out when the
action isn't valid.

---

**Q12. Can the plugin see unsaved changes on the form?**

**A.** No. The plugin reads only committed Dataverse data. Options:
- Pass the current field values as request parameters, or
- `await formContext.data.save();` before calling the API (if saving is acceptable), or
- Use `formContext.getAttribute("x").getValue()` in JS and send it along.

---

**Q13. What are the execution limits?**

**A.**
- **2-minute timeout** — a plug-in implementing a Custom API main operation is subject to
  the 2-minute execution limit, and it runs synchronously inside the transaction.
- **Sandbox isolation** — always register with isolation mode **Sandbox** and location
  **Database**. No file system, no registry; network access is permitted over **HTTP and
  HTTPS (ports 80 and 443)** only.
- For slow external work, have the Custom API queue an asynchronous job instead of doing
  the work inline.

---

**Q14. How do I verify the Custom API exists before writing JS?**

**A.** Open in a browser:

```
https://<yourorg>.crm.dynamics.com/api/data/v9.2/$metadata
```

Search for your unique name. You should see something like:

```xml
<ComplexType Name="new_DoSomethingResponse">
  <Property Name="Message" Type="Edm.String" Unicode="false" />
</ComplexType>
<Action Name="new_DoSomething">
  <Parameter Name="RecordId" Type="Edm.Guid" Nullable="false" />
  <ReturnType Type="mscrm.new_DoSomethingResponse" Nullable="false" />
</Action>
```

If it's missing, check that **Is Private = No** and that the solution is published.

---

**Q15. Common pitfalls**

| Pitfall | Fix |
|---|---|
| Field can't be edited after save | **Immutable:** Unique Name, Binding Type, Bound Entity Logical Name, Is Function, Allowed Custom Processing Step Type, Enabled for Workflow. **Editable:** Name, Display Name, Description, Is Private, Execute Privilege Name, Plugin Type |
| `Is Private = Yes` | Not exposed in `$metadata`; also can't be used in a plug-in |
| Prefix mismatch with solution publisher | Use the publisher's customization prefix |
| Registered an SDK step for the plugin | Remove it — the Custom API binds the type |
| Chose `None` for step type, now need config data | Can't be changed — recreate with `Sync and Async` (see Q17) |
| `response.json()` throws | The API returned 204; it has no response properties |
| Button does nothing, no error | Check browser console; confirm the web resource is published |
| Stale form data after the call | `formContext.data.refresh(false)` |

---

**Q16. When should I use something other than a Custom API?**

**A.**
- **Power Automate (instant/HTTP flow)** — long-running work, connectors to external
  services, or logic best maintained by makers.
- **Custom Page / PCF control** — the button needs to collect user input first.
- **Azure Function via Custom API** — heavy compute that exceeds the 2-minute limit; the
  Custom API just enqueues the job.

---

**Q17. What is secure vs. unsecure configuration in a plugin?**

**A.** They are two optional string values you supply when you **register an SDK message
processing step** in the Plugin Registration Tool. They let you pass settings into a plugin
without hard-coding and recompiling the assembly.

| | Unsecure Configuration | Secure Configuration |
|---|---|---|
| Visibility | Readable by any user who can read the step record | Only readable by System Administrators |
| Where stored | `sdkmessageprocessingstep.configuration` | `sdkmessageprocessingstepsecureconfig` (separate table) |
| Travels in a solution export? | ✅ Yes | ❌ No — must be re-entered after import |
| Typical use | Endpoint URLs, table names, feature flags, timeouts | API keys, passwords, connection strings, tokens |

**How to consume it —** implement the parameterized constructor. Dataverse passes the
values in automatically:

```csharp
public class MyPlugin : IPlugin
{
    private readonly string _unsecureConfig;
    private readonly string _secureConfig;

    public MyPlugin(string unsecureConfig, string secureConfig)
    {
        _unsecureConfig = unsecureConfig;
        _secureConfig = secureConfig;
    }

    public void Execute(IServiceProvider serviceProvider)
    {
        var settings = JsonConvert.DeserializeObject<Settings>(_unsecureConfig);
        var apiKey = _secureConfig;
        // ...
    }
}
```

**Key points and caveats**

- The constructor signature must be `(string unsecure, string secure)` in that exact order.
  A parameterless constructor is also valid, but then you get no configuration.
- Configuration is **per-step**, not per-assembly. Registering the same plugin on two steps
  lets you give each step different settings.
- **A Custom API's main-operation plug-in gets neither value.** Microsoft states plainly:
  *"You can't pass secure or unsecure configuration to the main operation plug-in for the
  custom API."*
- **Documented workaround:** instead of setting `PluginTypeId` on the Custom API, register
  the plug-in as a **PostOperation** step on the custom API's message using the PRT — steps
  support configuration data normally. This requires **Allowed Custom Processing Step Type =
  Sync and Async**, which **cannot be changed after the API is created**, so decide up front.
- **Alternative:** use **Environment Variables** and read them via a query against
  `environmentvariablevalue` / `environmentvariabledefinition`.
- "Secure" means *hidden from non-admins* — it is not a substitute for Azure Key Vault for
  high-value secrets.
- Because secure config is lost on solution import, document the values or use environment
  variables so ALM doesn't silently break.
- Plugin instances are **cached and reused** across executions. Parse configuration in the
  constructor for performance, but never store per-request state in fields — that is not
  thread-safe.

---

**Q18. How do I export Dataverse record data directly to PDF?**

**A.** There are four practical approaches. Pick based on licensing, who owns the output,
and how much control you need over layout.

### Option 1 — Native "Export to PDF" (Dynamics 365 Sales, no code)

> ⚠️ This is a **Dynamics 365 Sales** feature, **not** a base Dataverse / model-driven app
> capability. It requires a Sales licence and the Sales Hub or Sales Professional app.

**Enable it:**

- *Sales Hub:* **Change area** (lower-left) → **App Settings** → **Overview** (or
  **Productivity tools**) → **Convert to PDF** → tick the entities → **Save**.
- *Sales Professional:* **Setup** → **Sales settings** → **Advanced settings** →
  **PDF generation**.

**Then:**

1. Ensure a Word template exists for the table — *no template, no PDF*. Out-of-the-box
   templates ship for Account, Opportunity, Order, Quote, and Invoice. Create others via
   table **⋯ → Word Templates → Create Word Template**, then map fields with the XML
   Mapping Pane and upload it back.
2. Open a record → command bar → **Export to PDF** (it may be hidden under **⋮ More commands**).
3. Choose a template, then **Download**, **Email**, **Save to SharePoint**, or
   **Save to Dynamics** (saves the PDF as a Note).

**Supported tables:** Account, Contact, Lead, Opportunity, Quote, Order, Invoice, plus
custom tables. **Custom tables are web-app only** — the phone app is limited to the seven
out-of-the-box entities.

**Constraints:** fonts are limited to those installed on the render server; performance is
best under ~2 MB; the generated PDF is not fully accessible without extra steps.
**Save to SharePoint** requires SharePoint integration + Document Management on the table;
**Save to Dynamics** requires Notes enabled on the table.

### Option 2 — Call the `ExportPdfDocument` message from a ribbon button

> ⚠️ `ExportPdfDocument` is an **internal, undocumented message** — it has no Microsoft
> Learn reference page. It works and is widely used in the community, but it is unsupported
> and could change without notice. Prefer Option 1 or Option 4 for production workloads.

It merges a document template with a record and returns the PDF as **base64**.

```javascript
async function exportRecordToPdf(entityTypeCode, templateId, recordId, fileName) {
    var url = Xrm.Utility.getGlobalContext().getClientUrl()
            + "/api/data/v9.2/ExportPdfDocument";

    var body = {
        EntityTypeCode: entityTypeCode,          // integer Object Type Code
        SelectedTemplate: {
            "@odata.type": "Microsoft.Dynamics.CRM.documenttemplate",
            documenttemplateid: templateId
        },
        SelectedRecords: JSON.stringify([recordId])   // stringified array
    };

    var response = await fetch(url, {
        method: "POST",
        headers: {
            "OData-MaxVersion": "4.0",
            "OData-Version": "4.0",
            "Accept": "application/json",
            "Content-Type": "application/json; charset=utf-8"
        },
        body: JSON.stringify(body)
    });

    if (!response.ok) { throw new Error(await response.text()); }

    var result = await response.json();
    var bytes  = Uint8Array.from(atob(result.PdfFile), c => c.charCodeAt(0));
    var blob   = new Blob([bytes], { type: "application/pdf" });

    var link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = fileName + ".pdf";
    link.click();
    URL.revokeObjectURL(link.href);
}
```

Notes:
- `EntityTypeCode` is the table's **Object Type Code** (integer), retrievable from
  `EntityDefinitions(LogicalName='account')?$select=ObjectTypeCode`.
- `SelectedRecords` must be a **string** containing a JSON array, not an array.
- `ExportWordDocument` is the sibling message that returns a .docx.
- Combine with **Q1–Q11**: put this call behind a ribbon button; nothing is saved to
  Dataverse unless you also create a Note/File afterwards.

### Option 3 — Generate the PDF inside a plugin / Custom API (server-side)

Call the message from C# via `OrganizationRequest`, then attach the bytes as an
**annotation** (Note) or a **File column**:

```csharp
var req = new OrganizationRequest("ExportPdfDocument")
{
    ["EntityTypeCode"]   = 1,                       // e.g. account
    ["SelectedTemplate"] = templateEntity,          // documenttemplate record
    ["SelectedRecords"]  = "[\"" + recordId + "\"]"
};

var resp = service.Execute(req);

// The message is undocumented — defend against either return shape.
var raw      = resp["PdfFile"];
var pdfBytes = raw as byte[] ?? Convert.FromBase64String((string)raw);

service.Create(new Entity("annotation")
{
    ["subject"]      = "Generated PDF",
    ["filename"]     = "record.pdf",
    ["mimetype"]     = "application/pdf",
    ["documentbody"] = Convert.ToBase64String(pdfBytes),
    ["objectid"]     = new EntityReference("account", recordId)
    // Do NOT also set "objecttypecode" — objectid already carries the type.
});
```

Watch the plugin **2-minute timeout** and the sandbox's assembly restrictions — merge
dependencies with ILMerge, deploy a plug-in package, or offload to Azure.

### Option 4 — Power Automate

`Populate a Microsoft Word template` (Word Online Business) →
`Convert file` (OneDrive/SharePoint, output PDF) → email it, or write it back to Dataverse
as a Note or File column. Best when non-developers must maintain the layout, when no Sales
licence is available, or when the document is emailed rather than downloaded.

### Comparison

| Approach | Licence | Code | Layout control | Best for |
|---|---|---|---|---|
| Native Export to PDF | **D365 Sales** | None | Word template | Standard user-driven exports |
| `ExportPdfDocument` from JS | Any | Low | Word template | Custom ribbon button, browser download — *unsupported API* |
| Plugin / Custom API | Any | Medium | Word template or library | Server-side, attach to record, automation |
| Power Automate | Any | Low-code | Word template | Email delivery, maker-owned templates |
| Third-party (QuestPDF, iText, DocumentsCorePack) | Any | High | Pixel-level | Complex invoices, barcodes, precise branding |

### Common pitfalls

- **"Export to PDF" not visible** — no Sales licence, the feature isn't enabled for that
  table under **Convert to PDF**, or no Word template exists for the table.
- **Empty merge fields** — the Word template's XML mapping is stale; re-download and remap
  after adding columns.
- **Wrong fonts in output** — only fonts installed on the render server are honoured.
- **Template GUIDs differ per environment** — store the ID in an **Environment Variable**
  rather than hard-coding it.
- **Large PDFs** — base64 inflates size ~33%; performance degrades above ~2 MB, and
  attachments are capped by the environment's maximum file size setting.
- **Third-party PDF libraries in a sandboxed plugin** must be merged into a single assembly
  or deployed as a Dataverse plug-in package.
