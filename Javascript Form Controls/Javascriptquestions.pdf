
# 30 Essential JavaScript Interview Questions for Dynamics 365 CRM and Power Platform

These questions cover JavaScript fundamentals, the Dataverse client API, model-driven forms, Web API, command bars, performance, security, and real-world scenarios.

---

## 1. Why is JavaScript used in Dynamics 365 and Power Platform?

JavaScript provides client-side behavior in model-driven apps, including:

- Form validation
- Setting field values
- Showing or hiding controls
- Making fields required or read-only
- Responding to OnLoad, OnChange, and OnSave events
- Calling the Dataverse Web API
- Opening forms, dialogs, and web resources
- Implementing command-bar actions

JavaScript should enhance the user experience, not enforce critical business rules by itself. Server-side validation should use plug-ins, flows, or Dataverse business logic because client-side JavaScript can be bypassed.

---

## 2. What is `formContext`, and why should you avoid `Xrm.Page`?

`formContext` represents the form or form item on which JavaScript is executing.

It is normally obtained from the execution context:

`const formContext = executionContext.getFormContext();`

`Xrm.Page` is deprecated because it is a global object and does not reliably identify the current form context, particularly in grids and reusable event handlers.

A recommended handler looks like:

`function onLoad(executionContext) { const formContext = executionContext.getFormContext(); }`

In the event configuration, **Pass execution context as first parameter** must be enabled.

---

## 3. What is the difference between an attribute and a control?

An **attribute** contains the data value and related metadata.

`formContext.getAttribute("telephone1")`

A **control** is the UI component that displays the attribute.

`formContext.getControl("telephone1")`

Examples:

- Use the attribute to call `getValue()`, `setValue()`, or `setRequiredLevel()`.
- Use the control to call `setVisible()`, `setDisabled()`, or `setNotification()`.

A field may have one attribute but multiple controls, such as a field appearing in both the body and header.

---

## 4. How do you get and set field values?

Use `getAttribute()` followed by `getValue()` or `setValue()`:

`const name = formContext.getAttribute("name").getValue();`

`formContext.getAttribute("name").setValue("Contoso");`

Always check that the attribute exists:

`const attribute = formContext.getAttribute("new_code"); if (attribute) attribute.setValue("A100");`

Calling `setValue()` does not automatically execute the field’s OnChange handlers. If required, call:

`attribute.fireOnChange();`

---

## 5. How are common Dataverse field types handled in JavaScript?

| Dataverse type | JavaScript value |
|---|---|
| Single line of text | String or `null` |
| Whole/decimal/currency number | Number or `null` |
| Yes/No | Boolean or `null` |
| Choice | Numeric option value or `null` |
| Choices/multi-select | Array of numeric option values |
| Date and time | JavaScript `Date` or `null` |
| Lookup | Array of lookup objects or `null` |

A lookup value has this general structure:

`[{ id: "{GUID}", name: "Contoso", entityType: "account" }]`

Use option values—not labels—when comparing Choice columns.

---

## 6. How do you work with lookup fields?

Read a lookup:

`const value = formContext.getAttribute("primarycontactid").getValue();`

Because it is an array, access the first item only after checking it:

`if (value?.length) { const id = value[0].id.replace(/[{}]/g, ""); const name = value[0].name; const entityType = value[0].entityType; }`

Set a lookup:

`formContext.getAttribute("primarycontactid").setValue([{ id: contactId, name: contactName, entityType: "contact" }]);`

The ID must be a valid GUID, and `entityType` must be the table’s logical name.

---

## 7. How do you work with Choice and multi-select Choice columns?

For a single Choice column:

`const statusReason = formContext.getAttribute("statuscode").getValue();`

This returns the numeric option value, not the display label.

To retrieve selected text:

`const text = formContext.getAttribute("statuscode").getText();`

For a multi-select Choice column:

`const selectedValues = formContext.getAttribute("new_skills").getValue();`

Set multiple values with:

`formContext.getAttribute("new_skills").setValue([100000000, 100000002]);`

Avoid using labels in business logic because labels can change or be translated.

