# INCIDENT 1: Cloud Infrastructure Compromise
## Network Forensics Investigation Report

**Case ID:** VANTAGE-2025-0701  
**Investigation Date:** July 1, 2025  
**Status:** Resolved  
**Severity:** Critical  

---

## Executive Summary

A private cloud infrastructure operated by a small company was compromised through a combination of developer oversight (exposed dashboard) and active attacker exploitation. A suspected attacker discovered the exposed web server, performed reconnaissance, conducted a brute-force attack against the dashboard login, and exfiltrated sensitive employee data via cloud object storage. The investigation reconstructed the complete attack timeline from initial reconnaissance through data exfiltration.

**Investigation Scope:** 2 PCAP files (web server, controller node)  
**Key Finding:** 28 employee records compromised through Swift object storage  
**Root Cause:** Exposed dashboard redirect + weak authentication controls  
**Remediation:** Immediate account disablement, credential rotation, infrastructure audit  

---

## Attack Timeline

| Time (UTC) | Event | Details |
|-----------|-------|---------|
| 09:40:29 | **Reconnaissance** | OpenStack API remote access config file (openrc) downloaded |
| 09:41:44 | **API Discovery** | Initial HTTP GET /identity request to OpenStack Keystone |
| 09:41:44+ | **Service Enumeration** | Attacker queries /identity/v3/projects endpoint |
| 09:41:44+ | **Default Project ID** | 9fb84977ff7c4a0baf0d5dbb57e235c7 identified and accessed |
| 09:40:29+ | **Swift Endpoint** | http://134.209.71.220:8080/v1/AUTH_9fb84977ff7c4a0baf0d5dbb57e235c7 discovered |
| 09:40:29+ | **Container Discovery** | 3 containers enumerated: dev-files, employee-data, user-data |
| 09:45:23 | **Data Exfiltration** | user-details.csv downloaded (28 user records) |
| **Δ ≈ 5 minutes** | **Total Compromise Window** | From first reconnaissance to successful data theft |

---

## Detailed Investigation Findings

### Question 1: Attack Tool Identification

**Finding:** ffuf@2.1.0 (Fuzz Faster U Fool)

**Evidence:**
- Packet #116 HTTP GET request
- User-Agent header: "Fuzz Faster U Fool v2.1.0-dev"
- Format per question hint: `nmap@7.80` style → `ffuf@2.1.0`

<img width="1168" height="602" alt="image" src="https://github.com/user-attachments/assets/6dcb34bd-6388-44cf-a001-3154c59e0e1d" />

**Analysis:**
The attacker used a web fuzzer (ffuf) to discover web endpoints on the exposed server. This tool is designed to identify hidden directories, files, and parameters through brute-force scanning. The presence of ffuf indicates active reconnaissance and directory enumeration attacks against the web server.

**Technique Classification:**
- MITRE: T1592 - Gather Victim Org Information (Active Scanning)
- MITRE: T1046 - Network Service Discovery

---

### Question 2: Subdomain Discovery

**Finding:** cloud.vantage.tech

**Evidence:**
- HTTP request filtering by host header
- Statistics: HTTP request count by host
- Highest frequency: cloud.vantage.tech (42 requests)

<img width="1168" height="567" alt="image" src="https://github.com/user-attachments/assets/3ee8bff9-eaa8-4e5e-b5b9-3d8706ddfbc8" />


**Analysis:**
After web server fuzzing, the attacker identified `cloud.vantage.tech` as the target subdomain. This subdomain hosted the exposed dashboard and became the entry point for subsequent attacks. The high request frequency (42 requests) indicates focused reconnaissance on this specific service.

**Significance:**
The discovery of this subdomain represents successful reconnaissance. The attacker now knew:
- Valid domain structure for the company
- Web application hosting the dashboard
- Service running on this subdomain (OpenStack/health management)

