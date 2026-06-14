Security Research Report: SQL Injection Vulnerabilities in School Attendance System (SAS)Report ID: SAS-SQLI-2026-01
Version: 1.0
Author: uksmlqzzzxc (Defender Perspective Analysis)
Target Project: https://github.com/0mehedihasan/sas (commit 730de5a457eb4e4fe37a25b83e916bd9f1806980)
Vulnerability Reporter Reference: https://github.com/uksmlqzzzxc/cves/issues/1 (Chinese-language disclosure) 

Executive SummaryThe School Attendance System (SAS) is a PHP/MySQL web application using a distributed database architecture (four MySQL databases partitioned by grade level). It contains multiple critical SQL Injection (SQLi) vulnerabilities due to direct string concatenation of user-controlled input into SQL queries without sanitization or parameterization. 

Impact (from Defender View):  High to Critical severity (CVSS-like: 8.5–9.8). Authenticated attackers (or unauthenticated in some paths) can read, modify, or delete arbitrary data across distributed databases.  
Potential full compromise: data exfiltration (student/teacher records), unauthorized CRUD, privilege escalation, or denial-of-service via destructive DELETE/UPDATE.  
Distributed architecture amplifies risk—successful injection can target specific grade-level partitions (e.g., sas_six, sas_other).  
No evidence of input validation, prepared statements, or least-privilege DB users.

These issues violate core secure coding practices (OWASP Top 10 A03:2021 – Injection).Vulnerability Details and Root Cause Analysis1. Core Root CauseDirect string concatenation with mysqli_query() and variables from $_GET/$_POST (e.g., $Id = $_GET['Id']; ... "WHERE Id='$Id'").  
No use of prepared statements (mysqli_prepare + bind_param), PDO with placeholders, or escaping functions like mysqli_real_escape_string (and even escaping is insufficient for all cases).  
Weak or missing access controls in some endpoints (e.g., session checks bypassed or not enforced uniformly).  
Error reporting disabled (error_reporting(0);), hiding potential leaks.  
Distributed DB handler (Includes/dbcon.php) passes raw connections without abstraction for safe querying. 

2. Specific Vulnerable Endpoints (Examples)Admin/createClass.php (DELETE action):  php

if (isset($_GET['Id']) && isset($_GET['action']) && $_GET['action'] == "delete" && isset($_GET['dbKey'])) {
    $Id = $_GET['Id'];
    $dbKey = $_GET['dbKey'];
    // ... switch to select $dbConnection
    $query = mysqli_query($dbConnection, "DELETE FROM tblclass WHERE Id='$Id'");
}

Attacker controls $Id and $dbKey via URL parameters.  
Classic tautology or UNION-based injection possible (e.g., Id=1' OR '1'='1 or time-based blind).  
Similar patterns in edit/save (className from POST). 

raw.githubusercontent.com

Admin/createSessionTerm.php (Multiple operations):  INSERT/SELECT/UPDATE/DELETE all concatenate $sessionName, $termId, $Id directly.  
Example:  php

$query=mysqli_query($conn,"select * from tblsessionterm where sessionName ='$sessionName' and termId = '$termId'");
$query=mysqli_query($conn,"DELETE FROM tblsessionterm WHERE Id='$Id'");

Affects unauthenticated paths if session.php is bypassed; impacts shared tblsessionterm table. 

raw.githubusercontent.com

Other Likely Affected Files (Pattern Match):  createStudents.php, createClassArms.php, createClassTeacher.php, etc. (CRUD operations).  
Any file using $_POST/$_GET directly in queries.

Attack Scenarios (Defender View):  Authenticated low-priv user (e.g., teacher/student) escalates via admin endpoints.  
Delete arbitrary classes/sessions → data loss.  
Extract sensitive data (e.g., UNION SELECT password FROM tbladmin).  
Stack queries for multi-statement attacks if MySQL allows.

Risk AssessmentAffected Versions: All (up to latest commit analyzed).  
Prerequisites: Network access to admin panels; valid session in most cases.  
Exploitability: High (simple URL/POST manipulation; tools like sqlmap).  
Business Impact: Compromised attendance records, privacy breaches (student data), operational disruption.

Complete Remediation Plan (Defender Posture)Phase 1: Immediate Fixes (Short-Term Hardening)Adopt Prepared Statements Everywhere:
Replace all mysqli_query with prepared statements.
Example for DELETE in createClass.php:  php

if ($dbConnection) {
    $stmt = $dbConnection->prepare("DELETE FROM tblclass WHERE Id = ?");
    $stmt->bind_param("i", $Id);  // 'i' for integer
    $stmt->execute();
    $stmt->close();
}

Do the same for SELECT, INSERT, UPDATE across all files.
Input Validation & Sanitization:  Whitelist allowed values (e.g., class names: only 6-12).  
Cast IDs: $Id = (int)$_GET['Id'];  
Use filter_input() or strict type checks.  
For names/strings: length limits + regex.

Centralize Database Access:
Create a secure wrapper in Includes/dbcon.php or a new Database.php class with methods using prepared statements. Enforce it project-wide.

Phase 2: Secure Architecture ImprovementsUse PDO (Recommended) or mysqli Properly:
PDO with PDO::ATTR_EMULATE_PREPARES => false for true prepared statements.
Access Control & Authorization:  Enforce session.php checks at the top of every admin file.  
Role-based checks before any CRUD.  
CSRF tokens on all forms.

Database Best Practices:  Run MySQL with least-privilege users (no root for web app).  
Enable query logging and WAF (e.g., ModSecurity).  
Consider migrating away from distributed raw connections to a single secure layer or proper sharding library.

Additional Defenses:  Output Encoding: Prevent XSS (use htmlspecialchars).  
Password Hashing: (Noted in project contributing section—implement bcrypt/Argon2).  
Rate Limiting & Logging: On admin actions.  
Error Handling: Custom errors, no raw DB errors to users.

Phase 3: Verification & Long-TermScanning: Run sqlmap, OWASP ZAP, or static analyzers (PHPStan + security rules).  
Testing: Unit tests for all DB interactions; penetration testing.  
Dependencies: Update Bootstrap/jQuery/vendor libs.  
Monitoring: Implement application logging (e.g., Monolog) and DB audit.  
Refactoring Suggestion: Move to MVC (Laravel/CodeIgniter) for better security defaults, or at minimum a query builder.

Patch Priority Order:  All DELETE/UPDATE endpoints.  
Admin CRUD files.  
Any student/teacher-facing queries.  
Full audit of remaining PHP files.

Conclusion and RecommendationsThese SQLi issues stem from classic beginner PHP patterns and can be fully eliminated by adopting parameterized queries and input validation. As defenders, prioritize defense-in-depth: code fixes + access controls + monitoring.  After patching, re-test thoroughly and consider a security audit before production use. The project README already highlights security contributions—treat this as priority.  For questions or help verifying patches, provide updated code snippets. Secure coding prevents these issues proactively.

