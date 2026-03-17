# SC Request — Salesforce Native (Option A) Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current Salesforce SC request flow with a Lightning Web Component modal that auto-assigns SCs, checks M365 calendar availability, and automates all downstream Salesforce writes and notifications.

**Architecture:** A Lightning Web Component (`scRequestModal`) triggered from the existing "Request SC" Quick Action on the Opportunity. An Apex controller (`ScRequestController`) handles all DML. A separate Apex service (`M365CalendarService`) handles M365 free/busy and calendar write via Named Credential. A Record-Triggered Flow fires email alerts on `SC_Request__c` insert.

**Tech Stack:** Salesforce LWC, Apex, SOQL, Named Credentials, External Services (M365 Graph API), Record-Triggered Flow, Salesforce CLI (`sf`), Jest (LWC unit tests), ApexMocks or Apex test classes

---

## Chunk 1: Custom Objects & Fields

### Task 1: Create `SC_Pairing__c` custom object

**Files:**
- Create: `force-app/main/default/objects/SC_Pairing__c/SC_Pairing__c.object-meta.xml`
- Create: `force-app/main/default/objects/SC_Pairing__c/fields/AE__c.field-meta.xml`
- Create: `force-app/main/default/objects/SC_Pairing__c/fields/SC__c.field-meta.xml`
- Create: `force-app/main/default/objects/SC_Pairing__c/fields/Active__c.field-meta.xml`

- [ ] **Step 1: Initialize Salesforce DX project if not already done**

```bash
sf project generate --name sc-request --output-dir . --template standard
```

Expected: `force-app/` directory created with standard structure.

- [ ] **Step 2: Create `SC_Pairing__c` object metadata**

Create `force-app/main/default/objects/SC_Pairing__c/SC_Pairing__c.object-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>SC Pairing</label>
    <pluralLabel>SC Pairings</pluralLabel>
    <nameField>
        <label>Pairing Name</label>
        <type>AutoNumber</type>
        <displayFormat>PAIR-{0000}</displayFormat>
    </nameField>
    <deploymentStatus>Deployed</deploymentStatus>
    <sharingModel>ReadWrite</sharingModel>
    <description>Stores AE to SC assignment data. Replaces external spreadsheet. One active pairing per AE enforced by validation rule.</description>
</CustomObject>
```

- [ ] **Step 3: Create `AE__c` lookup field**

Create `force-app/main/default/objects/SC_Pairing__c/fields/AE__c.field-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>AE__c</fullName>
    <label>Account Executive</label>
    <type>Lookup</type>
    <referenceTo>User</referenceTo>
    <relationshipLabel>SC Pairings (AE)</relationshipLabel>
    <relationshipName>SC_Pairings_AE</relationshipName>
    <required>true</required>
    <description>The Account Executive in this pairing.</description>
</CustomField>
```

- [ ] **Step 4: Create `SC__c` lookup field**

Create `force-app/main/default/objects/SC_Pairing__c/fields/SC__c.field-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>SC__c</fullName>
    <label>Solutions Consultant</label>
    <type>Lookup</type>
    <referenceTo>User</referenceTo>
    <relationshipLabel>SC Pairings (SC)</relationshipLabel>
    <relationshipName>SC_Pairings_SC</relationshipName>
    <required>true</required>
    <description>The Solutions Consultant assigned to this AE.</description>
</CustomField>
```

- [ ] **Step 5: Create `Active__c` checkbox field**

Create `force-app/main/default/objects/SC_Pairing__c/fields/Active__c.field-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Active__c</fullName>
    <label>Active</label>
    <type>Checkbox</type>
    <defaultValue>true</defaultValue>
    <description>Whether this pairing is currently active. Only one active pairing per AE is allowed.</description>
</CustomField>
```

- [ ] **Step 6: Create validation rule — one active pairing per AE**

Create `force-app/main/default/objects/SC_Pairing__c/validationRules/One_Active_Pairing_Per_AE.validationRule-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ValidationRule xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>One_Active_Pairing_Per_AE</fullName>
    <active>true</active>
    <description>Prevents creating a second active pairing for the same AE. Deactivate the existing pairing before creating a new one.</description>
    <errorConditionFormula>AND(
  Active__c = TRUE,
  VLOOKUP($ObjectType.SC_Pairing__c.Fields.Id, $ObjectType.SC_Pairing__c.Fields.AE__c, AE__c) &lt;&gt; Id
)</errorConditionFormula>
    <errorMessage>This AE already has an active SC pairing. Please deactivate the existing pairing before creating a new one.</errorMessage>
</ValidationRule>
```

- [ ] **Step 7: Deploy to scratch org and verify**

```bash
sf org create scratch --definition-file config/project-scratch-def.json --alias sc-dev --set-default
sf project deploy start --source-dir force-app/main/default/objects/SC_Pairing__c
```

Expected: Deploy succeeds. Open scratch org and verify SC_Pairing__c appears in Setup → Object Manager.

- [ ] **Step 8: Commit**

```bash
git add force-app/
git commit -m "feat: add SC_Pairing__c custom object with validation rule"
```

---

### Task 2: Create `SC_Request__c` custom object

**Files:**
- Create: `force-app/main/default/objects/SC_Request__c/SC_Request__c.object-meta.xml`
- Create: `force-app/main/default/objects/SC_Request__c/fields/` (7 field files)

- [ ] **Step 1: Create object metadata**

Create `force-app/main/default/objects/SC_Request__c/SC_Request__c.object-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>SC Request</label>
    <pluralLabel>SC Requests</pluralLabel>
    <nameField>
        <label>Request Name</label>
        <type>AutoNumber</type>
        <displayFormat>SCR-{00000}</displayFormat>
    </nameField>
    <deploymentStatus>Deployed</deploymentStatus>
    <sharingModel>ReadWrite</sharingModel>
    <description>Stores SC requests for reporting, history, and display on the Opportunity.</description>
</CustomObject>
```

- [ ] **Step 2: Create all 9 fields**

Create `force-app/main/default/objects/SC_Request__c/fields/Opportunity__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Opportunity__c</fullName>
    <label>Opportunity</label>
    <type>MasterDetail</type>
    <referenceTo>Opportunity</referenceTo>
    <relationshipLabel>SC Requests</relationshipLabel>
    <relationshipName>SC_Requests</relationshipName>
    <relationshipOrder>0</relationshipOrder>
    <required>true</required>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Requested_SC__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Requested_SC__c</fullName>
    <label>Requested SC</label>
    <type>Lookup</type>
    <referenceTo>User</referenceTo>
    <relationshipLabel>SC Requests</relationshipLabel>
    <relationshipName>SC_Requests_SC</relationshipName>
    <required>true</required>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Request_Type__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Request_Type__c</fullName>
    <label>Request Type</label>
    <type>Picklist</type>
    <required>true</required>
    <valueSet>
        <valueSetDefinition>
            <sorted>true</sorted>
            <value><fullName>Biz Dev/Channel</fullName><default>false</default></value>
            <value><fullName>CSM Activity</fullName><default>false</default></value>
            <value><fullName>Demo Request</fullName><default>false</default></value>
            <value><fullName>Discovery Call</fullName><default>false</default></value>
            <value><fullName>Document Request</fullName><default>false</default></value>
            <value><fullName>On-Site Meeting</fullName><default>false</default></value>
            <value><fullName>Pro Serv Scoping</fullName><default>false</default></value>
            <value><fullName>RFX/Tender</fullName><default>false</default></value>
            <value><fullName>Security Questionnaire</fullName><default>false</default></value>
            <value><fullName>Security Team Escalation</fullName><default>false</default></value>
            <value><fullName>Strategic Planning</fullName><default>false</default></value>
            <value><fullName>Technical Call</fullName><default>false</default></value>
            <value><fullName>Training</fullName><default>false</default></value>
            <value><fullName>Trial Engagement/POC</fullName><default>false</default></value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Product_Line__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Product_Line__c</fullName>
    <label>Product Line</label>
    <type>Picklist</type>
    <required>true</required>
    <valueSet>
        <valueSetDefinition>
            <sorted>true</sorted>
            <value><fullName>Central</fullName><default>false</default></value>
            <value><fullName>EnhancedAudio</fullName><default>false</default></value>
            <value><fullName>GoTo Resolve</fullName><default>false</default></value>
            <value><fullName>GoToAssist</fullName><default>false</default></value>
            <value><fullName>GoToConnect</fullName><default>false</default></value>
            <value><fullName>GoToMeeting</fullName><default>false</default></value>
            <value><fullName>GoToMyPC</fullName><default>false</default></value>
            <value><fullName>GoToRoom</fullName><default>false</default></value>
            <value><fullName>GoToTraining</fullName><default>false</default></value>
            <value><fullName>GoToWebcast</fullName><default>false</default></value>
            <value><fullName>GoToWebinar</fullName><default>false</default></value>
            <value><fullName>ITSG</fullName><default>false</default></value>
            <value><fullName>MDM</fullName><default>false</default></value>
            <value><fullName>None</fullName><default>false</default></value>
            <value><fullName>Pro</fullName><default>false</default></value>
            <value><fullName>Rescue</fullName><default>false</default></value>
            <value><fullName>UCC</fullName><default>false</default></value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Request_Notes__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Request_Notes__c</fullName>
    <label>Request Notes</label>
    <type>LongTextArea</type>
    <length>32768</length>
    <visibleLines>5</visibleLines>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Meeting_Date__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Meeting_Date__c</fullName>
    <label>Meeting Date</label>
    <type>DateTime</type>
    <required>true</required>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Submitted_Date__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Submitted_Date__c</fullName>
    <label>Submitted Date</label>
    <type>Formula</type>
    <formula>CreatedDate</formula>
    <returnType>DateTime</returnType>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Status__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Status__c</fullName>
    <label>Status</label>
    <type>Picklist</type>
    <required>true</required>
    <valueSet>
        <valueSetDefinition>
            <sorted>false</sorted>
            <value><fullName>Submitted</fullName><default>true</default></value>
            <value><fullName>Confirmed</fullName><default>false</default></value>
            <value><fullName>Completed</fullName><default>false</default></value>
            <value><fullName>Cancelled</fullName><default>false</default></value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

Create `force-app/main/default/objects/SC_Request__c/fields/Submitted_By__c.field-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Submitted_By__c</fullName>
    <label>Submitted By</label>
    <type>Lookup</type>
    <referenceTo>User</referenceTo>
    <relationshipLabel>SC Requests (Submitted By)</relationshipLabel>
    <relationshipName>SC_Requests_Submitted_By</relationshipName>
    <required>true</required>
    <description>The AE who submitted this SC request.</description>
