1. Approach to Finding Vulnerabilities (Tools & Methodology)
My approach to securing this Next.js application was rooted in the OWASP Top 10 2021 framework, combining static code analysis with dynamic testing to validate real-world impact.

Static Code Review: I started by reading the application source code, specifically focusing on server-side handlers like app/api/sendgrid/route.ts. By tracing the data flow from the user's POST request to the SendGrid API execution, I identified the lack of input sanitization and missing application-level rate limits.

Dynamic Testing: I deployed the application to a live staging environment and acted as an external attacker. Using curl, I scripted rapid sequential payloads to prove the inbox-flooding vulnerability. I also utilized browser DevTools and manual payload injection to verify the HTML/XSS email injection.

Infrastructure Auditing: I reviewed the Nginx and system configurations, actively attempting to bypass intended routing. I utilized wget --mirror and curl to prove that the .git directory was exposed and verified the absence of protective HTTP headers (like X-Frame-Options).

2. Server Setup Decisions
My infrastructure setup was guided by the Principle of Least Privilege and Defense in Depth.

Non-Root Execution: A dedicated mohit user was created to run the PM2 process. Node.js applications should never run as root; isolating the process ensures that if an RCE vulnerability is discovered in an npm package, the attacker's blast radius is strictly confined to that unprivileged user.

SSH Hardening & UFW: The default SSH port was moved to 2222, and password authentication was completely disabled in favor of RSA/ED25519 keys. The UFW firewall was configured in a default-deny state, explicitly allowing only necessary ports. This drastically reduces the noise from automated botnet scanners.

Nginx as a Reverse Proxy & Shield: Nginx was chosen not just to route traffic to Node.js, but to act as a primary security layer. By handling SSL termination, injecting strict security headers (HSTS, CSP, X-Frame-Options), and explicitly returning 404 for hidden files like .git, Nginx protects the Next.js app from malformed requests before they ever reach the Node runtime.

3. Fix Implementations & Trade-offs
In security, every fix involves balancing risk, performance, and development overhead. Here are the trade-offs for the implemented solutions:

Rate Limiting (Nginx vs. Application): Implementation: I enforced rate limiting at the Nginx level (limit_req_zone) rather than strictly in the Node.js code.

Trade-off: Nginx drops malicious traffic at the edge, saving CPU/memory on the Node server. However, it is IP-based. If multiple legitimate users share a single NAT/Corporate IP, they might get falsely rate-limited. An application-level Redis-backed rate limiter would allow for more granular, user-session-based throttling but requires more infrastructure overhead.

Blocking .git via Reverse Proxy: Implementation: Added a location ~ /\.git { deny all; } block in Nginx.
