# 🎯 ServiceNow ITSM Implementation Guide

> A comprehensive implementation guide, configuration patterns, and automation blueprints for ServiceNow IT Service Management (ITSM) — covering Incident, Problem, Change, Service Catalog, Knowledge Management, and SLA Design — built from 7+ years of enterprise delivery.

<p>
  <img src="https://img.shields.io/badge/ServiceNow-ITSM-brightgreen?style=for-the-badge&logo=servicenow&logoColor=white"/>
  <img src="https://img.shields.io/badge/ITIL-v4%20Aligned-1c2b3a?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Author-Swarup%20Kumar%20Namana-b8922a?style=for-the-badge"/>
</p>

---

## 📌 Overview

This guide documents battle-tested ITSM implementation patterns, scripting examples, SLA design frameworks, workflow configurations, and reporting strategies for ServiceNow ITSM deployments. All patterns follow ITIL v4 best practices and are designed for enterprise-scale environments with thousands of users.

**Key outcomes this guide delivers:**
- ✅ Streamlined Incident → Problem → Change lifecycle
- ✅ Automated ticket routing reducing manual assignment by 80%+
- ✅ SLA breach prevention through intelligent escalation
- ✅ Self-service adoption increase through optimized Service Catalog
- ✅ Knowledge deflection reducing repeat incidents by 35%+
- ✅ Executive-ready dashboards with real-time service health visibility

---

## 📂 Repository Structure

```
servicenow-itsm-implementation-guide/
│
├── 01-incident-management/
│   ├── incident-lifecycle.md              # End-to-end incident process
│   ├── auto-assignment-rules.md           # Intelligent routing patterns
│   ├── escalation-workflow.md             # SLA breach escalation design
│   ├── major-incident-process.md          # P1/P2 major incident handling
│   └── incident-scripts.md                # Business Rules & Client Scripts
│
├── 02-problem-management/
│   ├── problem-lifecycle.md               # Problem → RCA → Known Error flow
│   ├── problem-detection.md               # Auto-detect problems from incidents
│   ├── rca-template.md                    # Root Cause Analysis template
│   └── known-error-database.md            # KEDB design & management
│
├── 03-change-management/
│   ├── change-types.md                    # Standard / Normal / Emergency
│   ├── change-approval-design.md          # CAB & approval group setup
│   ├── change-risk-assessment.md          # Auto risk scoring for changes
│   ├── change-calendar.md                 # Change freeze & blackout windows
│   └── post-implementation-review.md      # PIR automation
│
├── 04-service-catalog/
│   ├── catalog-design-principles.md       # Catalog item design best practices
│   ├── variable-sets.md                   # Reusable variable set patterns
│   ├── order-guides.md                    # Order guide design
│   ├── fulfillment-automation.md          # Fulfillment workflow patterns
│   └── catalog-governance.md             # Catalog lifecycle & ownership
│
├── 05-knowledge-management/
│   ├── knowledge-base-design.md           # KB structure & taxonomy
│   ├── article-lifecycle.md               # Draft → Review → Publish flow
│   ├── knowledge-deflection.md            # Incident deflection setup
│   └── ai-search-optimization.md          # Search optimization patterns
│
├── 06-sla-design/
│   ├── sla-framework.md                   # SLA design methodology
│   ├── ola-uc-design.md                   # OLA & UC configuration
│   ├── sla-scripts.md                     # SLA condition & duration scripts
│   └── breach-prevention.md               # Proactive SLA breach prevention
│
├── 07-reporting/
│   ├── pa-dashboards.md                   # Performance Analytics setup
│   ├── executive-scorecard.md             # Executive KPI scorecard
│   └── kpi-definitions.md                 # ITSM KPI definitions & targets
│
└── 08-best-practices/
    ├── naming-conventions.md              # Field & group naming standards
    ├── acl-design.md                      # ACL security patterns
    ├── update-set-management.md           # Update set discipline
    └── upgrade-readiness.md               # Pre/post upgrade checklist
```

---

## 🔥 Module 1: Incident Management

### Incident Lifecycle

