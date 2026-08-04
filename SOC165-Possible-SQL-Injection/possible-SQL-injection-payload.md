# SOC165 - Possible SQL Injection Payload Detected

> LetsDefend SOC Analyst Investigation Walkthrough

# Scenario

This investigation analyzes **SOC165 – Possible SQL Injection Payload Detected** on the LetsDefend platform.

Rather than immediately accepting the alert as malicious, the objective is to validate the detection by following a structured investigation process. Throughout the analysis, the alert is verified by collecting threat intelligence, examining HTTP requests, comparing application responses and determining whether the attacker successfully exploited the target.


# Initial Alert Review

Every investigation begins with understanding **why the alert was generated**.

The detection rule indicates that a request contained a well-known SQL Injection pattern. At this stage, however, the alert only confirms that a suspicious payload was observed—it does not confirm whether an attack actually occurred or whether the application was compromised.

The alert provides the following context:

- **Rule:** SOC165 – Possible SQL Injection Payload Detected
- **Source IP:** 167.99.169.17
- **Destination Host:** WebServer1001
- **Destination IP:** 172.16.17.18
- **Protocol:** HTTPS (443)
- **HTTP Method:** GET
- **Alert Trigger:** Requested URL contains `OR 1=1`

This information establishes the starting point of the investigation but is not sufficient to classify the alert as either a true or false positive.

![Alert Overview](images/01-alert-overview.png)


# Collecting Initial Intelligence

Before inspecting the HTTP traffic, it is important to understand who initiated the connection.

Threat intelligence provides valuable context by revealing whether the source has previously been associated with malicious activity. Although reputation alone should never be used as the sole indicator of compromise, it helps prioritize the investigation.

The source IP (**167.99.169.17**) was therefore investigated using **VirusTotal**.

The lookup shows that the address belongs to **DigitalOcean LLC**, while several security vendors classify it as **Malicious** or **Suspicious**.

This finding increases confidence that the activity deserves further investigation but does not yet prove that the attack succeeded.

![VirusTotal Overview](images/02-virustotal-overview.png)

Additional information confirms the Autonomous System ownership and network allocation.

![VirusTotal Details](images/03-virustotal-details.png)


# Reconstructing the Attacker's Activity

Looking at a single alert rarely provides the full picture.

Attackers usually send multiple requests while probing an application, testing different payloads until they discover a vulnerable input. Because of this, expanding the investigation to all events originating from the same source IP helps reconstruct the attack sequence instead of evaluating one isolated request.

Using LetsDefend Log Management, the investigation was expanded with the following search:

```text
Source Address contains "167.99.169.17"
```

Several HTTP GET requests targeting the **/search** endpoint were identified.

The observed payloads include:

```text
'
```

```text
' OR '1
```

```text
' OR 'x'='x
```

```text
1' ORDER BY 3--
```

```text
" OR 1=1 --
```

These are well-known SQL Injection payloads commonly used to test whether user input is directly interpreted by a backend SQL query.

Instead of repeatedly sending identical requests, the attacker systematically tested multiple SQL Injection techniques. This behavior suggests an active attempt to identify a vulnerable query rather than normal application usage.

![Source IP Investigation](images/04-log-search-source-ip.png)


# Examining the HTTP Responses

Finding malicious payloads is only part of the investigation.

The next objective is determining **how the application responded** to those payloads.

Server responses often reveal whether the payload reached the backend successfully or whether the application rejected the request.

A comparison between legitimate and malicious requests shows a clear pattern.

The normal homepage request returned:

- HTTP Status: **200 OK**
- Response Size: **3547 bytes**

Every SQL Injection request returned:

- HTTP Status: **500 Internal Server Error**
- Response Size: **948 bytes**

The consistency of these responses is significant.

If the SQL Injection had succeeded, differences in response sizes, successful responses, unexpected content, or evidence of backend interaction would generally be expected.

Instead, every payload produced the same internal server error.

From an investigation perspective, this indicates that although the payloads reached the application, there is no evidence that the backend executed the injected SQL statements successfully.

![HTTP Response Analysis](images/05-http-request-details.png)


# Comparing Against Legitimate Traffic

Understanding malicious behavior also requires understanding what normal behavior looks like.

To verify whether these requests were unique, additional traffic targeting the same web server was reviewed.

```text
Destination Address contains "172.16.17.18"
```

The comparison shows that other external clients simply requested the application's homepage and consistently received normal **HTTP 200** responses.

Only **167.99.169.17** repeatedly targeted the **/search** endpoint while sending SQL Injection payloads.

This comparison clearly separates legitimate client activity from attacker behavior and strengthens confidence that the alert represents malicious traffic.

![Destination Traffic Comparison](images/06-log-search-destination.png)


# Determining the Outcome

At this stage, enough evidence has been collected to answer the questions defined by the investigation playbook.

The investigation confirms that:

- Multiple SQL Injection payloads were observed.
- The traffic originated from an external IP address.
- The payloads specifically targeted the application's search functionality.
- The source IP has a suspicious reputation.
- Every malicious request consistently returned HTTP 500 responses.
- No evidence of successful exploitation or command execution was identified.
- No indication of a planned penetration test or attack simulation was found.

Taken together, these findings demonstrate that the activity was genuinely malicious while also indicating that the exploitation attempt was unsuccessful.


# Investigation Results

| Question | Answer |
|-----------|--------|
| Alert Classification | ✅ True Positive |
| Attack Type | SQL Injection |
| Traffic Direction | Internet → Company Network |
| Planned Test | No |
| Attack Successful | No |
| Tier 2 Escalation Required | No |

Since the attack originated from the Internet and no successful compromise was identified, escalation to Tier 2 was not necessary according to the investigation playbook.

![Final Results](images/07-final-results.png)


# Analyst Note

A True Positive SQL Injection attempt targeting **WebServer1001 (172.16.17.18)** was identified from the external IP address **167.99.169.17**.

Log analysis revealed multiple SQL Injection payloads targeting the `/search` endpoint. HTTP responses consistently returned **500 Internal Server Error** with identical response sizes, indicating unsuccessful exploitation. No evidence of command execution, backend compromise or post-exploitation activity was identified.

The malicious activity originated from the Internet, was not part of a planned security assessment and did not require Tier 2 escalation.


# Conclusion

This investigation demonstrates why security alerts should never be evaluated based solely on the detection rule.

Although the alert correctly identified a suspicious SQL Injection payload, the investigation required multiple sources of evidence—including threat intelligence, log correlation, HTTP response analysis and behavioral comparison—to determine the true impact of the activity.

Following a structured investigation methodology made it possible to distinguish a genuine attack attempt from a successful compromise, ultimately confirming the alert as a **True Positive** while concluding that the attack itself was unsuccessful.