---

## 8. What is the difference between `getValue()`, `getText()`, and `getSelectedOption()`?

- `getValue()` returns the stored value.
- `getText()` returns the selected label for a Choice column.
- `getSelectedOption()` returns an object containing both `text` and `value`.

Example:

`const selected = formContext.getAttribute("industrycode").getSelectedOption();`

The result is similar to:

`{ text: "Consulting", value: 100000001 }`

Business logic should normally compare `value`, while `text` is useful for display.

---

## 9. How do you show, hide, enable, disable, and require a field?

Use the control for visibility and enabled state:

- `control.setVisible(false)`
- `control.setDisabled(true)`

Use the attribute for required level:

- `attribute.setRequiredLevel("required")`
- `attribute.setRequiredLevel("recommended")`
- `attribute.setRequiredLevel("none")`

Be careful when hiding or disabling a business-required field. The user may be unable to provide a value, and save behavior can become confusing. Critical requirements should also be validated server-side.

---

## 10. What are the main model-driven form events?

The most common events are:

- **OnLoad:** Runs when the form loads.
- **OnChange:** Runs after a field value changes and focus leaves the field, or when explicitly fired.
- **OnSave:** Runs when the record is saved.
- **PostSave:** Runs after the save operation completes.
- **TabStateChange:** Runs when a tab is expanded or collapsed.
- **OnRecordSelect:** Commonly used with editable grids.

Handlers should be small, reusable, null-safe, and preferably placed inside a namespace rather than being global standalone functions.

---

## 11. How do you prevent a form from saving?

Inside an OnSave handler:

`const eventArgs = executionContext.getEventArgs(); eventArgs.preventDefault();`

Use this when validation fails. Display a clear notification explaining why:

`formContext.ui.setFormNotification("Complete the approval details before saving.", "ERROR", "approval_validation");`

For asynchronous OnSave logic, use the platform-supported async OnSave model and return the promise from the handler. Do not assume that simply declaring a function `async` will automatically pause the save in every configuration.

---

## 12. How do you determine the save mode?

Use:

`const saveMode = executionContext.getEventArgs().getSaveMode();`

The numeric value identifies operations such as Save, Save and Close, Auto Save, or Deactivate.

This is useful when behavior should differ by operation. However, avoid hard-coding undocumented assumptions. Use the values documented for the client API and test the behavior in the target application.

---

## 13. Does `setValue()` trigger the OnChange event?

No. `setValue()` changes the attribute value but does not automatically run its registered OnChange handlers.

If the handlers must run, call:

`attribute.fireOnChange();`

Be careful to avoid recursion. If an OnChange handler sets the same field and then calls `fireOnChange()`, it can create an infinite loop.

---

## 14. What is `addOnChange()`, and when would you use it?

`addOnChange()` dynamically registers a handler:

`formContext.getAttribute("new_category").addOnChange(MyApp.Form.categoryChanged);`

It is useful when handlers must be registered conditionally or when centralized initialization is preferred.

To prevent duplicate registrations:

- Register handlers once during form initialization.
- Use a module-level initialization flag if necessary.
- Remove handlers with `removeOnChange()` when appropriate.

Repeated registration can cause the same code to run multiple times.

---

## 15. How do you identify whether a form is Create, Update, Read Only, or Disabled?

Use:

`const formType = formContext.ui.getFormType();`

Common values include:

- `1`: Create
- `2`: Update
- `3`: Read Only
- `4`: Disabled
- `6`: Bulk Edit

Example:

`if (formContext.ui.getFormType() === 1) { /* create-form logic */ }`

Do not rely on record ID alone because form type communicates the actual form state more clearly.

---

## 16. How do you obtain the current record ID and table name?

Record ID:

`const id = formContext.data.entity.getId().replace(/[{}]/g, "");`

Table logical name:

`const entityName = formContext.data.entity.getEntityName();`

On a Create form, the ID may be unavailable until the record is saved. Code must account for this before making Web API calls that require an existing record.

---

## 17. How do you save or refresh a form programmatically?

Save:

`await formContext.data.save();`

Refresh data:

`await formContext.data.refresh(false);`

Passing `true` to `refresh()` may save pending changes before refreshing, so it should be used carefully.

Avoid unnecessary automatic saves or refreshes because they can:

- Interrupt users
- Trigger plug-ins and flows
- Cause duplicate processing
- Create race conditions
- Degrade performance

---

## 18. How do you retrieve a Dataverse record with JavaScript?

Use `Xrm.WebApi.retrieveRecord()`:

`const result = await Xrm.WebApi.retrieveRecord("account", accountId, "?$select=name,revenue");`

Important points:

- Use the table logical name.
- Remove braces from the GUID.
- Use `$select` to retrieve only required columns.
- Handle errors with `try/catch`.
- Lookup values are often returned using properties such as `_primarycontactid_value`.

Using `$select` improves performance and reduces network payload.

---

## 19. How do you retrieve multiple Dataverse records?

Use:

`const result = await Xrm.WebApi.retrieveMultipleRecords("contact", "?$select=fullname,emailaddress1&$filter=statecode eq 0&$top=50");`

Records are available in:

`result.entities`

If more records exist, pagination information may be returned through `nextLink`. Production code must handle paging rather than assuming the first response contains every record.

Use encoded and valid OData filters, and avoid retrieving all columns.

---

## 20. How do you create, update, and delete records through the Web API?

Create:

`const created = await Xrm.WebApi.createRecord("account", { name: "Contoso" });`

Update:

`await Xrm.WebApi.updateRecord("account", accountId, { telephone1: "123456789" });`

Delete:

`await Xrm.WebApi.deleteRecord("account", accountId);`

All operations return promises and should be awaited inside `try/catch`.

For lookups, Web API payloads commonly use `@odata.bind`, for example:

`"primarycontactid@odata.bind": "/contacts(<guid>)"`

The relationship and navigation property name must be correct.

---

## 21. What is the difference between `Xrm.WebApi.online` and `Xrm.WebApi.offline`?

- `Xrm.WebApi.online` executes operations against the online Dataverse service.
- `Xrm.WebApi.offline` works with data available in the mobile offline profile.
- `Xrm.WebApi` provides the standard interface and can select the appropriate behavior depending on context.

Offline mode has limitations around supported column types, operations, filters, and available records. Offline functionality must be designed and tested explicitly rather than assumed to behave identically to online mode.

---

## 22. Why should you use `async/await` instead of synchronous requests?

Synchronous requests block the browser’s UI thread and create a poor user experience.

`async/await` provides:

- Non-blocking execution
- Clearer code than nested callbacks
- Easier error handling
- Better sequencing of dependent operations

Example pattern:

`try { const account = await Xrm.WebApi.retrieveRecord(...); } catch (error) { console.error(error.message); }`

Independent requests can sometimes run concurrently with `Promise.all()`, but only when they do not depend on each other and the resulting load is reasonable.

---

## 23. How do you display errors, warnings, and confirmations?

Form-level notification:

`formContext.ui.setFormNotification(message, "ERROR", notificationId);`

Clear it with:

`formContext.ui.clearFormNotification(notificationId);`

Field-level notification:

`control.setNotification(message, notificationId);`

Clear it with:

`control.clearNotification(notificationId);`

Dialog:

`await Xrm.Navigation.openAlertDialog({ text: "Operation completed." });`

Confirmation:

`const result = await Xrm.Navigation.openConfirmDialog({ text: "Continue?" });`

Use notifications for actionable validation and avoid excessive intrusive alert dialogs.

---

## 24. How do you open forms and URLs?

Open a record form:

`await Xrm.Navigation.openForm({ entityName: "account", entityId: accountId });`

Open a Create form with default values:

`await Xrm.Navigation.openForm({ entityName: "contact", formParameters: { firstname: "Asha" } });`

Open a URL or web resource through the supported navigation APIs rather than using browser-specific or unsupported DOM techniques.

The Unified Interface is a single-page application, so direct assumptions about windows, frames, and page reloads are unreliable.

---

