<h1>Penetration Testing Report – ToolsRUs</h1>

<h2>TL;DR – Quick Overview</h2>
<p>
This repository documents an authorized, lab-based penetration test of the ToolsRUs lab available on the TryHackMe platform.
The assessment identified weak authentication controls, exposed internal usernames,
and an outdated Apache Tomcat service. These issues were chained to achieve
<strong>authenticated remote code execution</strong> through the use of Metasploit framework.
</p>

<p>
Skills demonstrated include network and web enumeration, credential attacks,
service pivoting, exploitation, and post-exploitation validation.
</p>

<p>
All testing was performed in a safe, legal, and authorized lab environment
for educational purposes.
</p>

<hr>

<h2>Target Information</h2>
<ul>
  <li><strong>Target:</strong> ToolsRUs (TryHackMe Lab)</li>
  <li><strong>Target IP:</strong> 10.48.184.134</li>
  <li><strong>Date:</strong> January 2026</li>
  <li><strong>Environment:</strong> Authorized CTF / Lab Environment (TryHackMe)</li>
</ul>

<hr>

<h2>Executive Summary</h2>
<p>
This penetration test was conducted against the <em>ToolsRUs</em> application
as part of an authorized TryHackMe laboratory environment. The objective of
the assessment was to identify security weaknesses, demonstrate realistic
attack paths, and assess the potential impact of discovered vulnerabilities.
</p>

<p>
The assessment resulted in <strong>full remote code execution</strong> on the
target system. The compromise was achieved through a combination of
<strong>information disclosure</strong>, <strong>weak authentication controls</strong>,
and <strong>insecure configuration of an Apache Tomcat application server</strong>.
</p>

<p>
No zero-day exploits were required. All findings stemmed from common real-world
misconfigurations frequently observed in production environments.
</p>

<hr>

<h2>Scope and Methodology</h2>

<h3>Scope</h3>
<ul>
  <li>Single target host: <code>10.48.184.134</code></li>
  <li>Black-box testing approach</li>
  <li>No credentials provided prior to testing</li>
</ul>

<h3>Methodology</h3>
<p>
The assessment followed a standard penetration testing lifecycle:
</p>
<ol>
  <li>Network enumeration</li>
  <li>Web application enumeration</li>
  <li>Authentication analysis</li>
  <li>Credential attacks</li>
  <li>Service pivoting</li>
  <li>Remote code execution</li>
  <li>Post-exploitation validation</li>
</ol>

<hr>

<h2>Technical Findings</h2>

<h3>1. Network Enumeration</h3>
<p>
Initial reconnaissance was conducted to identify exposed services and establish
an overview of the external attack surface. The scan revealed multiple accessible
services, including SSH, an Apache web server, and an Apache Tomcat instance
running on a non-standard port.
<pre><code>nmap -Pn -p- --open --min-rate 300 --max-retries 2 10.48.184.134 </code> </pre>
</p>

<p>
The further conducted service-focused scan indicated that several services were outdated, increasing the likelihood of
known vulnerabilities and misconfigurations. The presence of a management
service on a non-standard port suggested service obfuscation rather than
proper access control.
<pre><code>nmap -sV -sC -p 22,80,1234,8009 10.48.184.134 </code></pre>
<img src="https://i.imgur.com/erRPpMX.png" height="90%" width="90%" alt="Nmap Scan Results"/>
</p>

<hr>

<h3>2. Web Application Enumeration and Information Disclosure</h3>
<p>
Directory enumeration,
<pre><code>gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js -t 50 --status-codes 200,204,301,302,307,401,403 --status-codes-blacklist "" --timeout 10s -u 10.48.184.134 </code></pre>
<img src="https://i.imgur.com/nvXkcuF.png" height="95%" width="95%" alt="Gobuster directory enumeration">

followed by the manual inspection of the web application revealed
an internal message exposed to unauthenticated users. The message referenced
an internal user named <strong>bob</strong> and implied that an Apache Tomcat
backend had not been updated.

<img src="https://i.imgur.com/8PfPxgY.png" height="90%" width="90%" alt=Web app username disclose>
</p>

<p>
This disclosure provided the following intelligence:
</p>
<ul>
  <li>A valid internal username</li>
  <li>Confirmation of an outdated backend service</li>
</ul>

<hr>

<h3>3. Weak Authentication on Protected Resource</h3>
<p>
The previously identified <code>/protected</code> endpoint relied on <strong>HTTP Basic Authentication</strong>
over unencrypted HTTP. This was confirmed using the curl command-line tool. 
</p>

<img src="https://i.imgur.com/0aNj96j.png" height="90%" width="90%" alt=Browser display>
<p>
<pre><code>curl -I http://10.48.184.134:80/protected </code></pre>
</p>
<img src="https://i.imgur.com/q6ccHXh.png" height="65%" width="65%" alt=curl basic auth confirmation>

<p>
No rate limiting, monitoring, or account lockout mechanisms were observed.

This authentication method substantially lowered the barrier for credential-based attacks and increased the likelihood of successful attack.
</p>

<hr>

<h3>4. Credential Brute Force Attack</h3>
<p>
Using the previously disclosed username, a credential attack was conducted against the <code>/protected</code> endpoint.
<pre><code>hydra -l bob -P /usr/share/wordlists/rockyou.txt 10.48.184.134 -s 80 http-get /protected</code></pre>
</p>

