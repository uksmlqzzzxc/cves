# Security Research Report

## SQL Injection Vulnerabilities in School Attendance System (SAS)

| Item | Details |
|--------|--------|
| Report ID | SAS-SQLI-2026-01 |
| Version | 1.0 |
| Author | uksmlqzzzxc |
| Analysis Perspective | Defender-Oriented Security Analysis |
| Target Project | https://github.com/0mehedihasan/sas |
| Analyzed Commit | `730de5a457eb4e4fe37a25b83e916bd9f1806980` |
| Vulnerability Reference | Issue #1 |
| Vulnerability Type | SQL Injection |
| CWE | CWE-89 |
| Severity | High / Critical |
| Estimated CVSS | 8.5 – 9.8 |

---

# Executive Summary

The School Attendance System (SAS) is a PHP/MySQL-based web application that uses a distributed database architecture consisting of multiple grade-level database partitions.

During source-code review, multiple SQL Injection (SQLi) vulnerabilities were identified across administrative components. The application directly concatenates user-controlled input into SQL statements without parameterized queries, prepared statements, input validation, or secure database abstraction.

---

# Security Impact

## Confidentiality

Potential disclosure of:

- Student records
- Teacher information
- Administrative accounts
- Attendance data
- Academic information

## Integrity

Attackers may:

- Modify attendance records
- Alter class information
- Manipulate session data
- Escalate privileges

## Availability

Potential attacks include:

- Mass deletion of records
- Database corruption
- Denial-of-service through destructive SQL operations

## Architectural Risk

Successful injection may affect multiple database partitions including:

- sas_six
- sas_seven
- sas_eight
- sas_other

---

# Root Cause Analysis

## Root Cause #1 — Dynamic SQL Construction

Example:

```php
$Id = $_GET['Id'];

$query = mysqli_query(
    $dbConnection,
    "DELETE FROM tblclass WHERE Id='$Id'"
);
```

User-controlled input is directly embedded into executable SQL.

## Root Cause #2 — Lack of Prepared Statements

The application does not consistently use:

```php
mysqli_prepare();
bind_param();
```

or PDO prepared statements.

## Root Cause #3 — Missing Input Validation

User-controlled parameters from `$_GET` and `$_POST` are used without validation.

## Root Cause #4 — Weak Access Control

Authorization checks are not uniformly enforced.

## Root Cause #5 — Unsafe Database Architecture

Raw database connections are exposed without a secure abstraction layer.

---

# Vulnerable Components

## Admin/createClass.php

### Delete Operation

```php
$Id = $_GET['Id'];

$query = mysqli_query(
    $dbConnection,
    "DELETE FROM tblclass WHERE Id='$Id'"
);
```

### Risk

Attacker-controlled parameters:

- Id
- dbKey

may influence SQL execution.

---

## Admin/createSessionTerm.php

### Vulnerable SELECT

```php
$query = mysqli_query(
    $conn,
    "select * from tblsessionterm
     where sessionName ='$sessionName'
     and termId = '$termId'"
);
```

### Vulnerable DELETE

```php
$query = mysqli_query(
    $conn,
    "DELETE FROM tblsessionterm WHERE Id='$Id'"
);
```

---

# Additional Affected Areas

Potentially affected files include:

- createStudents.php
- createClassArms.php
- createClassTeacher.php

Any file directly embedding `$_GET` or `$_POST` values into SQL statements should be reviewed.

---

# Attack Scenarios

## Data Extraction

Potential access to:

- tbladmin
- tblstudents
- tblteachers
- tblattendance

## Data Manipulation

```sql
UPDATE tblattendance
SET status='Present';
```

## Data Destruction

```sql
DELETE FROM tblclass;
```

## Privilege Escalation

Compromised administrative functionality may enable broader access.

---

# Risk Assessment

| Metric | Value |
|----------|----------|
| Attack Vector | Network |
| Attack Complexity | Low |
| Privileges Required | Low / None |
| User Interaction | None |
| Confidentiality | High |
| Integrity | High |
| Availability | High |

### Estimated CVSS

`8.5 – 9.8 (High / Critical)`

---

# Complete Remediation Plan

## Phase 1 — Immediate Fixes

### Adopt Prepared Statements

```php
$stmt = $dbConnection->prepare(
    "DELETE FROM tblclass WHERE Id=?"
);

$stmt->bind_param("i", $Id);
$stmt->execute();
```

### Input Validation

```php
$Id = (int)$_GET['Id'];
```

Apply:

- Type validation
- Length validation
- Allowlist validation

### Centralized Database Layer

Create a secure Database.php abstraction layer.

---

## Phase 2 — Security Improvements

### Use PDO

Recommended:

```php
PDO::ATTR_EMULATE_PREPARES => false
```

### Strengthen Authorization

- Session validation
- RBAC
- CSRF protection

### Database Hardening

- Least privilege
- Query logging
- WAF protection

### Additional Security Controls

- htmlspecialchars()
- bcrypt / Argon2
- Rate limiting
- Secure error handling

---

## Phase 3 — Verification & Long-Term Security

### Security Testing

Perform:

- SQLMap validation
- OWASP ZAP testing
- Static analysis

Examples:

- PHPStan
- Psalm
- Semgrep

### Monitoring

Implement:

- Application logging
- Database auditing
- Security monitoring

### Refactoring

Recommended migration:

- Laravel
- CodeIgniter

or a secure query builder.

---

# Patch Priority

1. DELETE endpoints
2. UPDATE endpoints
3. Administrative CRUD modules
4. Student-facing queries
5. Full source-code audit

---

# Conclusion

The identified SQL Injection vulnerabilities stem from unsafe dynamic SQL construction and the absence of modern secure coding practices.

Recommended priorities:

1. Parameterized queries
2. Input validation
3. Centralized database access
4. Least-privilege permissions
5. Continuous security testing

A defense-in-depth strategy should be adopted to reduce future risk and improve the overall security posture of the application.

---

# References

## Original Disclosure

- Issue #1

## Security Standards

- CWE-89: https://cwe.mitre.org/data/definitions/89.html
- OWASP Top 10 A03:2021 Injection
- OWASP SQL Injection Prevention Cheat Sheet

## Risk Assessment

- https://www.first.org/cvss/calculator/3.1

---

# Author

**Researcher:** uksmlqzzzxc

**GitHub:** https://github.com/uksmlqzzzxc

**Research Repository:** https://github.com/uksmlqzzzxc/cves

**Contact:** uksmlqzzzxc@proton.me

---

# Disclaimer

This report was produced through source-code review and defensive security analysis.

The purpose of this research is to support vulnerability remediation, secure software development, and defensive security improvement.