</CustomField>
```

- [ ] **Step 3: Deploy and verify**

```bash
sf project deploy start --source-dir force-app/main/default/objects/SC_Request__c
```

Expected: Deploy succeeds. Verify object appears in Object Manager with all 7 fields.

- [ ] **Step 4: Commit**

```bash
git add force-app/
git commit -m "feat: add SC_Request__c custom object with all fields"
```

---

### Task 3: Add `SC_Request_Notes__c` field to Opportunity

**Files:**
- Create: `force-app/main/default/objects/Opportunity/fields/SC_Request_Notes__c.field-meta.xml`

- [ ] **Step 1: Create field metadata**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>SC_Request_Notes__c</fullName>
    <label>SC Request Notes</label>
    <type>LongTextArea</type>
    <length>32768</length>
    <visibleLines>5</visibleLines>
    <description>AE's freeform request details submitted via SC Request. Separate from SC_Notes__c which is SC-owned.</description>
</CustomField>
```

- [ ] **Step 2: Deploy and verify**

```bash
sf project deploy start --source-dir force-app/main/default/objects/Opportunity/fields/SC_Request_Notes__c.field-meta.xml
```

- [ ] **Step 3: Commit**

```bash
git add force-app/
git commit -m "feat: add SC_Request_Notes__c field to Opportunity"
```

---

## Chunk 2: Apex Controller & M365 Service

### Task 4: Write `ScRequestController` Apex class (TDD)

**Files:**
- Create: `force-app/main/default/classes/ScRequestController.cls`
- Create: `force-app/main/default/classes/ScRequestController.cls-meta.xml`
- Create: `force-app/main/default/classes/ScRequestControllerTest.cls`
- Create: `force-app/main/default/classes/ScRequestControllerTest.cls-meta.xml`

- [ ] **Step 1: Write the failing test first**

Create `force-app/main/default/classes/ScRequestControllerTest.cls`:

```apex
@IsTest
private class ScRequestControllerTest {

    @TestSetup
    static void makeData() {
        User ae = TestDataFactory.createUser('AE User', 'ae.test@goto.com.test');
        User sc = TestDataFactory.createUser('SC User', 'sc.test@goto.com.test');
        User aeManager = TestDataFactory.createUser('AE Manager', 'ae.manager@goto.com.test');
        User scManager = TestDataFactory.createUser('SC Manager', 'sc.manager@goto.com.test');

        ae.ManagerId = aeManager.Id;
        sc.ManagerId = scManager.Id;
        update new List<User>{ ae, sc };

        SC_Pairing__c pairing = new SC_Pairing__c(AE__c = ae.Id, SC__c = sc.Id, Active__c = true);
        insert pairing;

        Account acc = new Account(Name = 'Test Account');
        insert acc;
        Opportunity opp = new Opportunity(
            Name = 'Test Opp', AccountId = acc.Id,
            StageName = 'Prospecting', CloseDate = Date.today().addDays(30)
        );
        insert opp;
    }

    @IsTest
    static void getAssignedSC_returnsPairedSC() {
        User ae = [SELECT Id FROM User WHERE Email = 'ae.test@goto.com.test' LIMIT 1];
        User sc = [SELECT Id FROM User WHERE Email = 'sc.test@goto.com.test' LIMIT 1];

        Test.startTest();
        System.runAs(ae) {
            ScRequestController.SCSelectionResult result = ScRequestController.getAssignedSC();
            System.assertEquals(sc.Id, result.assignedSCId, 'Should return paired SC Id');
            System.assertEquals(false, result.hasMultiplePairings, 'Should not flag multiple pairings');
        }
        Test.stopTest();
    }

    @IsTest
    static void getAssignedSC_nopairing_returnsNull() {
        User ae = new User();
        // Insert a user with no pairing
        ae = TestDataFactory.createUser('Unpaired AE', 'unpaired@goto.com.test');
        Test.startTest();
        System.runAs(ae) {
            ScRequestController.SCSelectionResult result = ScRequestController.getAssignedSC();
            System.assertNull(result.assignedSCId, 'Should return null when no pairing exists');
        }
        Test.stopTest();
    }

    @IsTest
    static void submitRequest_createsAllRecords() {
        User ae = [SELECT Id FROM User WHERE Email = 'ae.test@goto.com.test' LIMIT 1];
        User sc = [SELECT Id FROM User WHERE Email = 'sc.test@goto.com.test' LIMIT 1];
        Opportunity opp = [SELECT Id, AccountId FROM Opportunity LIMIT 1];

        ScRequestController.SCRequestInput input = new ScRequestController.SCRequestInput();
        input.opportunityId = opp.Id;
        input.scId = sc.Id;
        input.requestType = 'Demo Request';
        input.productLine = 'GoToConnect';
        input.requestNotes = 'Test notes for demo';
        input.meetingDate = Datetime.now().addDays(3);
        input.durationMinutes = 60;
        input.sendCalendarInvite = false;

        Test.startTest();
        System.runAs(ae) {
            ScRequestController.submitRequest(input);
        }
        Test.stopTest();

        // Verify SC_Request__c created
        List<SC_Request__c> requests = [SELECT Id, Request_Type__c, Product_Line__c FROM SC_Request__c WHERE Opportunity__c = :opp.Id];
        System.assertEquals(1, requests.size(), 'Should create one SC_Request__c record');
        System.assertEquals('Demo Request', requests[0].Request_Type__c);

        // Verify Event created
        List<Event> events = [SELECT Id, Subject FROM Event WHERE WhatId = :opp.Id];
        System.assertEquals(1, events.size(), 'Should create one Event');
        System.assert(events[0].Subject.contains('Demo Request'), 'Event subject should contain request type');

        // Verify OpportunityTeamMember created
        List<OpportunityTeamMember> members = [SELECT Id, TeamMemberRole FROM OpportunityTeamMember WHERE OpportunityId = :opp.Id AND UserId = :sc.Id];
        System.assertEquals(1, members.size(), 'Should add SC to opportunity team');
        System.assertEquals('Solution Consultant', members[0].TeamMemberRole);

        // Verify Opportunity fields updated
        Opportunity updatedOpp = [
            SELECT SC_Request_Notes__c, SC_On_Team__c, Primary_Solution_Consultant__c, SC_Date_Requested__c
            FROM Opportunity WHERE Id = :opp.Id
        ];
        System.assertEquals('Test notes for demo', updatedOpp.SC_Request_Notes__c);
        System.assertEquals(true, updatedOpp.SC_On_Team__c);
        System.assertEquals(sc.Id, updatedOpp.Primary_Solution_Consultant__c, 'Primary SC should be set');
        System.assertNotEquals(null, updatedOpp.SC_Date_Requested__c, 'SC Date Requested should be set');
    }

    @IsTest
    static void submitRequest_demoType_checksDemoCheckbox() {
        User ae = [SELECT Id FROM User WHERE Email = 'ae.test@goto.com.test' LIMIT 1];
        User sc = [SELECT Id FROM User WHERE Email = 'sc.test@goto.com.test' LIMIT 1];
        Opportunity opp = [SELECT Id FROM Opportunity LIMIT 1];

        ScRequestController.SCRequestInput input = new ScRequestController.SCRequestInput();
        input.opportunityId = opp.Id;
        input.scId = sc.Id;
        input.requestType = 'Demo Request';
        input.productLine = 'GoToConnect';
        input.requestNotes = 'Demo notes';
        input.meetingDate = Datetime.now().addDays(2);
        input.durationMinutes = 60;
        input.sendCalendarInvite = false;

        Test.startTest();
        System.runAs(ae) { ScRequestController.submitRequest(input); }
        Test.stopTest();

        Opportunity updatedOpp = [SELECT Demo__c FROM Opportunity WHERE Id = :opp.Id];
        System.assertEquals(true, updatedOpp.Demo__c, 'Demo checkbox should be checked for Demo Request type');
    }
}
```

