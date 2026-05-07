# SECURITY REPORT — Nilavan Realtors (lt-nilavan)

# Summary Table

| # | Vulnerability | OWASP Category | Severity | Status |
|---|---------------|----------------|----------|--------|
| 1 | No rate limiting on `/api/sendgrid` | A05 – Security Misconfiguration | High | Fixed |
| 2 | No input sanitization (HTML injection) | A03 – Injection | Medium | Fixed |
| 3 | `.git` directory publicly accessible | A05 – Security Misconfiguration | Critical | Fixed |
| 4 | Missing `X-Frame-Options` (Clickjacking) | A05 – Security Misconfiguration | Medium | Fixed |
| 5 | Application running as root | A04 – Insecure Design | Critical | Fixed |
| 6 | HTTP not redirected to HTTPS | A02 – Cryptographic Failures | Medium | Fixed |



# Vulnerability 1 — No Rate Limiting on `/api/sendgrid`

OWASP Category: A05 – Security Misconfiguration  
Affected File: `app/api/sendgrid/route.ts` (entire POST handler)  
Severity: High

# Description
The `/api/sendgrid` endpoint accepts unlimited POST requests with no throttling, IP-based limits, or CAPTCHA. Any client can call it thousands of times per minute without restriction.

# Business Impact
An attacker can flood the business owner's inbox with thousands of spam emails, exhaust the free SendGrid daily quota (100 emails/day on free tier), and effectively perform a Denial-of-Service against the contact system — all at zero cost to the attacker.

# Proof of Concept

# Send 10 rapid spam requests to the live endpoint
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" \
    -X POST https://yoursite.com/api/sendgrid \
    -H "Content-Type: application/json" \
    -d '{
      "name":"test",
      "email":"x@x.com",
      "phone":"0000000000",
      "message":"spam"
    }'
done


Before fix — all 10 return HTTP 200 (emails sent):

Request 1: HTTP 200
Request 2: HTTP 200
****
Request 10: HTTP 200


After fix — requests 6-10 return HTTP 429:

Request 1: HTTP 200
Request 2: HTTP 200
...
Request 6: HTTP 429
Request 7: HTTP 429

# Recommended Fix
Rate limiting was implemented at Nginx limit_req module as the first line of defence — requests are blocked before reaching Node.js, reducing server load. 

sudo nano /etc/nginx/sites-available/nilavan