```
User reports issue
      │
      ▼
Incident Created (Portal / Email / Phone / Monitoring)
      │
      ▼
Auto-Classification
  ├── Category / Subcategory assigned
  ├── Priority calculated (Impact × Urgency)
  └── Assignment Group auto-populated
      │
      ▼
SLA Clock Starts
      │
      ├── Acknowledgement SLA (15 min for P1)
      ├── Resolution SLA (4h P1 / 8h P2 / 24h P3 / 72h P4)
      │
      ▼
Agent Investigation
  ├── Knowledge search → Article found? → Apply fix → Resolve
  └── No article → Investigate → Fix → Create KB Article
      │
      ▼
Resolved → User confirmation (wait 3 days)
      │
  ├── User confirms → Close
  └── No response → Auto-close
      │
      ▼
Closed → SLA stops → Metrics captured
```

### Auto-Assignment Business Rule

```javascript
// Business Rule: Auto-assign incidents based on category
// Table: incident | When: before | Insert
// Name: "ITSM - Auto Assignment on Insert"

(function executeRule(current, previous) {

    // Assignment matrix — map category+subcategory to group
    var assignmentMatrix = {
        'hardware.laptop':          'Desktop Support',
        'hardware.desktop':         'Desktop Support',
        'hardware.printer':         'Desktop Support',
        'software.email':           'Email Support',
        'software.office':          'Desktop Support',
        'software.erp':             'ERP Support Team',
        'network.connectivity':     'Network Operations',
        'network.vpn':              'Network Operations',
        'security.access':          'Identity & Access Management',
        'security.virus':           'Cybersecurity Team',
        'server.performance':       'Server Operations',
        'server.availability':      'Server Operations',
        'database.performance':     'Database Administration',
        'application.servicenow':   'ServiceNow Support'
    };

    var key = current.category + '.' + current.subcategory;
    var groupName = assignmentMatrix[key];

    if (groupName) {
        var gr = new GlideRecord('sys_user_group');
        gr.addQuery('name', groupName);
        gr.setLimit(1);
        gr.query();

        if (gr.next()) {
            current.assignment_group = gr.sys_id;
            gs.info('ITSM AutoAssign: Incident ' + current.number +
                    ' assigned to ' + groupName);
        }
    }

    // Auto-set priority from impact + urgency
    if (current.impact && current.urgency) {
        current.priority = _calculatePriority(
            parseInt(current.impact),
            parseInt(current.urgency)
        );
    }

    function _calculatePriority(impact, urgency) {
        // Priority Matrix: 1=Critical, 2=High, 3=Medium, 4=Low
        var matrix = {
            '1_1': '1', '1_2': '2', '1_3': '3',
            '2_1': '2', '2_2': '3', '2_3': '4',
            '3_1': '3', '3_2': '4', '3_3': '4'
        };
        return matrix[impact + '_' + urgency] || '3';
    }

})(current, previous);
```

### Major Incident (P1) Auto-Escalation

```javascript
// Scheduled Job: Check P1 incidents not acknowledged within 15 min
// Name: "ITSM - P1 Acknowledgement Check"
// Runs: Every 5 minutes

var gr = new GlideRecord('incident');
gr.addQuery('priority', '1');
gr.addQuery('state', 'IN', '1,2'); // New or In Progress
gr.addQuery('u_acknowledged', false);

// Created more than 15 minutes ago
var cutoff = new GlideDateTime();
cutoff.addSeconds(-900); // 15 minutes
gr.addQuery('sys_created_on', '<', cutoff);
gr.query();

while (gr.next()) {
    // Escalate to IT Manager
    var email = new GlideEmailOutbound();
    email.setTo(getITManagerEmail());
    email.setSubject('[URGENT P1] Unacknowledged Major Incident: ' + gr.number);
    email.setBody(
        'P1 Incident ' + gr.number + ' has NOT been acknowledged within 15 minutes.\n\n' +
        'Description: ' + gr.short_description + '\n' +
        'Assigned Group: ' + gr.assignment_group.getDisplayValue() + '\n' +
        'Created: ' + gr.sys_created_on.getDisplayValue() + '\n\n' +
        'Immediate action required.'
    );
    email.save();

    // Create escalation task
    var task = new GlideRecord('task');
    task.initialize();
    task.short_description = 'ESCALATION: P1 Not Acknowledged - ' + gr.number;
    task.priority = '1';
    task.assignment_group = getManagerGroupId();
    task.parent = gr.sys_id;
    task.insert();

    gs.info('P1 Escalation triggered for: ' + gr.number);
}

function getITManagerEmail() {
    var group = new GlideRecord('sys_user_group');
    group.addQuery('name', 'IT Management');
    group.setLimit(1);
    group.query();
    return group.next() ? group.email.toString() : '';
}

function getManagerGroupId() {
    var group = new GlideRecord('sys_user_group');
    group.addQuery('name', 'IT Management');
    group.setLimit(1);
    group.query();
    return group.next() ? group.sys_id : '';
}
```