- [ ] **Step 2: Run tests to verify they fail (class doesn't exist yet)**

```bash
sf apex run test --class-names ScRequestControllerTest --result-format human --synchronous
```

Expected: FAIL — `ScRequestController` does not exist.

- [ ] **Step 3: Create `TestDataFactory` helper**

Create `force-app/main/default/classes/TestDataFactory.cls`:

```apex
@IsTest
public class TestDataFactory {
    public static User createUser(String name, String email) {
        Profile p = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        String alias = email.substring(0, Math.min(8, email.indexOf('@')));
        User u = new User(
            FirstName = name.split(' ')[0],
            LastName = name.split(' ').size() > 1 ? name.split(' ')[1] : 'User',
            Email = email,
            Username = email,
            Alias = alias,
            TimeZoneSidKey = 'America/New_York',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US',
            ProfileId = p.Id
        );
        insert u;
        return u;
    }
}
```

Create `force-app/main/default/classes/TestDataFactory.cls-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

- [ ] **Step 4: Implement `ScRequestController`**

Create `force-app/main/default/classes/ScRequestController.cls`:

```apex
public with sharing class ScRequestController {

    public class SCSelectionResult {
        @AuraEnabled public Id assignedSCId;
        @AuraEnabled public Boolean hasMultiplePairings;
        @AuraEnabled public List<SCOption> allSCs;
    }

    public class SCOption {
        @AuraEnabled public Id id;
        @AuraEnabled public String name;
    }

    public class SCRequestInput {
        @AuraEnabled public Id opportunityId;
        @AuraEnabled public Id scId;
        @AuraEnabled public String requestType;
        @AuraEnabled public String productLine;
        @AuraEnabled public String requestNotes;
        @AuraEnabled public Datetime meetingDate;
        @AuraEnabled public Integer durationMinutes; // 30, 60, or 90
        @AuraEnabled public Boolean sendCalendarInvite;
    }

    @AuraEnabled(cacheable=true)
    public static SCSelectionResult getAssignedSC() {
        Id currentUserId = UserInfo.getUserId();
        SCSelectionResult result = new SCSelectionResult();
        result.hasMultiplePairings = false;

        List<SC_Pairing__c> pairings = [
            SELECT SC__c, CreatedDate
            FROM SC_Pairing__c
            WHERE AE__c = :currentUserId AND Active__c = true
            ORDER BY CreatedDate DESC
        ];

        if (!pairings.isEmpty()) {
            result.assignedSCId = pairings[0].SC__c;
            result.hasMultiplePairings = pairings.size() > 1;
            if (result.hasMultiplePairings) {
                // Notify the SC's manager to clean up duplicate pairings
                User sc = [SELECT Manager.Email, Manager.Name FROM User WHERE Id = :pairings[0].SC__c LIMIT 1];
                if (sc.Manager != null && String.isNotBlank(sc.Manager.Email)) {
                    Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
                    mail.setToAddresses(new List<String>{ sc.Manager.Email });
                    mail.setSubject('Action Required: Duplicate SC Pairing Detected');
                    mail.setPlainTextBody(
                        'AE ' + UserInfo.getName() + ' has multiple active SC pairings in Salesforce. ' +
                        'Please review SC_Pairing__c records for this AE and deactivate any stale entries.\n\n' +
                        'AE User Id: ' + currentUserId
                    );
                    Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
                }
            }
        }

        result.allSCs = getAllActiveSCs();
        return result;
    }

    @AuraEnabled(cacheable=true)
    public static List<SCOption> getAllActiveSCs() {
        List<SCOption> options = new List<SCOption>();
        for (User u : [
            SELECT Id, Name FROM User
            WHERE IsActive = true
            AND UserRoleId IN (SELECT Id FROM UserRole WHERE Name LIKE '%Solution%')
            ORDER BY Name
        ]) {
            SCOption opt = new SCOption();
            opt.id = u.Id;
            opt.name = u.Name;
            options.add(opt);
        }
        return options;
    }

    @AuraEnabled
    public static void submitRequest(SCRequestInput input) {
        Savepoint sp = Database.setSavepoint();
        try {
            Opportunity opp = [
                SELECT Id, Name, Account.Name, OwnerId, Owner.ManagerId,
                       SC_On_Team__c, Demo__c, SC_Request_Notes__c,
                       Primary_Solution_Consultant__c, SC_Date_Requested__c
                FROM Opportunity WHERE Id = :input.opportunityId
            ];

            User sc = [SELECT Id, Name, ManagerId FROM User WHERE Id = :input.scId];

            // 1. Create SC_Request__c
            SC_Request__c request = new SC_Request__c(
                Opportunity__c = input.opportunityId,
                Requested_SC__c = input.scId,
                Request_Type__c = input.requestType,
                Product_Line__c = input.productLine,
                Request_Notes__c = input.requestNotes,
                Meeting_Date__c = input.meetingDate,
                Status__c = 'Submitted',
                Submitted_By__c = UserInfo.getUserId()
            );
            insert request;

            // 2. Create Event
            String eventSubject = 'Request for ' + input.requestType +
                                  ' for ' + opp.Account.Name +
                                  ' w/ a Solutions Consultant';
            Integer duration = (input.durationMinutes != null && input.durationMinutes > 0) ? input.durationMinutes : 60;
            Event evt = new Event(
                Subject = eventSubject,
                Description = input.requestNotes,
                StartDateTime = input.meetingDate,
                EndDateTime = input.meetingDate.addMinutes(duration),
                WhatId = input.opportunityId,
                OwnerId = UserInfo.getUserId(),
                Type = input.requestType
            );
            insert evt;

            // 3. Add SC to Opportunity Team
            OpportunityTeamMember member = new OpportunityTeamMember(
                OpportunityId = input.opportunityId,
                UserId = input.scId,
                TeamMemberRole = 'Solution Consultant',
                OpportunityAccessLevel = 'Edit'
            );
            insert member;

            // 4. Update Opportunity fields
            opp.Primary_Solution_Consultant__c = input.scId;
            opp.SC_Date_Requested__c = input.meetingDate;
            opp.SC_On_Team__c = true;
            opp.SC_Request_Notes__c = input.requestNotes;
            if (input.requestType == 'Demo Request') {
                opp.Demo__c = true;
            }
            update opp;

            // 5. Send Outlook calendar invite if requested
            if (input.sendCalendarInvite) {
                M365CalendarService.createCalendarEvent(
                    input.scId, UserInfo.getUserId(),
                    eventSubject, input.requestNotes,
                    input.meetingDate, input.meetingDate.addMinutes(duration)
                );
            }

        } catch (Exception e) {
            Database.rollback(sp);
            throw new AuraHandledException('Request submission failed: ' + e.getMessage());
        }
    }
}
```

Create `force-app/main/default/classes/ScRequestController.cls-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

- [ ] **Step 5: Run tests and verify they pass**

```bash
sf apex run test --class-names ScRequestControllerTest --result-format human --synchronous
```

Expected: All 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add force-app/
git commit -m "feat: add ScRequestController with full submit logic and tests"
```

---

### Task 5: Write `M365CalendarService` Apex class (TDD)

**Files:**
- Create: `force-app/main/default/classes/M365CalendarService.cls`
- Create: `force-app/main/default/classes/M365CalendarService.cls-meta.xml`
- Create: `force-app/main/default/classes/M365CalendarServiceTest.cls`
- Create: `force-app/main/default/classes/M365CalendarServiceTest.cls-meta.xml`

- [ ] **Step 1: Write failing tests**

Create `force-app/main/default/classes/M365CalendarServiceTest.cls`:

```apex
@IsTest
private class M365CalendarServiceTest {

    @IsTest
    static void getAvailableSlots_returnsFiveSlots() {
        Test.setMock(HttpCalloutMock.class, new M365CalendarMock.FreeBusySuccess());

        Test.startTest();
        List<M365CalendarService.TimeSlot> slots = M365CalendarService.getAvailableSlots(
            UserInfo.getUserId(), UserInfo.getUserId(),
            Datetime.now(), Datetime.now().addDays(5), 60
        );
        Test.stopTest();

        System.assert(slots.size() <= 5, 'Should return at most 5 slots');
        System.assert(!slots.isEmpty(), 'Should return at least one slot on success');
    }

    @IsTest
    static void getAvailableSlots_apiFailure_returnsEmpty() {
        Test.setMock(HttpCalloutMock.class, new M365CalendarMock.FreeBusyFailure());

        Test.startTest();
        List<M365CalendarService.TimeSlot> slots = M365CalendarService.getAvailableSlots(
            UserInfo.getUserId(), UserInfo.getUserId(),
            Datetime.now(), Datetime.now().addDays(5), 60
        );
        Test.stopTest();

        System.assert(slots.isEmpty(), 'Should return empty list on API failure');
    }

    @IsTest
    static void createCalendarEvent_success_returnsTrue() {
        Test.setMock(HttpCalloutMock.class, new M365CalendarMock.CreateEventSuccess());

        Test.startTest();
        Boolean result = M365CalendarService.createCalendarEvent(
            UserInfo.getUserId(), UserInfo.getUserId(),
            'Test Meeting', 'Test notes',
            Datetime.now().addDays(2), Datetime.now().addDays(2).addHours(1)
        );
        Test.stopTest();

        System.assertEquals(true, result, 'Should return true on success');
    }
}
```

- [ ] **Step 2: Create `M365CalendarMock` test helper**

Create `force-app/main/default/classes/M365CalendarMock.cls`:

```apex
@IsTest
public class M365CalendarMock {
    public class FreeBusySuccess implements HttpCalloutMock {
        public HTTPResponse respond(HTTPRequest req) {
            HTTPResponse res = new HTTPResponse();
            res.setStatusCode(200);
            res.setBody('{"value":[{"scheduleId":"test@goto.com","availabilityView":"000002222200000"}]}');
            return res;
        }
    }
    public class FreeBusyFailure implements HttpCalloutMock {
        public HTTPResponse respond(HTTPRequest req) {
            HTTPResponse res = new HTTPResponse();
            res.setStatusCode(503);
            res.setBody('{"error":"Service unavailable"}');
            return res;
        }
    }
    public class CreateEventSuccess implements HttpCalloutMock {
        public HTTPResponse respond(HTTPRequest req) {
            HTTPResponse res = new HTTPResponse();
            res.setStatusCode(201);
            res.setBody('{"id":"event123","subject":"Test Meeting"}');
            return res;
        }
    }
}
```

Create `force-app/main/default/classes/M365CalendarMock.cls-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
sf apex run test --class-names M365CalendarServiceTest --result-format human --synchronous
```

Expected: FAIL — `M365CalendarService` does not exist.

- [ ] **Step 4: Implement `M365CalendarService`**

Create `force-app/main/default/classes/M365CalendarService.cls`:

```apex
public with sharing class M365CalendarService {

    private static final String NAMED_CREDENTIAL = 'M365_Graph_API';
    private static final Integer BUSINESS_HOURS_START = 8;
    private static final Integer BUSINESS_HOURS_END = 18;

    public class TimeSlot {
        @AuraEnabled public Datetime startTime;
        @AuraEnabled public Datetime endTime;
        @AuraEnabled public String label;
    }

    @AuraEnabled(cacheable=false)
    public static List<TimeSlot> getAvailableSlots(
        Id aeUserId, Id scUserId,
        Datetime rangeStart, Datetime rangeEnd,
        Integer durationMinutes
    ) {
        String aeEmail = getUserEmail(aeUserId);
        String scEmail = getUserEmail(scUserId);

        String body = JSON.serialize(new Map<String, Object>{
            'schedules' => new List<String>{ aeEmail, scEmail },
            'startTime' => new Map<String, String>{
                'dateTime' => rangeStart.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss'),
                'timeZone' => 'UTC'
            },
            'endTime' => new Map<String, String>{
                'dateTime' => rangeEnd.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss'),
                'timeZone' => 'UTC'
            },
            'availabilityViewInterval' => 30
        });

        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:' + NAMED_CREDENTIAL + '/v1.0/me/calendar/getSchedule');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(body);

        try {
            HttpResponse res = new Http().send(req);
            if (res.getStatusCode() != 200) return new List<TimeSlot>();
            return parseAvailableSlots(res.getBody(), rangeStart, durationMinutes);
        } catch (Exception e) {
            return new List<TimeSlot>();
        }
    }

    @AuraEnabled
    public static Boolean createCalendarEvent(
        Id scUserId, Id aeUserId,
        String subject, String notes,
        Datetime startDt, Datetime endDt
    ) {
        String scEmail = getUserEmail(scUserId);
        String aeEmail = getUserEmail(aeUserId);

        String body = JSON.serialize(new Map<String, Object>{
            'subject' => subject,
            'body' => new Map<String, String>{ 'contentType' => 'Text', 'content' => notes },
            'start' => new Map<String, String>{
                'dateTime' => startDt.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss'),
                'timeZone' => 'UTC'
            },
            'end' => new Map<String, String>{
                'dateTime' => endDt.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss'),
                'timeZone' => 'UTC'
            },
            'attendees' => new List<Map<String, Object>>{
                new Map<String, Object>{
                    'emailAddress' => new Map<String, String>{ 'address' => scEmail },
                    'type' => 'required'
                }
            }
        });

        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:' + NAMED_CREDENTIAL + '/v1.0/me/events');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(body);

        try {
            HttpResponse res = new Http().send(req);
            return res.getStatusCode() == 201;
        } catch (Exception e) {
            return false;
        }
    }

    private static String getUserEmail(Id userId) {
        return [SELECT Email FROM User WHERE Id = :userId LIMIT 1].Email;
    }

    private static List<TimeSlot> parseAvailableSlots(
        String responseBody, Datetime rangeStart, Integer durationMinutes
    ) {
        List<TimeSlot> slots = new List<TimeSlot>();
        Map<String, Object> parsed = (Map<String, Object>) JSON.deserializeUntyped(responseBody);
        List<Object> schedules = (List<Object>) parsed.get('value');
        if (schedules == null || schedules.isEmpty()) return slots;

        // Build combined busy map: index = 30-min block, true = busy
        List<Boolean> combinedBusy = new List<Boolean>();
        for (Object schedObj : schedules) {
            Map<String, Object> sched = (Map<String, Object>) schedObj;
            String av = (String) sched.get('availabilityView');
            for (Integer i = 0; i < av.length(); i++) {
                Boolean isBusy = av.substring(i, i+1) != '0';
                if (combinedBusy.size() <= i) combinedBusy.add(isBusy);
                else if (isBusy) combinedBusy.set(i, true);
            }
        }

        Integer blocksNeeded = durationMinutes / 30;
        for (Integer i = 0; i <= combinedBusy.size() - blocksNeeded; i++) {
            Boolean allFree = true;
            for (Integer j = i; j < i + blocksNeeded; j++) {
                if (combinedBusy[j]) { allFree = false; break; }
            }
            if (!allFree) continue;

            Datetime slotStart = rangeStart.addMinutes(i * 30);
            Integer hour = slotStart.hourGmt();
            if (hour < BUSINESS_HOURS_START || hour >= BUSINESS_HOURS_END) continue;

            TimeSlot ts = new TimeSlot();
            ts.startTime = slotStart;
            ts.endTime = slotStart.addMinutes(durationMinutes);
            ts.label = slotStart.format('EEE, MMM d \'at\' h:mm a');
            slots.add(ts);

            if (slots.size() == 5) break;
        }
        return slots;
    }
}
```

Create `force-app/main/default/classes/M365CalendarService.cls-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

- [ ] **Step 5: Run tests and verify they pass**

```bash
sf apex run test --class-names M365CalendarServiceTest --result-format human --synchronous
```

Expected: All 3 tests PASS.

- [ ] **Step 6: Deploy both classes**

```bash
sf project deploy start --source-dir force-app/main/default/classes
```

- [ ] **Step 7: Commit**

```bash
git add force-app/
git commit -m "feat: add M365CalendarService with free/busy and calendar write"
```

---

## Chunk 3: Lightning Web Component Modal

### Task 6: Create `scRequestModal` LWC

**Files:**
- Create: `force-app/main/default/lwc/scRequestModal/scRequestModal.js`
- Create: `force-app/main/default/lwc/scRequestModal/scRequestModal.html`
- Create: `force-app/main/default/lwc/scRequestModal/scRequestModal.css`
- Create: `force-app/main/default/lwc/scRequestModal/scRequestModal.js-meta.xml`
- Create: `force-app/main/default/lwc/scRequestModal/__tests__/scRequestModal.test.js`

- [ ] **Step 1: Write failing Jest tests**

Create `force-app/main/default/lwc/scRequestModal/__tests__/scRequestModal.test.js`:

```javascript
import { createElement } from 'lwc';
import scRequestModal from 'c/scRequestModal';
import getAssignedSC from '@salesforce/apex/ScRequestController.getAssignedSC';
import getAvailableSlots from '@salesforce/apex/M365CalendarService.getAvailableSlots';
import submitRequest from '@salesforce/apex/ScRequestController.submitRequest';

jest.mock('@salesforce/apex/ScRequestController.getAssignedSC', () => jest.fn(), { virtual: true });
jest.mock('@salesforce/apex/M365CalendarService.getAvailableSlots', () => jest.fn(), { virtual: true });
jest.mock('@salesforce/apex/ScRequestController.submitRequest', () => jest.fn(), { virtual: true });

describe('scRequestModal', () => {
    afterEach(() => { while (document.body.firstChild) document.body.removeChild(document.body.firstChild); });

    it('renders step 1 on load', async () => {
        getAssignedSC.mockResolvedValue({
            assignedSCId: '005xx000001X1AAAZ',
            hasMultiplePairings: false,
            allSCs: [{ id: '005xx000001X1AAAZ', name: 'Joe Varvel' }]
        });

        const el = createElement('c-sc-request-modal', { is: scRequestModal });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        await Promise.resolve();

        const step1 = el.shadowRoot.querySelector('[data-step="1"]');
        expect(step1).not.toBeNull();
    });

    it('shows multiple pairings warning when hasMultiplePairings is true', async () => {
        getAssignedSC.mockResolvedValue({
            assignedSCId: '005xx000001X1AAAZ',
            hasMultiplePairings: true,
            allSCs: [{ id: '005xx000001X1AAAZ', name: 'Joe Varvel' }]
        });

        const el = createElement('c-sc-request-modal', { is: scRequestModal });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        await Promise.resolve();

        const warning = el.shadowRoot.querySelector('[data-warning="multiple-pairings"]');
        expect(warning).not.toBeNull();
    });

    it('calendar invite checkbox is checked by default on step 3', async () => {
        getAssignedSC.mockResolvedValue({
            assignedSCId: '005xx000001X1AAAZ',
            hasMultiplePairings: false,
            allSCs: [{ id: '005xx000001X1AAAZ', name: 'Joe Varvel' }]
        });
        getAvailableSlots.mockResolvedValue([]);

        const el = createElement('c-sc-request-modal', { is: scRequestModal });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        await Promise.resolve();

        // Simulate selecting a manual date so step 1 is valid, then click Next twice
        const dateInput = el.shadowRoot.querySelector('lightning-input[type="datetime-local"]');
        if (dateInput) dateInput.dispatchEvent(new CustomEvent('change', { detail: { value: '2026-04-01T10:00' } }));
        el.shadowRoot.querySelector('[label="Next"]').click();
        await Promise.resolve();

        // Set required step 2 fields
        el.shadowRoot.querySelectorAll('lightning-combobox').forEach((combo, i) => {
            combo.dispatchEvent(new CustomEvent('change', { detail: { value: i === 0 ? 'Demo Request' : 'GoToConnect' } }));
        });
        el.shadowRoot.querySelector('[label="Next"]').click();
        await Promise.resolve();

        const checkbox = el.shadowRoot.querySelector('[data-field="sendCalendarInvite"]');
        expect(checkbox).not.toBeNull();
        expect(checkbox.checked).toBe(true);
    });
});
```

- [ ] **Step 2: Run Jest to verify tests fail**

```bash
cd force-app/main/default/lwc/scRequestModal && npx jest --no-coverage 2>&1 | head -20
```

Expected: FAIL — component doesn't exist yet.

- [ ] **Step 3: Create the LWC HTML template**

Create `force-app/main/default/lwc/scRequestModal/scRequestModal.html`:

```html
<template>
    <lightning-modal size="medium" onclose={handleClose}>
        <div slot="header">
            <h2 class="modal-title">Request a Solutions Consultant</h2>
            <div class="step-indicator">
                <span class={step1Class}>1 · SC &amp; Time</span>
                <span class="step-sep">›</span>
                <span class={step2Class}>2 · Details</span>
                <span class="step-sep">›</span>
                <span class={step3Class}>3 · Review</span>
            </div>
        </div>

        <!-- Step 1: SC Selection + Calendar -->
        <template lwc:if={isStep1}>
            <div data-step="1">
                <template lwc:if={hasMultiplePairings}>
                    <div class="warning-banner" data-warning="multiple-pairings">
                        ⚠️ Multiple active SC pairings found. Please verify the SC below is correct.
                    </div>
                </template>

                <div class="field-group">
                    <label class="field-label">Solutions Consultant</label>
                    <lightning-combobox
                        name="sc"
                        value={selectedSCId}
                        options={scOptions}
                        onchange={handleSCChange}
                        required>
                    </lightning-combobox>
                </div>

                <div class="field-group">
                    <label class="field-label">Meeting Duration</label>
                    <lightning-radio-group
                        name="duration"
                        options={durationOptions}
                        value={selectedDuration}
                        type="button"
                        onchange={handleDurationChange}>
                    </lightning-radio-group>
                </div>

                <div class="calendar-panel">
                    <label class="field-label">Available Times</label>
                    <template lwc:if={isLoadingSlots}>
                        <lightning-spinner alternative-text="Loading availability..." size="small"></lightning-spinner>
                    </template>
                    <template lwc:if={hasSlots}>
                        <div class="slot-list">
                            <template for:each={availableSlots} for:item="slot">
                                <button
                                    key={slot.label}
                                    class={slot.cssClass}
                                    data-start={slot.startTime}
                                    onclick={handleSlotSelect}>
                                    {slot.label}
                                </button>
                            </template>
                        </div>
                    </template>
                    <template lwc:if={noSlotsFound}>
                        <p class="no-slots-msg">No mutual availability found in the next 5 days.</p>
                    </template>
                    <template lwc:if={calendarError}>
                        <p class="error-msg">Calendar availability is currently unavailable.</p>
                    </template>

                    <div class="manual-date-toggle">
                        <button class="link-btn" onclick={toggleManualDate}>
                            {manualDateToggleLabel}
                        </button>
                    </div>
                    <template lwc:if={showManualDate}>
                        <lightning-input
                            type="datetime-local"
                            label="Enter date and time manually"
                            value={manualDateTime}
                            onchange={handleManualDateChange}>
                        </lightning-input>
                    </template>
                </div>
            </div>
        </template>

        <!-- Step 2: Request Details -->
        <template lwc:if={isStep2}>
            <div data-step="2">
                <div class="field-group">
                    <lightning-combobox
                        name="requestType"
                        label="Request Type"
                        value={requestType}
                        options={requestTypeOptions}
                        onchange={handleRequestTypeChange}
                        required>
                    </lightning-combobox>
                </div>
                <div class="field-group">
                    <lightning-combobox
                        name="productLine"
                        label="Product Line"
                        value={productLine}
                        options={productLineOptions}
                        onchange={handleProductLineChange}
                        required>
                    </lightning-combobox>
                </div>
                <div class="field-group">
                    <lightning-textarea
                        name="requestNotes"
                        label="What does the customer want to achieve / solve?"
                        placeholder="Describe the customer's current state, what they're trying to solve, and what the demo needs to include..."
                        value={requestNotes}
                        onchange={handleNotesChange}>
                    </lightning-textarea>
                </div>
            </div>
        </template>

        <!-- Step 3: Review & Submit -->
        <template lwc:if={isStep3}>
            <div data-step="3">
                <div class="review-grid">
                    <div class="review-item">
                        <span class="review-label">Solutions Consultant</span>
                        <span class="review-value">{selectedSCName}</span>
                    </div>
                    <div class="review-item">
                        <span class="review-label">Meeting Time</span>
                        <span class="review-value">{formattedMeetingDate}</span>
                    </div>
                    <div class="review-item">
                        <span class="review-label">Request Type</span>
                        <span class="review-value">{requestType}</span>
                    </div>
                    <div class="review-item">
                        <span class="review-label">Product Line</span>
                        <span class="review-value">{productLine}</span>
                    </div>
                </div>
                <div class="review-notes">
                    <span class="review-label">Notes</span>
                    <p class="review-notes-text">{requestNotes}</p>
                </div>
                <div class="calendar-invite-toggle">
                    <lightning-input
                        type="checkbox"
                        label="Send Outlook calendar invite to SC automatically"
                        data-field="sendCalendarInvite"
                        checked={sendCalendarInvite}
                        onchange={handleInviteToggle}>
                    </lightning-input>
                </div>
                <template lwc:if={submitError}>
                    <div class="error-banner">{submitError}</div>
                </template>
            </div>
        </template>

        <!-- Footer -->
        <div slot="footer">
            <template lwc:if={isStep1}>
                <lightning-button label="Cancel" onclick={handleClose}></lightning-button>
                <lightning-button variant="brand" label="Next" onclick={goToStep2} disabled={step1Invalid}></lightning-button>
            </template>
            <template lwc:if={isStep2}>
                <lightning-button label="Back" onclick={goToStep1}></lightning-button>
                <lightning-button variant="brand" label="Next" onclick={goToStep3} disabled={step2Invalid}></lightning-button>
            </template>
            <template lwc:if={isStep3}>
                <lightning-button label="Back" onclick={goToStep2}></lightning-button>
                <lightning-button variant="brand" label="Submit Request" onclick={handleSubmit} disabled={isSubmitting}></lightning-button>
            </template>
        </div>
    </lightning-modal>
</template>
```

- [ ] **Step 4: Create the LWC JavaScript controller**

Create `force-app/main/default/lwc/scRequestModal/scRequestModal.js`:

```javascript
import { LightningElement, api, track, wire } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { getRecord } from 'lightning/uiRecordApi';
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';
import USER_ID from '@salesforce/user/Id';
import getAssignedSC from '@salesforce/apex/ScRequestController.getAssignedSC';
import getAvailableSlots from '@salesforce/apex/M365CalendarService.getAvailableSlots';
import submitRequest from '@salesforce/apex/ScRequestController.submitRequest';

const REQUEST_TYPES = [
    'Biz Dev/Channel','CSM Activity','Demo Request','Discovery Call',
    'Document Request','On-Site Meeting','Pro Serv Scoping','RFX/Tender',
    'Security Questionnaire','Security Team Escalation','Strategic Planning',
    'Technical Call','Training','Trial Engagement/POC'
].map(v => ({ label: v, value: v }));

const PRODUCT_LINES = [
    'Central','EnhancedAudio','GoTo Resolve','GoToAssist','GoToConnect',
    'GoToMeeting','GoToMyPC','GoToRoom','GoToTraining','GoToWebcast',
    'GoToWebinar','ITSG','MDM','None','Pro','Rescue','UCC'
].map(v => ({ label: v, value: v }));

export default class ScRequestModal extends NavigationMixin(LightningElement) {
    @api recordId;
    @track currentStep = 1;
    @track selectedSCId;
    @track selectedSCName;
    @track hasMultiplePairings = false;
    @track scOptions = [];
    @track selectedDuration = '60';
    @track availableSlots = [];
    @track isLoadingSlots = false;
    @track calendarError = false;
    @track selectedSlot;
    @track showManualDate = false;
    @track manualDateTime;
    @track requestType;
    @track productLine;
    @track requestNotes = '';
    @track sendCalendarInvite = true;
    @track isSubmitting = false;
    @track submitError;

    requestTypeOptions = REQUEST_TYPES;
    productLineOptions = PRODUCT_LINES;
    durationOptions = [
        { label: '30 min', value: '30' },
        { label: '1 hour', value: '60' },
        { label: '90 min', value: '90' }
    ];

    connectedCallback() {
        getAssignedSC()
            .then(result => {
                this.selectedSCId = result.assignedSCId;
                this.hasMultiplePairings = result.hasMultiplePairings;
                this.scOptions = result.allSCs.map(sc => ({ label: sc.name, value: sc.id }));
                if (this.selectedSCId) {
                    const match = result.allSCs.find(s => s.id === this.selectedSCId);
                    this.selectedSCName = match ? match.name : '';
                    this.loadSlots();
                }
            })
            .catch(() => {});
    }

    loadSlots() {
        if (!this.selectedSCId) return;
        this.isLoadingSlots = true;
        this.calendarError = false;
        this.availableSlots = [];

        const now = new Date();
        const fiveDaysOut = new Date(now.getTime() + 5 * 24 * 60 * 60 * 1000);

        getAvailableSlots({
            aeUserId: USER_ID,
            scUserId: this.selectedSCId,
            rangeStart: now.toISOString(),
            rangeEnd: fiveDaysOut.toISOString(),
            durationMinutes: parseInt(this.selectedDuration)
        })
        .then(slots => {
            this.availableSlots = slots.map(s => ({
                ...s,
                cssClass: 'slot-btn' + (this.selectedSlot === s.startTime ? ' selected' : '')
            }));
        })
        .catch(() => { this.calendarError = true; })
        .finally(() => { this.isLoadingSlots = false; });
    }

    handleSCChange(e) {
        this.selectedSCId = e.detail.value;
        const match = this.scOptions.find(o => o.value === this.selectedSCId);
        this.selectedSCName = match ? match.label : '';
        this.selectedSlot = null;
        this.loadSlots();
    }

    handleDurationChange(e) {
        this.selectedDuration = e.detail.value;
        this.loadSlots();
    }

    handleSlotSelect(e) {
        this.selectedSlot = e.currentTarget.dataset.start;
        this.manualDateTime = null;
        this.availableSlots = this.availableSlots.map(s => ({
            ...s,
            cssClass: 'slot-btn' + (s.startTime === this.selectedSlot ? ' selected' : '')
        }));
    }

    toggleManualDate() { this.showManualDate = !this.showManualDate; }
    get manualDateToggleLabel() { return this.showManualDate ? 'Use suggested times' : 'Enter a time manually'; }

    handleManualDateChange(e) {
        this.manualDateTime = e.detail.value;
        this.selectedSlot = null;
    }

    handleRequestTypeChange(e) { this.requestType = e.detail.value; }
    handleProductLineChange(e) { this.productLine = e.detail.value; }
    handleNotesChange(e) { this.requestNotes = e.detail.value; }
    handleInviteToggle(e) { this.sendCalendarInvite = e.detail.checked; }

    get isStep1() { return this.currentStep === 1; }
    get isStep2() { return this.currentStep === 2; }
    get isStep3() { return this.currentStep === 3; }
    get step1Class() { return 'step-pill' + (this.currentStep === 1 ? ' active' : ''); }
    get step2Class() { return 'step-pill' + (this.currentStep === 2 ? ' active' : ''); }
    get step3Class() { return 'step-pill' + (this.currentStep === 3 ? ' active' : ''); }
    get step1Invalid() { return !this.selectedSCId || (!this.selectedSlot && !this.manualDateTime); }
    get step2Invalid() { return !this.requestType || !this.productLine; }
    get hasSlots() { return !this.isLoadingSlots && this.availableSlots.length > 0; }
    get noSlotsFound() { return !this.isLoadingSlots && !this.calendarError && this.availableSlots.length === 0 && !this.showManualDate; }

    get formattedMeetingDate() {
        const dt = this.selectedSlot || this.manualDateTime;
        if (!dt) return '';
        return new Date(dt).toLocaleString('en-US', { weekday: 'short', month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' });
    }

    goToStep1() { this.currentStep = 1; }
    goToStep2() { this.currentStep = 2; }
    goToStep3() { this.currentStep = 3; }

    handleSubmit() {
        this.isSubmitting = true;
        this.submitError = null;

        submitRequest({
            input: {
                opportunityId: this.recordId,
                scId: this.selectedSCId,
                requestType: this.requestType,
                productLine: this.productLine,
                requestNotes: this.requestNotes,
                meetingDate: this.selectedSlot || this.manualDateTime,
                durationMinutes: parseInt(this.selectedDuration),
                sendCalendarInvite: this.sendCalendarInvite
            }
        })
        .then(() => {
            this.dispatchEvent(new ShowToastEvent({ title: 'Success', message: 'SC Request submitted successfully.', variant: 'success' }));
            this.dispatchEvent(new CustomEvent('close'));
        })
        .catch(err => {
            this.submitError = err.body?.message || 'An error occurred. Please try again.';
        })
        .finally(() => { this.isSubmitting = false; });
    }

    handleClose() { this.dispatchEvent(new CustomEvent('close')); }
}
```

- [ ] **Step 5: Create the CSS**

Create `force-app/main/default/lwc/scRequestModal/scRequestModal.css`:

```css
.modal-title { font-size: 16px; font-weight: 600; margin-bottom: 8px; }
.step-indicator { display: flex; align-items: center; gap: 8px; font-size: 12px; color: #706e6b; }
.step-pill { padding: 2px 10px; border-radius: 12px; }
.step-pill.active { background: #0070d2; color: white; font-weight: 600; }
.step-sep { color: #c9c7c5; }
.field-group { margin-bottom: 16px; }
.field-label { font-size: 12px; font-weight: 600; color: #444; display: block; margin-bottom: 6px; }
.calendar-panel { margin-top: 8px; }
.slot-list { display: flex; flex-wrap: wrap; gap: 8px; margin: 8px 0; }
.slot-btn { padding: 8px 14px; border: 1px solid #dddbda; border-radius: 4px; background: white; cursor: pointer; font-size: 13px; transition: all 0.15s; }
.slot-btn:hover { border-color: #0070d2; color: #0070d2; }
.slot-btn.selected { background: #0070d2; color: white; border-color: #0070d2; }
.no-slots-msg, .error-msg { font-size: 13px; color: #706e6b; padding: 8px 0; }
.error-msg { color: #c23934; }
.manual-date-toggle { margin-top: 8px; }
.link-btn { background: none; border: none; color: #0070d2; font-size: 13px; cursor: pointer; padding: 0; }
.warning-banner { background: #fef3c7; border-left: 3px solid #d97706; padding: 10px 14px; font-size: 13px; border-radius: 0 4px 4px 0; margin-bottom: 16px; }
.error-banner { background: #fef2f2; border-left: 3px solid #c23934; padding: 10px 14px; font-size: 13px; border-radius: 0 4px 4px 0; margin-top: 12px; }
.review-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px 24px; background: #f4f6f9; border-radius: 8px; padding: 16px; margin-bottom: 16px; }
.review-label { font-size: 11px; text-transform: uppercase; color: #888; font-weight: 600; display: block; margin-bottom: 2px; }
.review-value { font-size: 14px; font-weight: 600; }
.review-notes { margin-bottom: 16px; }
.review-notes-text { font-size: 13px; color: #444; line-height: 1.6; background: #fffbf0; border-left: 3px solid #d97706; padding: 10px 14px; border-radius: 0 4px 4px 0; margin-top: 6px; }
.calendar-invite-toggle { margin-top: 8px; }
```

- [ ] **Step 6: Create the metadata file**

Create `force-app/main/default/lwc/scRequestModal/scRequestModal.js-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>false</isExposed>
</LightningComponentBundle>
```

- [ ] **Step 7: Run Jest tests and verify they pass**

```bash
cd force-app/main/default/lwc/scRequestModal && npx jest --no-coverage
```

Expected: All 3 tests PASS.

- [ ] **Step 8: Deploy LWC**

```bash
sf project deploy start --source-dir force-app/main/default/lwc/scRequestModal
```

- [ ] **Step 9: Commit**

```bash
git add force-app/
git commit -m "feat: add scRequestModal LWC with 3-step form, calendar, and submit"
```

---

### Task 7: Wire LWC to existing "Request SC" Quick Action

**Files:**
- Create: `force-app/main/default/lwc/scRequestAction/scRequestAction.js`
- Create: `force-app/main/default/lwc/scRequestAction/scRequestAction.html`
- Create: `force-app/main/default/lwc/scRequestAction/scRequestAction.js-meta.xml`
- Modify: Opportunity page layout (via Setup UI or metadata) to replace current Quick Action target

- [ ] **Step 1: Create wrapper LWC that launches the modal**

Create `force-app/main/default/lwc/scRequestAction/scRequestAction.html`:

```html
<template>
    <template lwc:if={isOpen}>
        <c-sc-request-modal record-id={recordId} onclose={handleClose}></c-sc-request-modal>
    </template>
</template>
```

Create `force-app/main/default/lwc/scRequestAction/scRequestAction.js`:

```javascript
import { LightningElement, api } from 'lwc';
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

export default class ScRequestAction extends LightningElement {
    @api recordId;
    @api invoke() { this.isOpen = true; }
    isOpen = false;
    handleClose() {
        this.isOpen = false;
        // Notify LDS to re-fetch the Opportunity record so updated fields reflect on-screen
        notifyRecordUpdateAvailable([{ recordId: this.recordId }]);
    }
}
```

Create `force-app/main/default/lwc/scRequestAction/scRequestAction.js-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordAction</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordAction">
            <actionType>Action</actionType>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

- [ ] **Step 2: Write Jest tests for scRequestAction**

Create `force-app/main/default/lwc/scRequestAction/__tests__/scRequestAction.test.js`:

```javascript
import { createElement } from 'lwc';
import scRequestAction from 'c/scRequestAction';
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

jest.mock('lightning/uiRecordApi', () => ({ notifyRecordUpdateAvailable: jest.fn() }), { virtual: true });

describe('scRequestAction', () => {
    afterEach(() => { while (document.body.firstChild) document.body.removeChild(document.body.firstChild); });

    it('does not render modal before invoke()', async () => {
        const el = createElement('c-sc-request-action', { is: scRequestAction });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        await Promise.resolve();
        expect(el.shadowRoot.querySelector('c-sc-request-modal')).toBeNull();
    });

    it('renders modal after invoke()', async () => {
        const el = createElement('c-sc-request-action', { is: scRequestAction });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        el.invoke();
        await Promise.resolve();
        expect(el.shadowRoot.querySelector('c-sc-request-modal')).not.toBeNull();
    });

    it('closes modal and calls notifyRecordUpdateAvailable on close event', async () => {
        const el = createElement('c-sc-request-action', { is: scRequestAction });
        el.recordId = '006xx000001234AAA';
        document.body.appendChild(el);
        el.invoke();
        await Promise.resolve();

        el.shadowRoot.querySelector('c-sc-request-modal').dispatchEvent(new CustomEvent('close'));
        await Promise.resolve();

        expect(el.shadowRoot.querySelector('c-sc-request-modal')).toBeNull();
        expect(notifyRecordUpdateAvailable).toHaveBeenCalledWith([{ recordId: '006xx000001234AAA' }]);
    });
});
```

- [ ] **Step 3: Run Jest tests for scRequestAction**

```bash
cd force-app/main/default/lwc/scRequestAction && npx jest --no-coverage
```

Expected: All 3 tests PASS.

- [ ] **Step 4: Deploy wrapper**

```bash
sf project deploy start --source-dir force-app/main/default/lwc/scRequestAction
```

- [ ] **Step 5: Replace Quick Action target in Salesforce Setup**

In Salesforce Setup → Object Manager → Opportunity → Buttons, Links, and Actions → find "Request SC" → Edit → change Action Type to "LWC" → select `scRequestAction` → Save.

- [ ] **Step 6: Verify in scratch org**

Open an Opportunity record → click the action bar overflow `▼` → click "Request SC" → verify the modal opens with Step 1 showing SC pre-populated.

- [ ] **Step 7: Commit**

```bash
git add force-app/
git commit -m "feat: wire scRequestAction LWC to existing Request SC quick action"
```

---

## Chunk 4: Email Flow, Named Credential & Layout

### Task 8: Create Record-Triggered Flow for email notifications

**Files:**
- Create: `force-app/main/default/flows/SC_Request_Email_Notification.flow-meta.xml`

- [ ] **Step 1: Create HTML email template in Salesforce Setup**

In Salesforce Setup → Classic Email Templates → New Template → HTML with Letterhead (or Custom HTML):
- Folder: SC Request Templates (create folder first if needed)
- Name: `SC Request Notification`
- Subject: `New SC Request: {!SC_Request__c.Opportunity__r.Account.Name} — {!SC_Request__c.Request_Type__c}`

Paste the following HTML as the template body:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<style>
  body { font-family: Arial, sans-serif; color: #1a1a2e; margin: 0; padding: 0; }
  .header { background: #0070d2; padding: 16px 24px; display: flex; align-items: center; gap: 12px; }
  .header-badge { background: #fff; color: #0070d2; font-size: 11px; font-weight: 700; padding: 3px 10px; border-radius: 4px; }
  .header-title { color: #fff; font-size: 15px; font-weight: 600; }
  .body { padding: 24px; }
  .meta { font-size: 13px; color: #6b7280; margin-bottom: 20px; }
  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px 24px; background: #f4f6f9; border-radius: 8px; padding: 16px 20px; margin-bottom: 20px; }
  .field-label { font-size: 11px; text-transform: uppercase; color: #888; font-weight: 600; margin-bottom: 3px; }
  .field-value { font-size: 13px; font-weight: 600; }
  .field-value.link { color: #0070d2; }
  .field-value.date { color: #c23934; }
  .notes-label { font-size: 11px; text-transform: uppercase; color: #888; font-weight: 600; margin-bottom: 8px; }
  .notes-body { background: #fffbf0; border-left: 3px solid #d97706; padding: 12px 16px; border-radius: 0 6px 6px 0; font-size: 13px; line-height: 1.6; color: #333; margin-bottom: 20px; }
  .actions { display: flex; gap: 10px; margin-bottom: 20px; flex-wrap: wrap; }
  .btn { padding: 9px 16px; border-radius: 5px; font-size: 12px; font-weight: 600; text-decoration: none; display: inline-block; }
  .btn-primary { background: #0070d2; color: #fff; }
  .btn-secondary { background: #fff; color: #0070d2; border: 1.5px solid #0070d2; }
  .footer { border-top: 1px solid #e5e7eb; padding-top: 14px; font-size: 12px; color: #888; }
</style>
</head>
<body>
  <div class="header">
    <span class="header-badge">SC REQUEST</span>
    <span class="header-title">A New SC Request Has Been Submitted</span>
  </div>
  <div class="body">
    <p class="meta">Submitted by <strong>{!SC_Request__c.Submitted_By__r.Name}</strong> on <strong>{!SC_Request__c.Submitted_Date__c}</strong></p>
    <div class="grid">
      <div>
        <div class="field-label">Opportunity</div>
        <div class="field-value link">{!SC_Request__c.Opportunity__r.Name}</div>
      </div>
      <div>
        <div class="field-label">Primary Contact</div>
        <div class="field-value">{!SC_Request__c.Opportunity__r.Primary_Contact__r.Name}</div>
      </div>
      <div>
        <div class="field-label">Request Type</div>
        <div class="field-value">{!SC_Request__c.Request_Type__c}</div>
      </div>
      <div>
        <div class="field-label">Product Line</div>
        <div class="field-value">{!SC_Request__c.Product_Line__c}</div>
      </div>
      <div>
        <div class="field-label">Assigned SC</div>
        <div class="field-value">{!SC_Request__c.Requested_SC__r.Name}</div>
      </div>
      <div>
        <div class="field-label">Meeting Date &amp; Time</div>
        <div class="field-value date">{!SC_Request__c.Meeting_Date__c}</div>
      </div>
    </div>
    <div class="notes-label">What does the customer want to achieve / solve?</div>
    <div class="notes-body">{!SC_Request__c.Request_Notes__c}</div>
    <div class="actions">
      <a href="{!URLFOR($Site.BaseUrl, SC_Request__c.Opportunity__r.Id)}" class="btn btn-primary">View Opportunity →</a>
      <a href="{!URLFOR($Site.BaseUrl, SC_Request__c.Id)}" class="btn btn-secondary">View SC Request →</a>
    </div>
    <div class="footer">
      Sent to: <strong>{!SC_Request__c.Requested_SC__r.Name}</strong> (SC) ·
      <strong>{!SC_Request__c.Submitted_By__r.Name}</strong> (AE) ·
      <strong>{!SC_Request__c.Submitted_By__r.Manager.Name}</strong> (AE Manager) ·
      <strong>{!SC_Request__c.Requested_SC__r.Manager.Name}</strong> (SC Manager)
    </div>
  </div>
</body>
</html>
```

> **Note:** `Primary_Contact__r` uses whatever the field API name is in your org for the Opportunity's primary contact lookup. Confirm with your Salesforce admin — common values are `Primary_Contact__c` or the standard `ContactId` via a related junction. Adjust the merge field if needed.

Note the Template ID after saving — you'll need it in the Flow.

- [ ] **Step 2: Create the Record-Triggered Flow**

In Salesforce Setup → Flows → New Flow → Record-Triggered Flow:
- Object: SC Request (`SC_Request__c`)
- Trigger: A record is created
- Optimize for: Actions and Related Records

Add 4 Send Email actions (one per recipient), each using the HTML template created in Step 1:

| Action | Recipient field |
|--------|----------------|
| Email to AE | `{!$Record.Opportunity__r.Owner.Email}` |
| Email to SC | `{!$Record.Requested_SC__r.Email}` |
| Email to AE Manager | `{!$Record.Opportunity__r.Owner.Manager.Email}` |
| Email to SC Manager | `{!$Record.Requested_SC__r.Manager.Email}` |

Save and Activate the Flow.

- [ ] **Step 3: Export Flow metadata for source control**

```bash
sf project retrieve start --metadata "Flow:SC_Request_Email_Notification"
```

- [ ] **Step 4: Verify email fires**

In scratch org, manually insert an `SC_Request__c` record via Developer Console:

```apex
SC_Request__c r = new SC_Request__c(
    Opportunity__c = '<any_opp_id>',
    Requested_SC__c = '<any_user_id>',
    Request_Type__c = 'Demo Request',
    Product_Line__c = 'GoToConnect',
    Request_Notes__c = 'Test',
    Meeting_Date__c = Datetime.now().addDays(3)
);
insert r;
```

Check email logs: Setup → Email Log Files → verify 4 emails sent.

- [ ] **Step 5: Commit**

```bash
git add force-app/
git commit -m "feat: add SC_Request_Email_Notification record-triggered flow"
```

---

### Task 9: Configure Named Credential for M365 Graph API

**Files:**
- Create: `force-app/main/default/namedCredentials/M365_Graph_API.namedCredential-meta.xml`
- Create: `force-app/main/default/authproviders/M365_OAuth.authprovider-meta.xml`

- [ ] **Step 1: Register Salesforce Connected App in Azure AD (IT step)**

Coordinate with IT to:
1. Register a new app in Azure AD App Registrations (or extend existing SSO app)
2. Grant API permissions: `Calendars.Read`, `Calendars.ReadWrite` (delegated)
3. Note the Client ID and Client Secret
4. Add Salesforce callback URL: `https://<your-org>.salesforce.com/services/authcallback/M365_OAuth`

- [ ] **Step 2: Create Auth Provider in Salesforce**

In Salesforce Setup → Auth. Providers → New:
- Provider Type: Microsoft Access Control Service (or OpenID Connect)
- Name: `M365_OAuth`
- Client ID: (from Azure AD step above)
- Client Secret: (from Azure AD step above)
- Authorize Endpoint: `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize`
- Token Endpoint: `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`
- Default Scopes: `Calendars.Read Calendars.ReadWrite offline_access`

- [ ] **Step 3: Create Named Credential**

Create `force-app/main/default/namedCredentials/M365_Graph_API.namedCredential-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<NamedCredential xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>M365_Graph_API</fullName>
    <label>M365 Graph API</label>
    <endpoint>https://graph.microsoft.com</endpoint>
    <principalType>NamedUser</principalType>
    <protocol>Oauth</protocol>
    <oauthProvider>M365_OAuth</oauthProvider>
    <generateAuthorizationHeader>true</generateAuthorizationHeader>
    <allowMergeFieldsInHeader>false</allowMergeFieldsInHeader>
    <allowMergeFieldsInBody>false</allowMergeFieldsInBody>
</NamedCredential>
```

- [ ] **Step 4: Deploy Named Credential**

```bash
sf project deploy start --source-dir force-app/main/default/namedCredentials
```

- [ ] **Step 5: Authorize users via OAuth — IMPORTANT: Per-user auth required**

The Named Credential uses `principalType: NamedUser`, meaning **every AE must individually authorize their M365 account** before the calendar integration will work for them. This is a one-time setup per user.

**For each AE:** In Salesforce → top-right avatar → Settings → Authentication Settings for External Systems → find `M365_Graph_API` → click "Authenticate". This opens the M365 OAuth flow. The AE logs in with their GoTo M365 credentials and grants access.

**Rollout plan:** IT or Salesforce admin should send instructions to all AEs before go-live. If an AE has not authenticated, the LWC gracefully falls back to the manual date entry — the form is never blocked.

First, authorize at least one test user to verify the full OAuth flow works end-to-end.

- [ ] **Step 6: Test calendar API call**

In Developer Console → Execute Anonymous:

```apex
List<M365CalendarService.TimeSlot> slots = M365CalendarService.getAvailableSlots(
    UserInfo.getUserId(), UserInfo.getUserId(),
    Datetime.now(), Datetime.now().addDays(5), 60
);
System.debug('Slots found: ' + slots.size());
for (M365CalendarService.TimeSlot s : slots) {
    System.debug(s.label);
}
```

Expected: Debug log shows 0-5 slots with formatted labels.

- [ ] **Step 7: Commit**

```bash
git add force-app/
git commit -m "feat: add M365_Graph_API named credential configuration"
```

---

### Task 10: Update Opportunity page layout and add related list

**Files:**
- Modify: Opportunity page layout (via Setup or metadata)

- [ ] **Step 1: Add `SC_Request_Notes__c` to Sales Engineering Info section**

In Salesforce Setup → Object Manager → Opportunity → Page Layouts → open your org's Opportunity layout → drag `SC Request Notes` field into the Sales Engineering Info section → Save.

- [ ] **Step 2: Add SC Requests related list**

In the same page layout editor → Related Lists tab → drag `SC Requests` (from `SC_Request__c`) onto the Related Lists section → configure columns: Request Name, Request Type, Product Line, Meeting Date, Requested SC, Status → Save.

- [ ] **Step 3: Retrieve layout metadata for source control**

```bash
sf project retrieve start --metadata "Layout:Opportunity-Opportunity Layout"
```

- [ ] **Step 4: Verify layout in scratch org**

Open an Opportunity → confirm:
- `SC Request Notes` field appears in Sales Engineering Info section
- SC Requests related list appears in Related tab
- "Request SC" button is in the action bar overflow and launches the LWC modal

- [ ] **Step 5: Run all Apex tests one final time**

```bash
sf apex run test --class-names ScRequestControllerTest,M365CalendarServiceTest --result-format human --synchronous
```

Expected: All 7 tests PASS.

- [ ] **Step 6: Final commit**

```bash
git add force-app/
git commit -m "feat: update Opportunity layout — SC Request Notes field and related list"
git push
```