limit_req_zone $binary_remote_addr zone=contact_limit:10m rate=5r/m;
location /api/sendgrid {
    limit_req zone=contact_limit burst=2 nodelay;
    limit_req_status 429;


# Vulnerability 2 — No Input Sanitization (HTML / Link Injection)

OWASP Category: A03 – Injection  
Affected File: `app/api/sendgrid/route.ts`, lines 20–45 (HTML email template)  
Severity: Medium

# Description
User-supplied fields (`name`, `email`, `phone`, `message`) are embedded directly into the HTML email body without any escaping or sanitization. An attacker can inject arbitrary HTML tags, including `<a>` links pointing to phishing or malware sites, into emails that appear to come from the legitimate business system.

# Business Impact
The business owner receives an email that appears to be from their own system (NILAVAN REALTORS branding) but contains a malicious link. The owner could be tricked into clicking it, potentially leading to credential theft, malware installation, or reputational damage if forwarded.

# Proof of Concept

# Inject a malicious phishing link into the message field
curl -X POST https://nilavan-mohit.duckdns.org/api/sendgrid \
  -H "Content-Type: application/json" \
  -d '{"name":"Hacker","email":"h@h.com","phone":"1234567890","message":"<a href=\"http://evil.com\">Claim your reward</a>"}'
  

Result before fix: The email received by the business owner contains a clickable phishing link rendered inside the official Nilavan Realtors branded email template.

Result after fix: The injected HTML is escaped — the email shows the literal text `&lt;a href=...&gt;` as plain text, not a rendered link.

# Recommended Fix

function sanitizeInput(input: string): string {
  return input
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/\//g, '&#x2F;');
}

// Apply before embedding in HTML:
const safeName    = sanitizeInput(name);
const safeMessage = sanitizeInput(message);
// ... use safeName, safeMessage in the HTML template


# Vulnerability 3 — `.git` Directory Publicly Accessible

OWASP Category: A05 – Security Misconfiguration  
Affected File: Nginx server configuration (missing deny rule)  
Severity: Critical

# Description
The `.git` directory was not blocked in the Nginx configuration. When a Next.js app is deployed by cloning a git repository, the `.git` folder exists on disk. Without explicit blocking, Nginx serves it publicly. An attacker can download the entire git history, including environment files, secrets, and API keys that may have been committed at any point.

# Business Impact
- Full source code disclosure
- Exposure of `.env` files if ever committed (contains `SENDGRID_API_KEY`)
- Enumeration of all file paths, developer names, emails from commit history
- Reconstruction of deleted secrets from git history

After fix — returns 404:

curl -I https://nilavan-mohit.duckdns.org/.git/config
HTTP/2 404

# Recommended Fix
Added to `/etc/nginx/sites-available/nilavan`:

location ~ /\.git {
    deny all;
    return 404;
}



# Vulnerability 4 — Missing `X-Frame-Options` Header (Clickjacking)

OWASP Category: A05 – Security Misconfiguration  
Affected File: Nginx server configuration (missing security headers)  
Severity: Medium

# Description
No `X-Frame-Options` or `Content-Security-Policy: frame-ancestors` header is set. This means any website can embed the Nilavan Realtors site in an `<iframe>`, overlaying invisible UI elements to trick users into clicking buttons they did not intend to click (e.g. submitting their contact details to an attacker's endpoint instead).

# Business Impact
An attacker can embed the Nilavan site inside their own page with an invisible overlay. A user thinking they're clicking "Get in Touch" on the real site is actually clicking something else on the attacker's page. This can lead to credential theft or unwanted form submissions.

# Proof of Concept
Create this HTML file and open it in any browser:

<!-- attacker-page.html -->
<html>
  <body style="margin:0">
    <h1 style="color:red">WIN A FREE PROPERTY! Click the button below!</h1>
    <!-- The real Nilavan site, invisible, positioned over the fake button -->
    <iframe 
      src="https://nilavan-mohit.duckdns.org" 
      style="opacity:0.01; position:absolute; top:0; left:0; width:100%; height:100%; z-index:999;">
    </iframe>
  </body>
</html>


Before fix: The iframe loads successfully. The attacker page renders.  
After fix: Browser refuses to render the iframe, shows blank frame.

# Recommended Fix
Added to Nginx config inside `location /`:

add_header X-Frame-Options "DENY";
add_header Content-Security-Policy "frame-ancestors 'none'";



# Vulnerability 5 — Application Running as Root

**OWASP Category:** A04 – Insecure Design  
**Affected File:** Server process configuration  
**Severity:** Critical

# Description
If the Next.js application is started by the root user (common mistake when deploying as default `ubuntu` or `root`), any Remote Code Execution (RCE) vulnerability in the application, Node.js runtime, or any npm dependency gives an attacker full root access to the server — there is no privilege boundary to limit damage.

# Business Impact
With root-level access an attacker can:
- Read `/etc/shadow` (password hashes for all users)
- Install persistent backdoors or rootkits
- Read all environment variables and secrets across all processes
- Pivot to other AWS services using the EC2 instance role credentials from the metadata endpoint (`http://169.254.169.254/latest/meta-data/`)
- Destroy or encrypt all data (ransomware)
- Use the server as a launching pad for attacks on other systems

# Proof of Concept (impact demonstration)

# If app runs as root and RCE is achieved via a dependency vulnerability:
# Attacker can read all secrets:
cat /etc/shadow
cat /home/deploy/.env.local   # SendGrid API key

# Attacker can access AWS metadata service for IAM credentials:
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Attacker can create a persistent backdoor:
echo "attacker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers


# Recommended Fix

# Create a dedicated non-root user
sudo adduser mohit
sudo usermod -aG sudo mohit

# Always start PM2 as the deploy user, never root
su - mohit
pm2 start npm --name "nilavan" -- start
pm2 startup
pm2 save

# Verify the process owner
ps aux | grep node




# Vulnerability 6 — HTTP Traffic Not Redirected to HTTPS

OWASP Category: A02 – Cryptographic Failures  
Affected File: Nginx server configuration  
Severity: Medium

# Description
Before SSL was configured, all traffic travelled in plaintext over HTTP. Even after SSL is obtained, without an explicit redirect rule, users who visit `http://nilavan-mohit.duckdns.org` receive an unencrypted response. This exposes all form submissions (name, email, phone, message) to network-level interception.

# Business Impact
Any attacker on the same network (coffee shop Wi-Fi, ISP-level), or any man-in-the-middle, can read all contact form submissions in plaintext, including customer PII (phone numbers, email addresses). They can also inject content into the HTTP response.

# After fix — HTTP 301 redirect to HTTPS
curl -I http://nilavan-mohit.duckdns.org
# HTTP/1.1 301 Moved Permanently
# Location: https://nilavan-mohit.duckdns.org

# Recommended Fix
Certbot automatically added this block to Nginx:

server {
    listen 80;
    server_name nilavan-mohit.duckdns.org;
    return 301 https://$host$request_uri;
}

Additionally, HSTS header forces browsers to always use HTTPS on future visits:

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";


# Threat Scenario Responses

# Scenario 1: "I want to flood the business inbox with thousands of spam emails"

How the attack works: The `/api/sendgrid` route has no authentication, rate limiting, or CAPTCHA. A simple shell loop or Python script can POST to it indefinitely, triggering a real email for every request.

Attack command:

for i in $(seq 1 100); do
  curl -s -X POST https://nilavan-mohit.duckdns.org/api/sendgrid \
    -H "Content-Type: application/json" \
    -d '{"name":"Bot","email":"x@x.com","phone":"0000000000","message":"SPAM"}'
done

Fix applied: IP-based rate limiter — max 7 requests per minute. Requests beyond limit return `HTTP 429 Too Many Requests`.



# Scenario 2: "I want to inject a malicious link into an email that appears to come from the business"

How the attack works: The `message` field is embedded unsanitized into the HTML email template. Any HTML tags in the input are rendered by the email client. An attacker sends `<a href="http://evil.com">Click here</a>` as the message, which appears as a styled, clickable link inside the official Nilavan email.

Attack command:

curl -X POST https://nilavan-mohit.duckdns.org/api/sendgrid \
  -H "Content-Type: application/json" \
  -d '{"name":"Hacker","email":"h@h.com","phone":"1234567890","message":"<a href=\"http://evil.com\">Claim your reward</a>"}'


Fix applied, All user inputs are HTML-escaped before embedding. `<` becomes `&lt;`, `>` becomes `&gt;`, links render as plain text.



# Scenario 3: "I want to access the full source code from the browser without credentials"

How the attack works: The `.git` directory exists on disk after `git clone`. Without a deny rule in Nginx, `GET /.git/config` returns the git configuration, and tools like `git-dumper` can reconstruct the full repository from the object store.

Attack command:

curl https://nilavan-mohit.duckdns.org/.git/config


Fix applied: Nginx `location ~ /\.git { deny all; return 404; }` — all `.git` paths return 404.

# Scenario 4: "I want to embed this website inside my malicious site to trick users (clickjacking)"

How the attack works: Without `X-Frame-Options`, any page can embed the site in an `<iframe>`. The attacker overlays invisible elements on top of the real site's buttons. Users think they're interacting with the real site but are actually triggering actions on the attacker's page.

Attack PoC:

<iframe src="https://nilavan-mohit.duckdns.org" style="opacity:0.01; position:absolute; top:0; left:0; width:100%; height:100%; z-index:999;"></iframe>

Fix applied: `add_header X-Frame-Options "DENY"` in Nginx — browsers refuse to render the iframe.



# Scenario 5: "I gained access to the server — how did running the app as root make things worse?"

How it worsens impact:
Running as root removes the operating system's primary defence — process isolation. Normally, a compromised web process can only access files owned by its user. As root, it can read `/etc/shadow`, access all users' files, modify system binaries, install backdoors, and access AWS instance metadata for IAM credentials — turning a limited application vulnerability into full infrastructure compromise.

Fix applied: Application runs as `deploy` (non-root, limited sudo). Even if exploited, the attacker's blast radius is confined to that user's permissions only.



# Additional Security Headers Applied

| `X-Frame-Options` | `DENY` | Prevents clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Forces HTTPS on all future visits |
| `Content-Security-Policy` | `frame-ancestors 'none'` | Modern clickjacking protection |