<img src="https://i.imgur.com/PKsC5VD.png" height="90%" width="90%" alt=Hydra creds bruteforce>

<p><strong>Valid credentials obtained:</strong></p>
<ul>
  <li>Username: <code>bob</code></li>
  <li>Password: <code>bubbles</code></li>
</ul>

<p>
The password was present in a common password wordlist, indicating weak password policy and poor credential hygiene.
</p>

<hr>

<h3>5. Service Pivot to Apache Tomcat</h3>
<p>
Following authentication, the application indicated that the protected
resource had moved to a different port. This behavior suggested service
segmentation rather than remediation.
</p>

<img src="https://i.imgur.com/ozPCRVS.png" height="90%" width="90%" alt=Tomcat protected move>

<p>
Further enumeration identified an exposed Apache Tomcat Manager interface running on
port <code>1234</code>. The service was fingerprinted as <strong>Apache Tomcat 7</strong>,
an end-of-life and unsupported version.
</p>

<p>

Enumeration revealed:

<ul>
  <li>Exposed management interfaces</li>
  <li>Dangerous HTTP methods (PUT, DELETE)</li>
  <li>Default example applications enabled</li>
</ul>
<pre><code>nikto -h 10.48.184.134:1234</code></pre>
</p>

<img src="https://i.imgur.com/Rb1XMJE.png" height="90%" width="90%" alt=Tomcat manager enumeration >


<hr>

<h3>6. Apache Tomcat Manager Exposure &amp; Exploit Selection Reasoning</h3>
<p>
Analysis of service behavior and HTTP headers identified the backend as
Apache Tomcat (Apache-Coyote/1.1). Additional enumeration confirmed that the
Tomcat Manager interface was exposed at:
</p>

<pre><code>http://10.48.184.134:1234/manager/html</code></pre>

<p>
Given that valid credentials were already obtained, the Tomcat version was
outdated, and the Manager interface was externally accessible, exploitation
paths focusing on authenticated functionality were prioritized.
<code>search tomcat manager</code>
</p>

<img src="https://i.imgur.com/YCzA099.png" height="90%" width="90%" alt=Metasploit exploit selection>

<p>
The Metasploit module <code>exploit/multi/http/tomcat_mgr_upload</code> was selected
because it:
</p>
<ul>
  <li>Leverages legitimate Tomcat Manager functionality</li>
  <li>Requires valid credentials, matching the identified weakness</li>
  <li>Abuses WAR file deployment, a realistic attack vector</li>
  <li>Provides stable and repeatable remote code execution</li>
</ul>

<img src="https://i.imgur.com/GE9K8Lj.png" height="90%" wight="90%" alt=Metsploit exploit setup>

<hr>

<h3>7. Remote Code Execution (Critical)</h3>
<p>
Authenticated access to the Apache Tomcat Manager interface permitted
the deployment and execution of a malicious WAR file,
resulting in a reverse Meterpreter session from the target system.
</p>

<p><strong>Result:</strong> Remote Code Execution achieved.</p>

<img src="https://i.imgur.com/wPGXPli.png" height="90%" width="90%" alt=Meterpreter shell achieved>

<hr>

<h2>Post-Exploitation Validation</h2>
<p>
Basic system enumeration confirmed the level of access obtained:
<pre><code>sysinfo</code></pre>
<pre><code>getuid</code></pre>
</p>

<img src="https://i.imgur.com/5Fz9lOs.png" height="65%" width="65%" alt=Meterpreter system enumeration>

<ul>
  <li><strong>Operating System:</strong> Linux 4.4.0-1075-aws</li>
  <li><strong>Architecture:</strong> x64</li>
  <li><strong>Access Level:</strong> Arbitrary command execution</li>
</ul>

<p>
Inspection of the following configuration file revealed plaintext credentials:
</p>
<pre><code>search -f tomcat-users.xml</code></pre>
<pre><code>/usr/local/tomcat7/conf/tomcat-users.xml</code></pre>
<img src="https://i.imgur.com/AZhwHd1.png" height="90%" width="90%" alt= Tomcat-users file discovery>
<img src="https://i.imgur.com/IbD64fX.png" height="90%" width="90%" alt= Tomcat-users file output>
<pre><code>&lt;user name="bob" password="bubbles" roles="admin-gui,manager-gui" /&gt;</code></pre>

<p>
This confirmed that the compromised account was a high-privilege Tomcat
administrator and that credentials were stored insecurely.
</p>

<hr>

<h2>Impact Assessment</h2>
<ul>
  <li><strong>Severity:</strong> Critical</li>
  <li><strong>Category:</strong> Authenticated Remote Code Execution</li>
  <li><strong>Estimated CVSS:</strong> 9.8</li>
</ul>

<hr>

<h2>Recommendations</h2>
<ul>
  <li>Remove internal information from public-facing pages</li>
  <li>Enforce strong, unique passwords</li>
  <li>Prevent credential reuse across services</li>
  <li>Disable HTTP Basic Authentication or enforce HTTPS</li>
  <li>Restrict access to the Tomcat Manager interface</li>
  <li>Upgrade Apache Tomcat to a supported version</li>
  <li>Remove default examples and documentation</li>
  <li>Never store credentials in plaintext</li>
</ul>

<hr>

<h2>Disclaimer</h2>
<p>
This penetration test was conducted exclusively within an
<strong>authorized TryHackMe laboratory environment</strong>.
All activities were performed for educational purposes only.
</p>