## 25. How do you filter a lookup dynamically?

Register a PreSearch handler:

`lookupControl.addPreSearch(addCustomFilter);`

Inside the handler, call `addCustomFilter()` with FetchXML filter XML.

Key considerations:

- Register the PreSearch function only once.
- Escape dynamic values used in XML.
- Use the correct related table logical name.
- Keep filters efficient.
- For custom lookup views, `addCustomView()` may be appropriate, subject to lookup-type limitations.

Lookup filtering improves user selection but should not be treated as a security boundary.

---

## 26. How do you work with tabs, sections, header fields, and Business Process Flow controls?

Tabs:

`formContext.ui.tabs.get("tab_general")`

Sections:

`formContext.ui.tabs.get("tab_general").sections.get("section_details")`

Business Process Flow:

`formContext.data.process`

Important caveats:

- Header or BPF controls may have generated names such as `header_process_<column>`.
- The same attribute can have multiple controls.
- BPF stage changes are asynchronous and may require process event handlers.
- Code should not depend on fragile control-name suffixes unless unavoidable.

Iterating over `attribute.controls` is often safer when all controls for an attribute must be updated.

---

## 27. How is JavaScript used in command-bar or ribbon buttons?

A command-bar action can call a JavaScript function for:

- Opening a dialog
- Updating a record
- Starting a custom process
- Calling a Custom API
- Performing validation

Display and enable rules determine when a button is visible or enabled.

Depending on command configuration, the function may receive `PrimaryControl`, which generally represents the form context:

`function runCommand(primaryControl) { const formContext = primaryControl; }`

The parameter type differs between form and grid commands, so the function must be designed for its command context.

---

## 28. How should JavaScript libraries be structured in Dynamics 365?

Use a namespace or module pattern to avoid polluting the global scope:

`Contoso.AccountForm.onLoad`

Recommended practices:

- Expose only functions required by form events or commands.
- Keep helper functions private where possible.
- Use constants for column and control names.
- Separate UI, validation, and API logic.
- Avoid duplicate libraries across forms.
- Remove obsolete functions and dependencies.
- Enable strict mode where compatible.
- Maintain source code outside minified deployment artifacts.

A structured library is easier to test, review, and maintain than many unrelated global functions.

---

## 29. What are the most important performance and security considerations?

### Performance

- Retrieve only required columns with `$select`.
- Avoid API calls inside unnecessary OnChange events.
- Cache stable data when appropriate.
- Avoid repeated handler registration.
- Do not refresh or save the form unnecessarily.
- Debounce high-frequency logic when appropriate.
- Avoid unsupported DOM manipulation.
- Load only required JavaScript libraries.

### Security

- JavaScript is not a security boundary.
- Users can inspect or bypass client-side code.
- Never store secrets, tokens, or privileged credentials in web resources.
- Dataverse security roles and field security must enforce access.
- Critical validation must also execute server-side.
- Validate and encode values used in URLs, OData, FetchXML, or HTML.

---

## 30. When should you use JavaScript, business rules, Power Automate, plug-ins, Custom APIs, or PCF?

Choose based on where and when the logic must run:

| Technology | Best suited for |
|---|---|
| JavaScript | Immediate form behavior and client-side validation |
| Business Rules | Simple declarative field logic |
| Power Automate | Asynchronous automation and system integrations |
| Plug-ins | Synchronous or asynchronous server-side business logic |
| Custom API | Reusable business operations exposed through Dataverse |
| PCF | Rich reusable custom UI controls |
| Classic workflow | Existing legacy processes; generally not preferred for new development |

A strong interview answer explains that critical business logic should not exist only in JavaScript. If validation must apply to imports, integrations, mobile, API clients, and background jobs, enforce it on the server.

---

# Five interview points candidates often miss

1. `Xrm.Page` is deprecated; use `executionContext.getFormContext()`.
2. `setValue()` does not automatically trigger OnChange.
3. Lookup values are arrays, even for single-value lookups.
4. Choice logic should compare numeric values, not translated labels.
5. Client-side validation is not security or authoritative business enforcement.
