# Incident Report: SSH Brute Force Detection

## 1. Summary
On 2026-03-01, repeated failed SSH authentication attempts were detected from a single internal source IP (192.168.180.131). The activity exceeded the defined threshold of 5 failed login attempts within one minute, triggering an automated brute-force detection alert.

## 2. Environment

The lab environment consists of three virtual machines deployed in an isolated network segment:

- **Kali Linux** – attacker simulation host used to generate SSH brute-force traffic.
- **Ubuntu Application Server** – primary log source, running OpenSSH service and hosting the detection script.
- **Ubuntu Log Server** – centralized log collector receiving forwarded logs via rsyslog.

Detection logic is executed locally on the Ubuntu Application Server. 

This host-based detection approach ensures that suspicious activity is identified directly at the source of log generation, reducing dependency on centralized processing. 

Even if log forwarding to the central log server becomes unavailable, detection logic remains operational and alerts are still generated locally. This improves resilience and ensures visibility at the host level.

### Log Flow Architecture

1. SSH authentication events are generated on the Ubuntu Application Server.
2. Logs are stored locally and accessible through journald.
3. A time-based detection script analyzes SSH logs using journald filtering.
4. When threshold conditions are met, an alert is generated using the `logger` utility.
5. Alerts and system logs are forwarded to the centralized log server via rsyslog for aggregation.

## 3. Detection Trigger
The detection mechanism uses:
- Time-based filtering (`journalctl --since "1 minute ago"`)
- Pattern matching for "Failed password"
- Aggregation per source IP
- Threshold > 5 failed attempts within 1 minute
- Alert generation using `logger` with severity `user.err`

## 4. Observed Indicators
- Source IP: 192.168.180.131
- Number of failed attempts: 11
- Time window: 1 minute (time-based filtering using journald)
### Sample log evidence:
Failed authentication example: "Failed password for invalid user kali from 192.168.180.131 port 39954 ssh2"

## 5. Alert Generation
When the threshold was exceeded, the detection script generated an alert:
"ALERT: SSH brute force detected from <IP> (<COUNT> failed attempts in last minute)"

The alert was logged locally and forwarded to the centralized log server.

## 6. Risk Assessment
The source IP address (192.168.180.131) belongs to a private network range, indicating that the activity originated from within the internal lab network. 

In a production environment, internal brute-force activity may suggest:
- A compromised internal host
- Lateral movement attempts
- Misconfigured internal services
- Unauthorized access testing

Internal-origin attacks are often more critical than external attempts, as they indicate potential breach or insider threat.

## 7. Response & Mitigation

In this lab scenario, the detected activity was generated intentionally from a controlled attacker machine (Kali Linux) to simulate brute-force behavior.

In a production environment, the following actions would be recommended:

- Immediate verification of the source IP ownership.
- Temporary blocking of the offending IP via firewall or intrusion prevention system.
- Review of authentication logs for successful login attempts.
- Verification of account lockout policies.
- Enabling or tuning protective mechanisms such as Fail2Ban or rate limiting.
- Reviewing password policy strength and enforcing multi-factor authentication if available.

## 8. Limitations & Improvements
Current implementation uses a fixed time window of one minute for detection. This approach may miss distributed low-and-slow brute-force attempts spanning across time boundaries.

Potential improvements:

- Implementation of sliding window detection logic.
- Stateful tracking of previous failed attempts.
- Integration with a full SIEM platform for correlation across multiple hosts.
- Geolocation enrichment of source IP addresses.
- Automated IP blocking based on repeated violations.
- Integration with centralized alert dashboard.