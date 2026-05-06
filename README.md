# Developer Kickstart: Workers' Compensation Capstone

This repository is the capstone project for the Developer Kickstart curriculum at Cloud Code Academy. You will design and build a Salesforce solution that tracks workers' compensation claims for an HR team — from data model and CSV import through Apex, triggers, and record pages — while applying what you learned across the program.

## Goals of the Practice

Throughout this project, you will practice:

- Modeling standard and custom objects and relationships that fit real-world workers' comp scenarios (including many-to-many links between employees and claims).
- Importing the provided CSV data in `data/` with the Data Import Wizard or Data Loader — not manual entry.
- Implementing bulk-safe Apex and triggers for risk levels, claim duration, rollups, status rules, and high-value flags, as specified by your instructor.
- Delivering record pages and collaboration practices (GitHub, team workflow) that reflect how you would ship in a real org.

Your instructor will share the full requirements, timeline, presentation expectations, and rubric.

---

## Setup

### Setup Checklist

1. Create or configure a Trailhead Playground or Developer Edition org for this project.
2. Install [Visual Studio Code](https://code.visualstudio.com/).
3. Install **Salesforce Extension Pack** in VS Code (search Extensions for "Salesforce Extension Pack").
4. Authorize your org: **Cmd/Ctrl + Shift + P** → `SFDX: Authorize an Org` → complete login in the browser.
5. Deploy from your machine using the command palette or **SFDX: Deploy Source to Org** on folders or files.

---

## Getting Started

1. Open `force-app/main/default/` and deploy the metadata to your org (right-click the folder → **SFDX: Deploy Source to Org**).
2. Review the CSV files in `data/` and align your schema before importing.
3. Build to your instructor's requirements, deploy iteratively, and keep metadata in this repository.
4. Run Apex tests from VS Code if you add tests (**SFDX: Run Apex Tests**).
5. Push your work to GitHub and submit the repository link where your cohort asks (for example, the Slack submission form).

---

## Apex Classes — A Guide for First-Time Developers

Apex is Salesforce's strongly-typed, Java-like programming language. Everything you write runs on Salesforce's servers, not your machine. This section walks you through what Apex classes and triggers you will likely need for this project, how to think about each one, and the patterns that keep them production-ready.

### How to Create an Apex Class in VS Code

1. In VS Code open the Command Palette (`Cmd/Ctrl + Shift + P`).
2. Type `SFDX: Create Apex Class` and press Enter.
3. Give the class a name (e.g., `ClaimTriggerHandler`) and choose `force-app/main/default/classes` as the folder.
4. Two files are created: `ClassName.cls` and `ClassName.cls-meta.xml`. Both must be committed to Git.
5. Write your code, save, then deploy: right-click the `.cls` file → **SFDX: Deploy This Source to Org**.

### How to Create an Apex Trigger in VS Code

1. Open the Command Palette → `SFDX: Create Apex Trigger`.
2. Name it after the object it listens on (e.g., `EmployeeClaimTrigger`).
3. Choose `force-app/main/default/triggers` as the folder.
4. Two files are created: `TriggerName.trigger` and `TriggerName.trigger-meta.xml`. Commit both.

---

### The Handler Pattern — Keep Your Triggers Thin

The #1 best practice in Salesforce development is the **one trigger per object, handler class** pattern. A trigger should contain almost no logic — it just calls a handler class that does the work. This makes your code testable, readable, and easy to maintain.

```apex
// EmployeeClaimTrigger.trigger
trigger EmployeeClaimTrigger on Employee_Claim__c (
    after insert, after update, after delete, after undelete
) {
    EmployeeClaimTriggerHandler.handle(Trigger.new, Trigger.old, Trigger.operationType);
}
```

```apex
// EmployeeClaimTriggerHandler.cls
public class EmployeeClaimTriggerHandler {
    public static void handle(
        List<Employee_Claim__c> newRecords,
        List<Employee_Claim__c> oldRecords,
        System.TriggerOperation operationType
    ) {
        // Route to different methods based on the event
        if (operationType == System.TriggerOperation.AFTER_INSERT) {
            // call your method
        } else if (operationType == System.TriggerOperation.AFTER_DELETE) {
            // call your method
        }
        // etc.
    }
}
```

---

### Apex Classes You Will Likely Need

Below is a breakdown of the classes and triggers this project calls for. Read each section carefully — the goal is to understand what each piece does and why, not to copy code.

---

#### 1. Employee Claim Trigger + Handler (Core Requirement)

**What it does:** Fires whenever an `Employee_Claim__c` junction record is created, updated, deleted, or undeleted. It collects the affected employee IDs, queries their claims, and recalculates the risk level on each `Employee__c` record.

**Key concepts to understand before writing it:**

- **`Trigger.new` vs `Trigger.old`** — `Trigger.new` holds the records after the change; `Trigger.old` holds the before-state. On delete, only `Trigger.old` exists. You need to handle both to correctly collect affected employee IDs.
- **Bulkification** — Triggers fire once for a batch of records, not once per record. Never put a SOQL query or DML statement inside a `for` loop. Collect all the IDs you need first, run one SOQL query, process the results in memory, then run one DML update.
- **Maps for grouping** — The requirement asks you to group claim data by insurance provider per employee. A `Map<Id, Map<Id, Decimal>>` (employee → insurer → total amount) is the right tool. Think about how you would build that structure as you loop through query results.

**Skeleton to think about:**

```
1. Collect a Set<Id> of all employee IDs touched by this operation
   (from Trigger.new for insert/update, from Trigger.old for delete/undelete)

2. Query Employee_Claim__c records for those employees, joining to the Claim
   to get Settlement_Amount__c and Insurance_Provider__c

3. Loop through query results and build a Map grouping totals by
   (Employee Id → Insurance Provider Id → running total)

4. Loop through your map, apply your risk thresholds, and build a
   List<Employee__c> with the updated Risk_Level__c

5. Update that list in a single DML call outside the loop
```

---

#### 2. Claim Trigger + Handler

**What it does:** Fires on `Claim__c` insert and update. It handles three separate requirements:

- **Claim Status Automation** — sets `Settlement_Date__c` when status becomes Settled; clears `Settlement_Amount__c` when status becomes Denied; validates date logic.
- **High-Value Claim Flag** — sets or clears the `Is_High_Value__c` checkbox based on `Settlement_Amount__c`.
- **Days Open Calculation** — calculates the number of days the claim has been open and stores it in `Days_Open__c`.

**Key concepts:**

- **`Trigger.oldMap`** — a `Map<Id, SObject>` of the old record values. Use it to detect field changes: `if (newClaim.Status__c != Trigger.oldMap.get(newClaim.Id).Status__c)`.
- **`addError()`** — call this on a record to block the save and display a message to the user. It is the correct way to enforce validation in Apex (as opposed to declarative Validation Rules). Example: `newClaim.addError('Settlement Amount must be greater than zero when marking a claim as Settled.')`.
- **`Date` arithmetic** — `Date.today() - someDate` returns an `Integer` representing calendar days. The `Date` class also has `daysBetween()`. Think about which date you should subtract from and whether the claim being settled should change your calculation.
- **Inserting vs updating** — on insert `Trigger.oldMap` is null, so guard any old-value check: `if (!Trigger.isInsert && ...)`.

**Skeleton:**

```
For each claim in Trigger.new:
    // Status automation
    if status changed to 'Settled':
        validate Settlement Amount > 0, else addError()
        if Settlement Date is blank, set it to today

    if status changed to 'Denied':
        clear Settlement Amount

    // Date validation (on insert and update)
    validate Incident Date is not in the future
    validate Incident Date is not after Date Filed

    // Days open
    if status is Settled or Denied:
        Days Open = Date Filed - Incident Date
    else:
        Days Open = Date.today() - Incident Date

    // High-value flag
    if Settlement Amount > YOUR_THRESHOLD:
        Is High Value = true
    else:
        Is High Value = false
```

---

#### 3. Employee Claims Summary Helper (called from the Employee Claim handler)

**What it does:** Keeps `Total_Claims__c` and `Total_Settlement_Amount__c` on `Employee__c` accurate. Any time an `Employee_Claim__c` is inserted, updated, or deleted these numbers need to recalculate.

**Key concepts:**

- **Aggregate SOQL** — you can write a SOQL query that sums and counts in the database rather than loading every record into memory. Look into `AggregateResult` and the `SUM()` and `COUNT()` aggregate functions in SOQL.
- **Alternative: re-query and loop** — if aggregate SOQL feels too advanced, you can query all `Employee_Claim__c` records for the affected employees (joining to the `Claim__c` for `Settlement_Amount__c`), loop through them, and accumulate totals in a `Map<Id, Decimal>`.
- This logic fits naturally inside `EmployeeClaimTriggerHandler` since you are already collecting the affected employee IDs there.

**Skeleton:**

```
Given a Set<Id> of employee IDs that were affected:

Query all related Employee_Claim__c records for those employees
    (include the Claim's Settlement_Amount__c in your query)

Build a Map<Id, Integer> for claim counts
Build a Map<Id, Decimal> for settlement totals

Loop through query results and accumulate into those maps

Loop through the employee IDs and build a List<Employee__c>
    with Total_Claims__c and Total_Settlement_Amount__c set from the maps

DML update that list
```

---

### Apex Best Practices Checklist

Before submitting, review each class and trigger against this list:

- [ ] **No SOQL inside a loop.** Collect all IDs first, query once.
- [ ] **No DML inside a loop.** Build a list, update once.
- [ ] **One trigger per object.** All logic for `Employee_Claim__c` lives in one trigger + one handler.
- [ ] **Meaningful names.** `employeeIdSet` is better than `eIds`; `updateEmployeeRiskLevel()` is better than `doStuff()`.
- [ ] **Comments explain *why*, not *what*.** `// Exclude denied claims from payout total per business rule` is a good comment. `// loop through list` is not.
- [ ] **`addError()` messages are user-friendly.** Write them as if a real HR person will read them.
- [ ] **Test coverage exists** (optional but encouraged). At minimum, run your trigger scenarios manually in the org to verify they behave correctly in bulk.

---

### Common Mistakes to Avoid

| Mistake | Why It's a Problem | Fix |
|---|---|---|
| SOQL in a loop | Hits governor limits (max 100 SOQL queries per transaction) | Collect IDs → query once → process in memory |
| Separate triggers for the same object | Execution order is not guaranteed; logic splits across files | One trigger → one handler class |
| Hard-coding record IDs | IDs differ between orgs; code breaks on deploy | Query by Name or use `Custom Settings`/`Custom Metadata` for thresholds |
| Not handling `Trigger.old` on delete | Delete events only have `Trigger.old`, not `Trigger.new` | Always check which context you are in |
| Forgetting `.cls-meta.xml` | File won't deploy without its metadata companion | Always commit both files |

---

### Where to Put Each File

```
force-app/main/default/
├── classes/
│   ├── ClaimTriggerHandler.cls
│   ├── ClaimTriggerHandler.cls-meta.xml
│   ├── EmployeeClaimTriggerHandler.cls
│   └── EmployeeClaimTriggerHandler.cls-meta.xml
└── triggers/
    ├── ClaimTrigger.trigger
    ├── ClaimTrigger.trigger-meta.xml
    ├── EmployeeClaimTrigger.trigger
    └── EmployeeClaimTrigger.trigger-meta.xml
```

---

## Resources

If you get stuck, these may help:

- [Apex Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/)
- [Apex Trigger Best Practices](https://developer.salesforce.com/blogs/developer-relations/2015/01/apex-best-practices-15-apex-commandments)
- [SOQL and SOSL Reference](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/)
- [Salesforce Stack Exchange](https://salesforce.stackexchange.com/)
- [Visual Studio Code Documentation](https://code.visualstudio.com/docs)
- [Salesforce Extensions for Visual Studio Code](https://developer.salesforce.com/tools/vscode)

Good luck with your capstone.