---

## 🔍 Module 2: Problem Management

### Problem Lifecycle

```
Trigger Sources:
  ├── Manual: Agent raises problem from recurring incidents
  ├── Auto:   3+ incidents same category/CI in 7 days
  └── Proactive: Trend analysis from Performance Analytics
        │
        ▼
Problem Record Created
        │
        ▼
Root Cause Analysis (RCA)
  ├── Assign to Problem Manager
  ├── Link related incidents
  ├── 5-Why / Fishbone analysis documented
  └── Workaround identified → Update all linked incidents
        │
        ▼
Fix identified?
  ├── Yes → Create Change Request → Implement fix
  └── No  → Log Known Error → Add to KEDB
        │
        ▼
Change implemented → Problem resolved → Close
  └── Auto-resolve linked incidents if still open
```

### Auto-Detect Recurring Incidents → Create Problem

```javascript
// Script Include: ProblemDetector
// Called by Scheduled Job (Daily)

var ProblemDetector = Class.create();
ProblemDetector.prototype = {
    initialize: function() {
        this.threshold = 3;   // Min incidents to trigger problem
        this.lookbackDays = 7; // Days to look back
    },

    detect: function() {
        var cutoff = new GlideDateTime();
        cutoff.addDaysLocalTime(-this.lookbackDays);

        // Group incidents by category + subcategory + CI
        var ga = new GlideAggregate('incident');
        ga.addQuery('state', 'NOT IN', '6,7'); // Not resolved or closed
        ga.addQuery('sys_created_on', '>=', cutoff);
        ga.addNotNullQuery('category');
        ga.groupBy('category');
        ga.groupBy('subcategory');
        ga.groupBy('cmdb_ci');
        ga.addAggregate('COUNT');
        ga.addHaving('COUNT', '>=', this.threshold);
        ga.query();

        while (ga.next()) {
            var count    = ga.getAggregate('COUNT');
            var category = ga.category.toString();
            var subcat   = ga.subcategory.toString();
            var ci       = ga.cmdb_ci.toString();

            // Check if problem already exists for this pattern
            if (!this._problemExists(category, subcat, ci)) {
                this._createProblem(category, subcat, ci, count);
            }
        }
    },

    _problemExists: function(category, subcat, ci) {
        var prob = new GlideRecord('problem');
        prob.addQuery('category', category);
        prob.addQuery('subcategory', subcat);
        prob.addQuery('cmdb_ci', ci);
        prob.addQuery('state', 'NOT IN', '4'); // Not closed
        prob.query();
        return prob.next();
    },

    _createProblem: function(category, subcat, ci, incidentCount) {
        var prob = new GlideRecord('problem');
        prob.initialize();
        prob.short_description = 'Recurring ' + category + ' issue detected (' +
                                  incidentCount + ' incidents in 7 days)';
        prob.category    = category;
        prob.subcategory = subcat;
        prob.cmdb_ci     = ci;
        prob.priority    = '2';
        prob.state       = '1'; // Open
        prob.u_auto_detected = true;
        var sysId = prob.insert();

        gs.info('ProblemDetector: Created problem ' + prob.number +
                ' for ' + incidentCount + ' recurring incidents');
        return sysId;
    },

    type: 'ProblemDetector'
};
```

