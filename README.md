# Multi-Tenancy Implementation Guide

This README explains in detail how Iomad (a fork of Moodle 4.0.5) implements robust multi-tenant functionality by layering a “company” model on top of Moodle core. It covers:

- How new database structures isolate tenants  
- The custom context level that scopes permissions per company  
- API classes that manage tenant data  
- UI components that filter content  
- Event hooks that maintain tenant associations  
- Core patches that enforce access at entry points  
- Utilities and usage flow for developers and administrators  

All custom logic resides in Iomad’s local/iomad and blocks/iomad_company_admin plugins and a single custom context class. No upstream Moodle files are altered except for two entry-point scripts.

---

Table of Contents

1. Overview  
2. Database Schema Extensions  
   2.1 Company Table  
   2.2 Related Tables  
3. Context Level: Company  
4. Local API Classes  
5. UI Blocks & Selectors  
6. Event Observers  
7. Core File Patches  
8. iomad_check_course() Utility  
9. Usage Flow  
10. Contributing & Extending  

---

Overview

Iomad extends Moodle by introducing a **company** abstraction as the fundamental tenant unit. Each company is isolated at every layer:

- Data: new tables map courses and users to companies  
- Context: a custom context level ensures capabilities are scoped per company  
- API: helper classes simplify tenant-aware operations  
- UI: selectors and forms filter lists to the current company  
- Access Control: core scripts are lightly patched to block cross-company access  

This architecture ensures a user in one company cannot inadvertently see or manipulate another company’s data.

---

Database Schema Extensions

Before any code runs, Iomad adds new tables to hold tenant metadata. These tables appear under local/iomad/db/install.xml.

2.1 Company Table

What it does: Defines each tenant with its own settings and hierarchy. This table is entirely absent in vanilla Moodle.

File: local/iomad/db/install.xml  
<TABLE NAME="company" COMMENT="Tenant companies">  
  <FIELD NAME="id" TYPE="int" LENGTH="10" NOTNULL="true" SEQUENCE="true"/>  
  <FIELD NAME="name" TYPE="char" LENGTH="50" NOTNULL="true"/>  
  <FIELD NAME="shortname" TYPE="char" LENGTH="25" NOTNULL="true"/>  
  <FIELD NAME="parentid" TYPE="int" LENGTH="20" NOTNULL="true" DEFAULT="0"/>  
  <FIELD NAME="logo" TYPE="text" NOTNULL="false"/>  
  <FIELD NAME="theme" TYPE="char" LENGTH="50" NOTNULL="false"/>  
  <!-- Additional fields for authentication, SMTP, policies -->  
</TABLE>

---

2.2 Related Tables

What they do: Link core Moodle entities to companies, enabling join-based isolation.

- company_course: Records which courses belong to a given company.  
- company_shared_courses: Tracks sharing modes (open/closed) across companies.  
- company_created_courses: Logs which company originally created each course.  
- iomad_courses: A SQL view normalizing data from the above tables for easy queries.  
- company_users: (Optional) Associates users to companies; in some versions, a company field on user is used instead.  

---

Context Level: Company

To leverage Moodle’s capability system per tenant, Iomad introduces a new context level under the system context.

What it achieves: Allows require_capability() calls to be scoped to a company, blocking users from executing actions outside their tenant.

File: lib/classes/context/company.php  
namespace core\context;  
class company extends context {  
    public const LEVEL = 13;  
    protected function __construct(stdClass $record) {  
        parent::__construct($record);  
        if ($record->contextlevel != self::LEVEL) {  
            throw new coding_exception('Invalid context level');  
        }  
    }  
    public static function instance($companyid, $strictness = MUST_EXIST) {  
        return parent::instance(self::LEVEL, $companyid, $strictness);  
    }  
    public static function get_possible_parent_levels(): array {  
        return [\context_system::LEVEL];  
    }  
}

- LEVEL=13: New numeric context code.  
- Parent: Always under system context (no nested contexts).  
- Usage: Pass $companycontext to require_capability() for any company-scoped operation.

---

Local API Classes

Iomad provides PHP classes to abstract tenant-specific operations, minimizing direct DB queries in your own code.

4.1 Company Wrapper

What it achieves: Encapsulates company record retrieval and context instantiation.

File: local/iomad/lib/company.php  
class company {  
    public $id;  
    public $context;  
    protected $companyrecord;  
    public function __construct($companyid) {  
        global $DB;  
        $this->id = $companyid;  
        $this->companyrecord = $DB->get_record('company', ['id'=>$companyid]);  
        $this->context = \core\context\company::instance($companyid);  
    }  
    public function get($fields) { /* fetch fields */ }  
    public static function by_userid($userid) { /* lookup company by user */ }  
}