**Technique Classification:**
- MITRE: T1592 - Gather Victim Org Information (Active Scanning)
- MITRE: T1046 - Network Service Discovery

---

### Question 3: Dashboard Login Attempts

**Finding:** 3 failed login attempts before successful compromise

**Evidence:**
- Packet #20696-#21103: Dashboard login requests
- HTTP GET request (Packet #20696): Retrieve login page (200 OK)
- HTTP POST requests:
  - Packet #20822: Failed login attempt (200 OK response)
  - Packet #20837: Failed login attempt (200 OK response)
  - Additional failed attempts with same response pattern
  - Packet #21091: Successful login attempt
  - Packet #21103: HTTP 302 Found (redirect to /dashboard/)

<img width="1168" height="566" alt="image" src="https://github.com/user-attachments/assets/5115e1b6-7094-42b2-91fd-13503cfb97cb" />

<img width="1172" height="547" alt="image" src="https://github.com/user-attachments/assets/a510a95b-5ce4-41f4-a931-93a32ebaa21e" />

<img width="1168" height="562" alt="image" src="https://github.com/user-attachments/assets/a50c5869-092f-4d87-9b3a-9af5c3cf202a" />

<img width="1172" height="566" alt="image" src="https://github.com/user-attachments/assets/8f1ae4f5-4d1a-4a43-b0ff-799662fbf80e" />


**Analysis:**
The attacker conducted a brute-force attack against the dashboard authentication mechanism. After 3 failed credential attempts, the 4th attempt succeeded, indicating either:
- Credential recovery from other sources
- Weak password use
- Successful password guessing

The 302 redirect to `/dashboard/` confirms successful authentication and dashboard access.

**Technique Classification:**
- MITRE: T1110 - Brute Force (Credential-based)
- MITRE: T1190 - Exploit Public-Facing Application

---

### Question 4: OpenStack API Configuration Download

**Finding:** 2025-07-01 09:40:29 (UTC)

**Evidence:**
- HTTP GET /dashboard/project/api_access/openrc/
- Frame timestamp: 2025-07-01 09:40:29
- This represents first access to API credentials

<img width="1171" height="623" alt="image" src="https://github.com/user-attachments/assets/1be0de5f-01d3-41d7-b1ca-b3e6a271c9fb" />

<img width="1170" height="578" alt="image" src="https://github.com/user-attachments/assets/f5250a0b-3803-467f-b036-7a09db250a08" />

**Analysis:**
The attacker accessed the OpenStack dashboard and immediately downloaded the openrc configuration file. This file contains:
- API endpoint URLs
- Authentication credentials
- Project/tenant information
- Environment variables for API access

Downloading this file enabled the attacker to:
1. Use command-line tools to interact with OpenStack APIs
2. Query available services and resources
3. Gain programmatic access to cloud infrastructure
4. Download data via Swift object storage

**Significance:**
This action represents the transition from web interface access to API-level compromise. The attacker now had programmatic access to the entire cloud infrastructure.

---

### Question 5: First API Interaction

**Finding:** 2025-07-01 09:41:44 (UTC)

**Evidence:**
- Controller PCAP analysis
- HTTP GET /identity HTTP/1.1
- First packet: #8490
- Response: HTTP/1.1 300 MULTIPLE CHOICES
- Server response included: http://134.209.71.220/identity/v3/
- Location field specified API version 3

<img width="1168" height="566" alt="image" src="https://github.com/user-attachments/assets/ec3cab5a-7723-4852-b2a1-7d262eb73873" />

<img width="1168" height="565" alt="image" src="https://github.com/user-attachments/assets/5a1dcb4a-cc45-4fdf-8675-dec57305caff" />

<img width="1171" height="562" alt="image" src="https://github.com/user-attachments/assets/062e8d54-5109-4fad-9bfd-031f8ae2ebcd" />


**Analysis:**
Using the downloaded openrc credentials, the attacker made the first API call to OpenStack Keystone (identity service). The HTTP 300 response indicates multiple API versions available, but the server-provided location header directs to version 3.

This action marks the beginning of systematic API enumeration and resource discovery.

---

### Question 6: Project ID Identification

**Finding:** 9fb84977ff7c4a0baf0d5dbb57e235c7

**Evidence:**
- HTTP GET /identity/v3/projects/ (Packet #18066)
- HTTP/1.1 200 OK response
- JSON body contains project information
- domain_id field confirmed as default project

<img width="1167" height="598" alt="image" src="https://github.com/user-attachments/assets/b3d70227-decc-4875-ad89-cc1acec4b286" />

<img width="1165" height="545" alt="image" src="https://github.com/user-attachments/assets/f97a5d12-8959-447b-aa5c-8888323c1200" />

**Analysis:**
The attacker queried the OpenStack Keystone projects endpoint, which returned all accessible projects. The response JSON included:
- Project ID: 9fb84977ff7c4a0baf0d5dbb57e235c7
- Domain ID: (matching field confirming default project)
- Project name and metadata

**Significance:**
This project ID is critical for accessing cloud resources. The attacker now knew the exact project to target for data access via Swift object storage.

---

### Question 7: OpenStack Authentication Service

**Finding:** Keystone (OpenStack Identity Service)

**Evidence:**
- Official OpenStack documentation
- Service type: Identity Service
- Functionality: API client authentication and multi-tenant authorization
- Manages user access to resources

<img width="1036" height="455" alt="image" src="https://github.com/user-attachments/assets/7a329356-5faa-4f01-8b3f-70e8f5dfb0ab" />

**Analysis:**
Keystone is the authentication and authorization component of OpenStack. It provides:
- Token-based authentication (used by attacker)
- Multi-tenant isolation
- Service catalog (listing available endpoints)
- Authorization policies

The attacker leveraged Keystone to:
1. Authenticate using downloaded credentials
2. Retrieve service catalog
3. Discover available endpoints
4. Obtain access tokens for subsequent API calls

---

### Question 8: Swift Object Storage Endpoint

**Finding:** http://134.209.71.220:8080/v1/AUTH_9fb84977ff7c4a0baf0d5dbb57e235c7

**Evidence:**
- HTTP POST /identity/v3/auth/tokens (Packet #8506)
- Response packet #8590: HTTP/1.1 201 CREATED
- JSON response body contains service catalog
- Endpoints section: interface=public, URL shown above
- Service type: object-store

<img width="1037" height="502" alt="image" src="https://github.com/user-attachments/assets/703256cb-f217-414a-826f-22dfb58e2a80" />

<img width="1022" height="527" alt="image" src="https://github.com/user-attachments/assets/3eb38a9a-1f95-42d8-8ec5-02674f515528" />

**Analysis:**
The Keystone authentication response included the service catalog listing all available OpenStack services. The attacker identified:
- Service: Swift (object storage)
- Interface: Public
- Endpoint URL: http://134.209.71.220:8080/v1/AUTH_9fb84977ff7c4a0baf0d5dbb57e235c7
- Project identifier in URL: 9fb84977ff7c4a0baf0d5dbb57e235c7

This endpoint is the direct connection point for accessing cloud object storage.

**Significance:**
Swift is OpenStack's object storage service, functionally similar to Amazon S3. Access to this endpoint allows downloading any files stored in the project's containers.

---

### Question 9: Container Discovery

**Finding:** 3 containers discovered:
1. dev-files
2. employee-data
3. user-data

**Evidence:**
- HTTP GET /v1/AUTH_9fb84977ff7c4a0baf0d5dbb57e235c7?format=json (Packet #12892)
- HTTP/1.1 200 OK response (Packet #13367)
- JSON body contains: X-Account-Container-Count: 3
- Container names listed in JSON array

<img width="1166" height="562" alt="image" src="https://github.com/user-attachments/assets/557c98a7-9164-4183-838e-5789a2976aa9" />

<img width="1168" height="617" alt="image" src="https://github.com/user-attachments/assets/9dad7ef6-ab32-4cb9-a230-c301234615d0" />

**Analysis:**
The attacker made a GET request to the Swift endpoint without specifying a container, which lists all available containers. The response revealed 3 containers holding different types of data:
- **dev-files:** Development resources (non-sensitive)
- **employee-data:** Employee information (moderate sensitivity)
- **user-data:** User records (HIGH sensitivity - TARGET)

The attacker identified user-data as the target for maximum impact.

---

### Question 10: Sensitive Data Download

**Finding:** 2025-07-01 09:45:23 (UTC)

**Evidence:**
- HTTP GET /v1/AUTH_9fb84977ff7c4a0baf0d5dbb57e235c7/user-data/user-details.csv
- Packet #16398
- Frame timestamp: 2025-07-01 09:45:23 UTC
- Response: HTTP/1.1 200 OK
- File transfer completed successfully

<img width="1171" height="622" alt="image" src="https://github.com/user-attachments/assets/a2193020-778f-4fa3-a0a5-fa85f611f33c" />


**Analysis:**
The attacker directly accessed the user-data container and downloaded user-details.csv, a file containing sensitive employee information. The request path reveals:
- Container: user-data
- Object: user-details.csv
- Full URL shows direct file access with known structure

The successful 200 OK response confirms the file was accessible and downloaded without restriction.

---

### Question 11: Data Breach Scope

**Finding:** 28 user records

**Evidence:**
- Following HTTP stream for packet #16398
- CSV file contents enumeration
- Manual count of data records
- Each record includes: Name, Email, Phone, Employee ID

<img width="1167" height="582" alt="image" src="https://github.com/user-attachments/assets/4c7c3f2b-30bd-4089-bec6-c2040943df1d" />

<img width="1152" height="531" alt="image" src="https://github.com/user-attachments/assets/e21d1c69-f225-40fb-8c76-2bc2dac52e96" />

**Analysis:**
The downloaded file contained 28 employee records with the following information per record:
- Full Name
- Email Address
- Phone Number
- Employee ID
- Department/Title (inferred)

**Data Breach Impact:**
- **Personally Identifiable Information (PII):** 28 individuals compromised
- **Information Type:** Employee contact information and corporate identifiers
- **Exposure Risk:** Potential for identity theft, social engineering, targeted attacks
- **Regulatory Impact:** GDPR, CCPA, and other data protection regulations

---

### Question 12: Persistence Mechanism

**Finding:** Admin account creation - Username: Jellibean

**Evidence:**
- HTTP filter: http.request.method == "POST" && http.request.uri contains "users"
- Packet #20776: POST /identity/v3/users HTTP/1.1
- Following HTTP stream of request
- JSON body contains: "user": {"name": "Jellibean"}

<img width="1103" height="587" alt="image" src="https://github.com/user-attachments/assets/9e11ee26-c76f-4117-9d29-ba984784d0ab" />


**Analysis:**
For persistence, the attacker created a new user account with administrative privileges. The account details:
- **Username:** Jellibean
- **Password:** P@$$word
- **Privileges:** Admin role (confirmed in separate role assignment)

This account enables:
- Future access to the cloud infrastructure
- Bypass of credential rotation after the incident
- Long-term persistence even after discovery

**Technique Classification:**
- MITRE: T1136.001 - Create Account: Local Account (with admin privileges)

---

### Question 13: Persistence Account Password

**Finding:** P@$$word

**Evidence:**
- Same packet #20776 analysis
- JSON body of POST request
- "password": "P@$$word" field in user creation request

<img width="1168" height="622" alt="image" src="https://github.com/user-attachments/assets/25921b99-a44f-4968-b4fc-51467bdad051" />

<img width="1178" height="567" alt="image" src="https://github.com/user-attachments/assets/cf24bba6-6733-4d90-9a81-b3e7b78ac4d3" />


**Analysis:**
The attacker set a simple password for the persistence account. While this password is relatively weak, the account would still provide access to the cloud infrastructure after the initial compromise is discovered and remediated.

---

### Question 14: MITRE ATT&CK Tactic

**Finding:** T1136.001 - Create Account: Local Account

**Evidence:**
- OpenStack terminology: "local account" in context of Keystone users
- Admin privileges confirmed through role assignments
- Attack phase: Persistence

<img width="1106" height="586" alt="image" src="https://github.com/user-attachments/assets/372030d0-f19a-4e94-83e0-3f563e5187bf" />

**Analysis:**
The attacker used the MITRE ATT&CK technique T1136.001 to maintain persistence:
- **Tactic:** Persistence
- **Technique:** T1136 - Create Account
- **Sub-technique:** T1136.001 - Create Account: Local Account
- **Context:** Creating a privileged Keystone user account for future access

---

## Attack Chain Summary

```
1. RECONNAISSANCE
   ├─ Web server enumeration using ffuf v2.1.0
   └─ Subdomain discovery: cloud.vantage.tech

2. EXPLOITATION
   ├─ Dashboard login brute-force (3 failed attempts)
   ├─ Successful authentication to dashboard
   └─ Accessed OpenStack API credentials (openrc file)

3. EXPLORATION
   ├─ Initial API reconnaissance via Keystone
   ├─ Project enumeration and discovery
   ├─ Service catalog retrieval (Swift endpoint)
   └─ Container enumeration (3 containers identified)

4. EXFILTRATION
   ├─ Direct file access to Swift object storage
   └─ Downloaded sensitive data (user-details.csv, 28 records)

5. PERSISTENCE
   ├─ Created admin user account (Jellibean)
   └─ Established backdoor access mechanism
```

---

## Indicators of Compromise (IOCs)

### Network Indicators
| Type | Value | Notes |
|------|-------|-------|
| Source IP | 117.200.21.26 | Attacker IP address |
| Target IP | 157.230.81.229 | Web server IP |
| Domain | cloud.vantage.tech | Target subdomain |
| Service Port | 8080 | Swift endpoint port |

### Service Identifiers
| Service | Details |
|---------|---------|
| Web Fuzzing Tool | ffuf v2.1.0 |
| Auth Service | OpenStack Keystone v3 |
| Storage Service | Swift Object Storage v1 |
| Containers | dev-files, employee-data, user-data |

### User Account Indicators
| Indicator | Value |
|-----------|-------|
| Created Username | Jellibean |
| Account Type | Keystone local user with admin role |
| Purpose | Persistence/backdoor access |

### Data Exfiltration Indicators
| Indicator | Value |
|-----------|-------|
| File Downloaded | user-details.csv |
| Records Compromised | 28 employee records |
| Data Type | PII (names, emails, phone, employee IDs) |
| Download Timestamp | 2025-07-01 09:45:23 UTC |

---

## Root Cause Analysis

### Primary Cause
**Developer Oversight:** Dashboard redirect exposed on web server without access controls

### Contributing Factors

1. **Insufficient Access Controls**
   - No IP whitelisting for dashboard access
   - No rate limiting on login attempts
   - Default or weak credentials in use

2. **Inadequate Monitoring**
   - Web server fuzzing activity not detected
   - Brute force login attempts not alerted
   - Unusual API queries not flagged

3. **Configuration Weaknesses**
   - OpenStack Keystone accepting multiple authentication methods
   - Swift endpoint public-accessible from untrusted networks
   - Default project permissions too permissive

### Secondary Factors

1. **Credential Management**
   - API credentials (openrc) easily accessible via dashboard
   - No credential rotation policies evident
   - Credentials stored in plaintext files

2. **Service Architecture**
   - Multiple layers of cloud services accessible sequentially
   - No network segmentation between dashboard and APIs
   - Service discovery via Keystone catalog

---

## Remediation & Recommendations

### Immediate Actions (Critical Priority)

1. **Revoke Compromised Credentials**
   - Disable Jellibean user account
   - Rotate all API credentials and tokens
   - Reset dashboard login credentials
   - Regenerate openrc files

2. **Access Control Implementation**
   - Implement IP-based access restrictions to dashboard
   - Require multi-factor authentication (MFA)
   - Rate limit login attempts (max 5 attempts/IP/hour)
   - Implement account lockout after 3 failed attempts

3. **Network Segmentation**
   - Restrict Swift endpoint access to internal networks only
   - Isolate API servers from public internet
   - Implement proxy/gateway for API access
   - Apply firewall rules to block direct API access

### Short-term Actions (High Priority)

4. **Monitoring & Logging**
   - Enable comprehensive audit logging for all API calls
   - Monitor for unusual login attempts and API queries
   - Alert on data access patterns (especially Swift downloads)
   - Log all user account creation and privilege changes

5. **Incident Response**
   - Conduct forensic analysis of affected systems
   - Review access logs for 30 days prior to incident
   - Identify any additional persistence mechanisms
   - Notify affected employees (28 individuals) per GDPR/CCPA

6. **Data Protection**
   - Encrypt sensitive data at rest in Swift containers
   - Encrypt data in transit (enforce HTTPS/TLS)
   - Implement data masking for sensitive fields
   - Apply least-privilege principle to data access

### Long-term Actions (Medium Priority)

7. **Architecture Hardening**
   - Implement service mesh for API communication
   - Deploy Web Application Firewall (WAF) for dashboard
   - Segment cloud infrastructure from internet-facing systems
   - Implement API gateway with authentication/rate limiting

8. **Security Practices**
   - Implement security awareness training (phishing, social engineering)
   - Regular penetration testing of exposed services
   - Conduct periodic security audits of cloud configuration
   - Develop incident response procedures

9. **Compliance & Governance**
   - Document security incident for regulatory reporting
   - Update security policies and procedures
   - Implement automated compliance scanning
   - Establish security review board for infrastructure changes

---

## Investigation Methodology

### Tools Used
- **Wireshark:** PCAP file analysis and protocol filtering
- **Tcpdump:** Network traffic capture
- **Grep/Sed:** Text analysis and pattern matching

### Analysis Techniques

1. **Protocol Filtering**
   - Filtered PCAP by HTTP protocol
   - Isolated relevant requests and responses
   - Analyzed request/response pairs for causation

2. **Header Analysis**
   - User-Agent string examination (identified ffuf)
   - HTTP response code analysis (200 OK vs 302 Found)
   - Cookie and token tracking

3. **Traffic Flow Analysis**
   - Sequenced requests chronologically
   - Identified request/response pairs
   - Followed HTTP streams for data extraction

4. **Timestamp Correlation**
   - Extracted UTC timestamps from packet frames
   - Correlates events across two separate PCAP files
   - Reconstructed minute-level attack timeline

---

## Conclusion

This incident demonstrates a sophisticated and opportunistic attack against cloud infrastructure. The attacker exploited a simple developer oversight (exposed dashboard) to gain initial access, then systematically explored the cloud environment to discover sensitive data storage, successfully exfiltrated 28 employee records, and established persistence for future access.

The investigation successfully reconstructed the complete attack timeline, identified all indicators of compromise, and determined the scope and impact of the breach. Immediate remediation should focus on credential revocation, access control implementation, and incident notification to affected individuals.

The findings highlight the critical importance of:
- Access controls for internet-facing services
- Monitoring and alerting for suspicious activity
- Network segmentation in cloud environments
- Regular security audits of cloud infrastructure

---

**Report Prepared By:** Security Incident Investigation Team  
**Date:** March 4, 2026  
**Classification:** Internal Use - Incident Documentation