---

## 🔄 Module 3: Change Management

### Change Risk Auto-Scoring

```javascript
// Business Rule: Auto-calculate change risk score
// Table: change_request | When: before | Insert + Update
// Name: "ITSM - Change Risk Auto Score"

(function executeRule(current, previous) {

    var riskScore = 0;

    // Factor 1: Number of CIs affected (0-25 points)
    var ciCount = _getAffectedCICount(current.sys_id);
    if      (ciCount >= 20) riskScore += 25;
    else if (ciCount >= 10) riskScore += 15;
    else if (ciCount >= 5)  riskScore += 10;
    else                    riskScore += 5;

    // Factor 2: Change window (0-20 points)
    var hour = new GlideDateTime(current.start_date).getHourLocalTime();
    var isWeekend = _isWeekend(current.start_date);
    if (!isWeekend && hour >= 8 && hour <= 18) riskScore += 20; // Business hours
    else if (isWeekend)                         riskScore += 5;  // Weekend
    else                                        riskScore += 0;  // Off-hours (low risk)

    // Factor 3: Downtime required (0-25 points)
    if (current.u_requires_downtime == true) riskScore += 25;

    // Factor 4: Backout plan documented (0-15 points)
    if (!current.backout_plan || current.backout_plan.toString().length < 50) {
        riskScore += 15; // No/inadequate backout plan = more risk
    }

    // Factor 5: Test plan documented (0-15 points)
    if (!current.test_plan || current.test_plan.toString().length < 50) {
        riskScore += 15; // No test plan = more risk
    }

    // Set risk and risk score
    current.u_risk_score = riskScore;

    if      (riskScore >= 75) current.risk = '1'; // High
    else if (riskScore >= 40) current.risk = '2'; // Medium
    else                      current.risk = '3'; // Low

    function _getAffectedCICount(changeSysId) {
        var rel = new GlideAggregate('task_ci');
        rel.addQuery('task', changeSysId);
        rel.addAggregate('COUNT');
        rel.query();
        return rel.next() ? parseInt(rel.getAggregate('COUNT')) : 0;
    }

    function _isWeekend(dateTime) {
        var dt = new GlideDateTime(dateTime);
        var day = dt.getDayOfWeek();
        return day === 1 || day === 7; // Sunday=1, Saturday=7
    }

})(current, previous);
```

### Change Freeze Window Enforcement

```javascript
// Client Script: Warn if change falls in freeze period
// Table: change_request | Type: onChange | Field: start_date

function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || !newValue) return;

    // Define freeze windows (format: MM-DD)
    var freezeWindows = [
        { start: '12-15', end: '01-02', label: 'Year-End Freeze' },
        { start: '03-30', end: '04-02', label: 'Quarter-End Freeze' },
        { start: '06-29', end: '07-02', label: 'Mid-Year Freeze' }
    ];

    var changeDate = new GlideDateTime(newValue);
    var month = changeDate.getMonthLocalTime();
    var day   = changeDate.getDayOfMonthLocalTime();
    var mmdd  = String(month).padStart(2,'0') + '-' + String(day).padStart(2,'0');

    var inFreeze = freezeWindows.some(function(w) {
        return mmdd >= w.start && mmdd <= w.end;
    });

    if (inFreeze) {
        var matchedWindow = freezeWindows.find(function(w) {
            return mmdd >= w.start && mmdd <= w.end;
        });
        g_form.showFieldMsg(
            'start_date',
            '⚠️ WARNING: This date falls within the ' + matchedWindow.label +
            '. Emergency changes only. CAB approval required.',
            'warning'
        );
    } else {
        g_form.clearFieldMessages('start_date');
    }
}
```

---

## 📋 Module 4: Service Catalog

### Catalog Item Design Principles