4.2 Company User Helper

What it achieves: Determines the active tenant for the current user and checks company membership.

File: local/iomad/lib/user.php  
class company_user {  
    public static function companyid() {  
        global $USER, $DB;  
        return $DB->get_field('company', ['shortname'=>$USER->company], 'id');  
    }  
    public static function is_company_user() {  
        return !is_siteadmin() && self::companyid() > 0;  
    }  
    public static function can_see_company($shortname) { /* permission logic */ }  
}

- companyid(): Fetches the current user’s company ID.  
- is_company_user(): Excludes site administrators.

---

UI Blocks & Selectors

All administrative forms and lists in Iomad are wrapped to only show tenant-relevant data.

5.1 Course Selector

What it alters: Replaces Moodle’s default course dropdown with one filtered to the current company.

File: blocks/iomad_company_admin/lib/course_selectors.php  
class current_company_course_selector extends course_selector {  
    protected function build_sql(&$params) {  
        $companyid = $this->options['companyid'];  
        $sql = "SELECT c.id, c.fullname AS name FROM {course} c"  
             ." JOIN {company_course} cc ON cc.courseid=c.id"  
             ." WHERE cc.companyid=:companyid AND c.visible=1"  
             ." ORDER BY c.sortorder";  
        $params = ['companyid' => $companyid];  
        return [$sql, $params];  
    }  
}

5.2 User Selector

What it alters: Limits user assignments to those belonging to the active company.

File: blocks/iomad_company_admin/lib/user_selectors.php  
class current_company_user_selector extends user_selector_base {  
    protected function get_sql($search) {  
        $companyid = $this->options['companyid'];  
        $sql = "SELECT u.id, u.firstname, u.lastname FROM {user} u"  
             ." JOIN {company_users} cu ON cu.userid=u.id"  
             ." WHERE cu.companyid=:companyid";  
        return [$sql, ['companyid' => $companyid]];  
    }  
}

- Selectors: Always include a JOIN on the relevant company link table.

---

Event Observers

Iomad hooks into Moodle events to maintain tenant link tables automatically.

6.1 Connecting the Observer

File: local/iomad/db/events.php  
\core\event\course_created => [  
    'callback' => 'local_iomad\classes\observer::course_created',  
],

6.2 Observer Implementation

File: local/iomad/classes/observer.php  
publicstatic function course_created(\core\event\course_created $event) {  
    global $DB;  
    $courseid = $event->objectid;  
    $companyid = iomad::get_my_companyid(  
        \context_coursecat::instance($event->contextinstanceid)  
    );  
    $DB->insert_record('company_course', (object)[  
        'companyid' => $companyid,  
        'courseid'  => $courseid  
    ]);  
}

- Purpose: Ensures every new course is tracked under its creator’s tenant.

---

Core File Patches

To block cross-tenant access at runtime, Iomad lightly patches two entry points. All other core files remain untouched.

7.1 course/view.php

Customization: Injects a guard that verifies the requested course belongs to the user’s company. If not, it hijacks the ID to SITEID, causing the standard login or permission checks to fail.

[diff against upstream omitted for brevity]

7.2 enrol/index.php

Customization: Applies the same guard before loading the enrolment page.

[diff omitted for brevity]

---

iomad_check_course() Utility

Central helper to verify ownership of a course by the current tenant.

File: local/iomad/lib.php  
publicstatic function iomad_check_course($courseid) {  
    global $DB;  
    $companyid = company_user::companyid();  
    return $DB->record_exists('company_course', [  
        'companyid' => $companyid,  
        'courseid'  => $courseid  
    ]);  
}

- Outcome: All access points calling this function enforce tenant boundaries.

---

Usage Flow

1. Login: company_user::companyid() determines tenant context.  
2. Course creation: Observer writes to company_course.  
3. Listing: UI selectors filter by companyid.  
4. Viewing/enrolling: Core patches call iomad_check_course().  
5. Capabilities: Scoped to context_company.

---

Contributing & Extending

- Adding features: Always include companyid filters or joins in new queries.  
- Enforcing permissions: Use company::instance($companyid) and scoped require_capability().  
- Event hooks: Maintain custom tables on CRUD operations.

For full context, see Iomad’s code directories: local/iomad and blocks/iomad_company_admin.