```
Good Catalog Item Anatomy:
┌─────────────────────────────────────────┐
│  📦 Item Name (clear, user-friendly)    │
│  🏷️ Category (logical grouping)         │
│  📝 Description (what user gets)        │
│  ⏱️ Fulfillment time expectation        │
│  💰 Cost (if applicable)               │
├─────────────────────────────────────────┤
│  VARIABLES (only ask what's needed):   │
│   ├── Required fields first            │
│   ├── Optional fields last             │
│   └── Use reference fields not text   │
├─────────────────────────────────────────┤
│  WORKFLOW:                              │
│   ├── Auto-approve if low risk         │
│   ├── Manager approval if high cost    │
│   └── Auto-fulfill where possible     │
└─────────────────────────────────────────┘
```

### Dynamic Catalog Variable — Manager Auto-Populate

```javascript
// Client Script: Auto-populate manager when user is selected
// Table: sc_cart_item | Type: onChange | Field: requested_for

function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || !newValue) return;

    // Look up the manager of selected user
    var ga = new GlideAjax('CatalogHelper');
    ga.addParam('sysparm_name', 'getManager');
    ga.addParam('sysparm_user_id', newValue);
    ga.getXMLAnswer(function(answer) {
        if (answer) {
            g_form.setValue('manager', answer);
            g_form.setReadOnly('manager', true);
        }
    });
}
```

```javascript
// Script Include: CatalogHelper (called by Client Script above)
var CatalogHelper = Class.create();
CatalogHelper.prototype = Object.extendsObject(AbstractAjaxProcessor, {

    getManager: function() {
        var userId = this.getParameter('sysparm_user_id');
        var user = new GlideRecord('sys_user');
        if (user.get(userId)) {
            return user.manager.toString();
        }
        return '';
    },

    type: 'CatalogHelper'
});
```

### Variable Set: Standard Requester Info (Reusable)

```
Variable Set Name: "Standard Requester Information"
Variables:
  1. Requested For    [Reference: sys_user] [Mandatory]
  2. Business Purpose [String 200]          [Mandatory]
  3. Cost Center      [Reference: cost_center] [Auto-populated]
  4. Manager Approval [Reference: sys_user] [Read-only, auto-populated]
  5. Priority         [Choice: Normal/Urgent/Emergency] [Default: Normal]
  6. Additional Notes [Multi-line text]     [Optional]

Usage: Include this variable set in ALL catalog items
       → Consistent data capture across entire catalog
```

---

## 📚 Module 5: Knowledge Management

### Knowledge Base Structure

```
Knowledge Base: IT Self-Service
│
├── 📂 Hardware
│   ├── Laptop Issues
│   ├── Printer Setup
│   └── Mobile Devices
│
├── 📂 Software
│   ├── Microsoft Office
│   ├── Email & Outlook
│   ├── VPN Access
│   └── Enterprise Applications
│
├── 📂 Network & Access
│   ├── Password Reset
│   ├── Account Unlock
│   ├── VPN Setup
│   └── Wi-Fi Configuration
│
├── 📂 How-To Guides
│   ├── Submit a Ticket
│   ├── Request New Software
│   └── Onboarding Checklist
│
└── 📂 Known Errors (KEDB)
    ├── Active Known Errors
    └── Resolved Known Errors
```

### Knowledge Deflection Setup

```javascript
// Business Rule: Suggest KB articles before incident creation
// Table: incident | When: before | Insert
// Name: "ITSM - Knowledge Deflection Check"

(function executeRule(current, previous) {

    if (!current.short_description) return;

    // Search knowledge base for matching articles
    var search = new GlideRecord('kb_knowledge');
    search.addActiveQuery();
    search.addQuery('workflow_state', 'published');
    search.addQuery(
        search.addQuery('short_description', 'CONTAINS', current.short_description)
            .addOrCondition('text', 'CONTAINS', current.short_description)
    );
    search.setLimit(3);
    search.query();

    var articles = [];
    while (search.next()) {
        articles.push({
            number: search.number.toString(),
            title:  search.short_description.toString(),
            sys_id: search.sys_id
        });
    }

    if (articles.length > 0) {
        // Store suggested articles on incident for UI display
        current.u_suggested_kb = JSON.stringify(articles);
        gs.info('Knowledge Deflection: Found ' + articles.length +
                ' articles for incident');
    }

})(current, previous);
```

---

## ⏱️ Module 6: SLA Design

### SLA Framework

```
SLA Hierarchy:
  SLA Definition (the commitment)
    ├── SLA Conditions (when it applies)
    ├── Duration (how long to resolve)
    ├── Schedule (business hours vs 24x7)
    └── Breach Actions (what happens when breached)

OLA (Operational Level Agreement)
  └── Internal team commitment to support the SLA

UC (Underpinning Contract)
  └── Vendor/supplier commitment to support the SLA
```

### SLA Design Table

| Priority | Response | Acknowledge | Resolution | Schedule |
|----------|----------|-------------|------------|----------|
| P1 - Critical | 5 min | 15 min | 4 hours | 24x7 |
| P2 - High | 15 min | 30 min | 8 hours | 24x7 |
| P3 - Medium | 30 min | 2 hours | 24 hours | Business Hours |
| P4 - Low | 2 hours | 4 hours | 72 hours | Business Hours |

### SLA Condition Script

```javascript
// SLA Condition: Apply P1 Resolution SLA
// Returns true when SLA should START

(function slaCondition(current, isReset) {
    // Start SLA when:
    // 1. Priority is Critical (1)
    // 2. Incident is in active state
    // 3. Not already breached

    return current.priority == '1' &&
           current.state != '6' &&  // Not resolved
           current.state != '7';    // Not closed
})();
```

### Proactive SLA Breach Prevention

```javascript
// Scheduled Job: Warn before SLA breach
// Name: "ITSM - SLA Breach Warning"
// Runs: Every 15 minutes

var gr = new GlideRecord('incident_sla');
gr.addQuery('stage', 'IN_PROGRESS');
gr.addQuery('breach', false);

// SLA at 75% or more of time used
var now = new GlideDateTime();
gr.query();

while (gr.next()) {

    var plannedEnd = new GlideDateTime(gr.planned_end_time);
    var totalDuration = gr.duration.dateNumericValue();
    var elapsed = now.dateNumericValue() - new GlideDateTime(gr.start_time).dateNumericValue();
    var percentUsed = (elapsed / totalDuration) * 100;

    if (percentUsed >= 75 && percentUsed < 100) {

        var incident = new GlideRecord('incident');
        if (!incident.get(gr.task)) continue;

        // Only notify once (check flag)
        if (incident.u_sla_warning_sent == true) continue;

        // Notify assigned agent
        var email = new GlideEmailOutbound();
        email.setTo(incident.assigned_to.email.toString());
        email.setCC(incident.assignment_group.manager.email.toString());
        email.setSubject('[SLA WARNING] ' + Math.round(100 - percentUsed) +
                         '% time remaining — ' + incident.number);
        email.setBody(
            'SLA Warning for Incident: ' + incident.number + '\n' +
            'Priority: ' + incident.priority.getDisplayValue() + '\n' +
            'Time Remaining: ' + Math.round(100 - percentUsed) + '%\n' +
            'SLA Breach At: ' + plannedEnd.getDisplayValue() + '\n\n' +
            'Please take immediate action to resolve this incident.'
        );
        email.save();

        // Mark warning as sent
        incident.u_sla_warning_sent = true;
        incident.update();
    }
}
```

---

## 📊 ITSM KPIs & Performance Analytics

### Core KPI Definitions

| KPI | Formula | Target | Frequency |
|-----|---------|--------|-----------|
| First Call Resolution (FCR) | Incidents resolved at Level 1 / Total Incidents × 100 | > 70% | Weekly |
| Mean Time to Resolve (MTTR) | Sum of Resolution Times / Total Incidents | P1 < 4h | Daily |
| SLA Compliance Rate | Incidents resolved within SLA / Total × 100 | > 95% | Daily |
| Incident Reopen Rate | Reopened Incidents / Resolved Incidents × 100 | < 5% | Weekly |
| Change Success Rate | Successful Changes / Total Changes × 100 | > 95% | Monthly |
| Change-Induced Incidents | Incidents caused by changes / Total Changes × 100 | < 5% | Monthly |
| Knowledge Deflection Rate | Self-resolved via KB / Total Contacts × 100 | > 30% | Monthly |
| Catalog Fulfillment Time | Avg days from request to fulfillment | < 3 days | Weekly |
| Problem Resolution Rate | Closed Problems / Total Problems × 100 | > 80% | Monthly |
| Customer Satisfaction (CSAT) | Avg survey score (1-5) | > 4.2 | Weekly |

### Performance Analytics Dashboard Layout

```
┌─────────────────────────────────────────────────────────┐
│  ITSM EXECUTIVE DASHBOARD                               │
├──────────────┬──────────────┬───────────────────────────┤
│ SLA          │ Open P1/P2   │ MTTR Trend (30 days)      │
│ Compliance   │ Incidents    │ [Line Chart]               │
│ 96.4% ✅    │ P1: 2 | P2:7 │                           │
├──────────────┼──────────────┼───────────────────────────┤
│ FCR Rate     │ Incidents    │ Change Success Rate        │
│ 72% ✅      │ by Category  │ 97.2% ✅                  │
│              │ [Pie Chart]  │                           │
├──────────────┼──────────────┼───────────────────────────┤
│ KB Deflection│ CSAT Score   │ Open Problems              │
│ 34% ✅      │ 4.3/5.0 ✅   │ High: 2 | Med: 8 | Low: 14│
└──────────────┴──────────────┴───────────────────────────┘
```

---

## 🔒 ACL Security Patterns

### ITSM Role Structure

```
itil_admin
  └── Full access to all ITSM records and configuration

itil
  └── Create/Read/Write own incidents and assigned incidents
  └── Read all incidents in assignment group

itil_view
  └── Read-only access to incidents

sn_change_requester
  └── Create and track own change requests

sn_change_manager
  └── Full change management access

sn_problem_manager
  └── Full problem management access

kb_manager
  └── Manage knowledge bases and article approval

catalog_admin
  └── Create and manage catalog items
```

### ACL Design: Protect Sensitive Fields

```javascript
// ACL Script: Protect work notes from non-ITSM users
// Operation: Read | Object: incident | Field: work_notes

(function reCell(/*GlideRecord*/ current, /*GlideUser*/ currentUser) {
    // Only agents with itil role can see work notes
    return currentUser.hasRole('itil') || currentUser.hasRole('itil_admin');
})(current, currentUser);
```

---

## ✅ Go-Live Checklist

### Pre-Go-Live
- [ ] All assignment groups created with correct members
- [ ] SLA definitions tested end-to-end (start, pause, resume, breach)
- [ ] Auto-assignment business rules tested for all categories
- [ ] Email notifications verified (all templates render correctly)
- [ ] Service Catalog items tested by end users (UAT signed off)
- [ ] Knowledge base populated with minimum 20 articles
- [ ] Performance Analytics dashboards displaying live data
- [ ] P1 escalation path verified (call tree documented)
- [ ] Integration with monitoring tools tested (auto-incident creation)
- [ ] Update sets packaged and deployed to production

### Post-Go-Live (30 Days)
- [ ] Daily SLA compliance review
- [ ] FCR rate monitored (target > 70%)
- [ ] Catalog fulfillment times reviewed
- [ ] Knowledge deflection rate tracked
- [ ] User feedback / CSAT surveys reviewed
- [ ] Assignment group workload balanced
- [ ] Hypercare support team on standby

---

## 👤 Author

**Swarup Kumar Namana**
Senior ServiceNow Developer & Platform Architect
Columbus, Ohio, USA

[![Portfolio](https://img.shields.io/badge/Portfolio-swarup--namana.netlify.app-b8922a?style=flat-square)](https://swarup-namana.netlify.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-swarupnamana-0077B5?style=flat-square&logo=linkedin)](https://linkedin.com/in/swarupnamana)
[![Email](https://img.shields.io/badge/Email-swarupnamana03%40gmail.com-D14836?style=flat-square&logo=gmail)](mailto:swarupnamana03@gmail.com)

---

*All patterns built from real enterprise ITSM implementations. Client data abstracted. Patterns generalized for reuse across organizations.*
