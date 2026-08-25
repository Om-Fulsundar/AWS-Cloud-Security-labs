# AWS Cloud Security Lab Report
**Lab 6: AWS Security Hub Unified Dashboard - Enablement, Posture Review & Custom Insights**  
**Prepared by:** Om Fulsundar (August 2026)

---

| | |
|---|---|
| **Date** | August 24, 2026 |
| **Platform** | Amazon Web Services (AWS) |
| **Region** | eu-north-1 (Europe - Stockholm) |
| **Account ID** | 712934828848 |
| **Lab Type** | Cloud Security Posture Management (CSPM) / SOC Dashboard Build |
| **Services Enabled** | AWS Security Hub CSPM, Amazon GuardDuty, AWS Config, AWS CloudTrail, IAM Access Analyzer |
| **Scope** | Standalone lab, no dependency on previous labs |

---

## 1. Lab Overview

Lab 5 was about chasing one compromised instance. This lab is the opposite exercise: instead of investigating a single host, I stood up AWS Security Hub as a unified posture dashboard that pulls findings from GuardDuty, AWS Config, and IAM Access Analyzer into one place and scores the account against two compliance benchmarks. This is the tool a SOC actually opens first thing in the morning, before drilling into any one alert.

I built the supporting services from scratch in a fresh region (eu-north-1, Stockholm) rather than reusing the us-east-1 setup from Lab 5, so the account still carried some leftover CloudWatch policy resources from that earlier trail in the global findings count. That is expected and I have called it out where it shows up in the screenshots rather than pretending the account was pristine.

> **Real-world relevance:** the AWS console for Security Hub has moved on since this lab script was written. What used to be a manual flow (enable Security Hub, then accept findings from each service one by one) is now bundled into Security Hub CSPM, which auto-links GuardDuty, Config, Inspector, and Macie coverage as soon as it is turned on. This report documents what the console actually did, not what an older script assumed it would do.

---

## 2. Objectives

Going into this lab, here is what I set out to accomplish:

- Build the supporting detection and audit stack from scratch: CloudTrail, GuardDuty, AWS Config, and IAM Access Analyzer, with no dependency on the Lab 5 environment.
- Enable AWS Security Hub CSPM and confirm it aggregates findings from the connected services automatically.
- Watch the compliance score populate over time against two standards: AWS Foundational Security Best Practices (FSBP) v1.0.0 and CIS AWS Foundations Benchmark v1.2.0.
- Drill into individual controls to see how Security Hub reports control status, severity, and check history.
- Cross-check GuardDuty's own findings console against what Security Hub aggregates, to understand how the two consoles relate.
- Build three custom Insights that group findings the way a SOC analyst would triage them: by severity and resource, by product, and by workflow status.

---

## 3. Tools and Services Used

| Tool / Service | Purpose in This Lab |
|---|---|
| AWS Security Hub CSPM | Central posture dashboard; aggregates findings and control checks from GuardDuty, Config, and Access Analyzer, and scores the account against FSBP and CIS. |
| Amazon GuardDuty | Threat detection; enabled with Foundational GuardDuty plus the AI Protection and S3 Protection trial add-ons. |
| AWS Config | Continuous resource recording, feeding compliance checks (all resource types, continuous recording). |
| AWS CloudTrail | Multi-region audit trail (lab6-trail) capturing management events for the account. |
| IAM Access Analyzer | Account-level external access analyzer (lab6-analyzer), feeding findings into Security Hub. |
| Custom Insights | Saved Security Hub filter-and-group views for repeatable triage, built in Part 5 of the walkthrough. |

---

## 4. Lab Environment Setup

The entire lab ran on a live AWS account (Account ID: 712934828848) in the eu-north-1 (Stockholm) region. There was no sandbox or emulated environment - everything was built on real AWS infrastructure, in a clean region with no pre-existing resources besides what earlier labs had left behind in the account globally.

### Setup 1 - CloudTrail Trail (lab6-trail)

I created a fresh multi-region trail named lab6-trail, logging to a new S3 bucket (cloudtrail-lab6-om) and streaming to CloudWatch Logs group lab6-cloudtrail-logs. The console confirmed the trail active with Status: Logging, ARN arn:aws:cloudtrail:eu-north-1:712934828848:trail/lab6-trail. Trail Insights was left disabled since anomaly detection on management events was not needed for this lab.

**[ SS-Setup1 - CloudTrail Trail Created ]**  

<img width="1917" height="457" alt="Screenshot 2026-08-24 183553" src="https://github.com/user-attachments/assets/67a51e28-f157-4d3a-a754-2aded192f236" />


*CloudTrail Trails list showing lab6-trail, Home region Europe (Stockholm), Multi-region trail: Yes, S3 bucket: cloudtrail-lab6-om, CloudWatch Logs log group: lab6-cloudtrail-logs, Status: Logging*

### Setup 2 - GuardDuty Protection Plans

GuardDuty was enabled with Foundational GuardDuty active by default, plus two 30-day trial add-ons: AI Protection (detects threats targeting AI applications) and S3 Protection (monitors S3 data events). These trial protection plans are exactly what let GuardDuty surface the AttackSequence and Runtime findings seen later in this lab.

**[ SS-Setup2 - GuardDuty Protection Plans Active ]**  


<img width="1901" height="858" alt="Screenshot 2026-08-24 183742" src="https://github.com/user-attachments/assets/adb6ab2f-3089-4330-9142-2080013da80f" />


*GuardDuty Protection Plans console: Foundational GuardDuty Enabled, AI Protection Enabled (free trial), S3 Protection Enabled (free trial)*

### Setup 3 - AWS Config

AWS Config was set up with a customer-managed recorder, recording all resource types continuously, delivering to S3 bucket config-lab6-om. The settings page confirmed Recording is on, with 506 resource types on default settings and 4 carrying override settings. Config takes a while to start evaluating rules against real resources, which matters later when the standards pages show No Data instead of Passed or Failed.

**[ SS-Setup3 - AWS Config Enabled ]**  


<img width="1917" height="907" alt="Screenshot 2026-08-24 185101" src="https://github.com/user-attachments/assets/23ba5b9b-684f-4073-b0a0-223aa8c5149f" />


*AWS Config Settings page: recorder status Recording is on, S3 bucket config-lab6-om, recording strategy: all resource types with customizable overrides, continuous frequency*

### Setup 4 - IAM Access Analyzer

I created an account-level analyzer named lab6-analyzer with the zone of trust set to the current account. It returned zero findings immediately, which is the expected result for a fresh analyzer with no externally shared resources to flag yet.

**[ SS-Setup4 - IAM Access Analyzer Created ]**  

<img width="1917" height="588" alt="Screenshot 2026-08-24 185300" src="https://github.com/user-attachments/assets/c8ff2ea3-7e6c-46be-b464-2e2fe2b2b84d" />


*IAM Access Analyzer, External access page: analyzer lab6-analyzer created, zone of trust Current account (712934828848), Findings (0)*

---

## 5. Step-by-Step Walkthrough

### Phase 1 - Enabling Security Hub CSPM

With the four supporting services running, I enabled Security Hub. The very first look at the Summary page, taken within seconds of enabling, showed the dashboard essentially empty: 0 threats, 0 exposures, 0 resources, and only 3 findings recorded so far. That near-empty state is worth keeping as a reference point, because every later screenshot in this report is really just watching that number grow as GuardDuty, Config, and Access Analyzer feed data in.

**[ SS1 - Security Hub Enabled, Initial State ]**  


<img width="1917" height="915" alt="Screenshot 2026-08-24 190009" src="https://github.com/user-attachments/assets/bb661a08-810a-4814-a997-8f31a7f1975e" />


*Security Hub CSPM Summary immediately after enabling: Trends overview showing Threats 0, Exposure 0, Resources 0, All findings 3*

A separate confirmation banner on the General settings page laid out exactly what got deployed: threat analytics, vulnerability management, posture management, and network reachability scanning, all in one enablement step. The Security coverage panel broke this down by underlying service - Amazon GuardDuty at 100% (threat analytics), AWS Security Hub CSPM itself at 100% (posture management), Amazon Inspector at 75% (vulnerability management), and Amazon Macie at 0% (sensitive data discovery, which I did not turn on for this lab). This confirms Security Hub CSPM now bundles services that used to require separate manual setup, which is the console change I flagged in the overview.

**[ SS2 - Security Hub Coverage After Enablement ]**  


<img width="1917" height="911" alt="Screenshot 2026-08-24 221346" src="https://github.com/user-attachments/assets/bb0abbbf-6532-4f80-8289-d19a45903259" />


*Security Hub General settings: Successfully enabled Security Hub banner, Security coverage panel showing GuardDuty 100%, Security Hub CSPM 100%, Inspector 75%, Macie 0%*

### Phase 2 - Watching the Security Score Populate

I came back to the Summary page after findings had time to flow in. The Trends overview widget (six-month, day-over-day view) now showed 2 threats, 35 resources, and 103 total findings - up from the 3 findings seen minutes earlier. The Resources panel was already surfacing the noisiest resource in the account: the account root itself, tied to 36 findings from an EC2 snapshot public-access check, alongside a handful of IAM policy resources related to VPC Flow Logs CloudWatch delivery. A couple of those policy names still referenced lab5-trail, a reminder that this is a shared account and Lab 5's leftover resources were still contributing to the noise floor here.

**[ SS3 - Trends Overview After Findings Populate ]**  

<img width="1650" height="785" alt="Screenshot 2026-08-24 192505" src="https://github.com/user-attachments/assets/3920fe58-5947-4397-9f5a-93f344b8fa0b" />


*Security Hub Trends overview (6 months, day-over-day): Threats 2, Exposure 0, Resources 35, All findings 103; Resources panel led by the account root (36 findings) and VPC Flow Logs / CloudTrail IAM policy resources*

### Phase 3 - Reviewing Standards: FSBP and CIS

This is where the lab script and the actual console diverged the most. I opened the AWS Foundational Security Best Practices v1.0.0 standard expecting a red-and-green failed-controls list. Instead, all 331 controls showed as No Data, with a banner stating the security score was still being generated and would be available in 24 minutes. The CIS AWS Foundations Benchmark v1.2.0 standard, checked right after, showed the same thing: all 42 controls at No Data, with its own message estimating 25 minutes until the score was ready.

**[ SS4 - FSBP Standard, Score Still Generating ]**  

<img width="1917" height="903" alt="Screenshot 2026-08-24 193342" src="https://github.com/user-attachments/assets/a531317d-d3f1-4fcf-a18b-e9b80d0e1431" />


*AWS Foundational Security Best Practices v1.0.0: 0 Passed, 0 Failed, 331 No data; banner reads security score still being generated, available in 24 minutes*

**[ SS5 - CIS Benchmark, Score Still Generating ]**  


<img width="1917" height="907" alt="Screenshot 2026-08-24 194429" src="https://github.com/user-attachments/assets/bf4306b7-0268-4815-b7db-dcbafdd61cfd" />


*CIS AWS Foundations Benchmark v1.2.0: 0 Passed, 0 Failed, 42 No data; banner reads security score still being generated, available in 25 minutes*

> *The lab script assumed controls would already be evaluated and mostly failing on first look, so the plan called for a walkthrough of a red failed-controls list. What actually happened is that Security Hub needs a first evaluation cycle (20 to 25 minutes in this run) before any control shows Passed or Failed. I have kept the No Data screenshots because they are a more honest picture of a fresh Security Hub enablement than a fabricated failed-controls list would be.*

While the account-wide score was still generating, I could still open individual controls and see their own check history, since some checks run independently of the account rollup. Hardware MFA for the root user (IAM.6) is a good example: the control itself showed No Data at the account level, but its one recorded check already showed PASSED and RESOLVED, evaluated against the account root about 41 minutes earlier.

**[ SS6 - Individual Control Detail: Hardware MFA for Root (IAM.6) ]**  

<img width="1675" height="727" alt="Screenshot 2026-08-24 193926" src="https://github.com/user-attachments/assets/1e58364f-b828-4c98-8aec-6a4953dbc7e3" />


*IAM.6 - Hardware MFA should be enabled for the root user: Severity Critical, Control status No Data at account level, Checks (1): PASSED, RESOLVED, region eu-north-1, resource Account 712934828848*

Once the score had fully generated, I went back to the Summary page and got the number the lab was actually after. The account sat at 52% overall (27 controls passed, 25 failed), broken down as 47% on CIS AWS Foundations Benchmark (8 passed, 9 failed) and 60% on AWS Foundational Security Best Practices (26 passed, 17 failed). Two additional standards, AI Security Best Practices and AWS Resource Tagging Standard, were listed but not enabled, so they were not part of this score. The Assets with the most findings panel again put the account root at the top with 16 findings, followed by the two VPCs and a couple of S3 buckets tied to the CloudTrail logging setup.

**[ SS7 - Final Security Score by Standard ]**  

<img width="1917" height="907" alt="Screenshot 2026-08-24 200049" src="https://github.com/user-attachments/assets/6bb7812c-35ba-43b3-baae-1da9cfb9234e" />


*Security Hub CSPM Summary: overall security score 52% (27 passed, 25 failed); CIS AWS Foundations Benchmark 47% (8/9), AWS Foundational Security Best Practices 60% (26/17); AI Security Best Practices and AWS Resource Tagging Standard listed but not enabled*

### Phase 4 - Reviewing GuardDuty Findings Directly

The lab script called for filtering Security Hub's own Findings page down to Product name = GuardDuty. In practice, the fastest and clearest view of what GuardDuty had actually detected was its own Summary page, so that is what I captured instead. It reported 412 total findings across 24 resources in the account, split by severity as 11 Critical, 162 High, 170 Medium, and 69 Low, plus 11 attack sequence findings - GuardDuty's correlated, multi-step detections rather than single isolated alerts.

The top findings list read like a live incident feed: a potential Kubernetes cluster compromise on eks-demo-cluster, a potential credential compromise tied to an IAM user, a potential S3 data compromise linked to the same user, and several EC2 instance group compromise findings, all flagged Critical. The Most common finding types chart showed one dominant slice, AttackSequence:EC2/CompromisedInstanceGroup, meaning most of the volume was GuardDuty correlating the same underlying compromise pattern across multiple resources rather than 400-odd unrelated one-off alerts.

**[ SS8 - GuardDuty Findings Summary ]**  

<img width="1917" height="906" alt="Screenshot 2026-08-24 200720" src="https://github.com/user-attachments/assets/ead2d825-2bcb-4f09-9031-26eb2bccfa08" />


*GuardDuty Summary: 412 total findings, 24 resources with findings, 11 attack sequences; severity breakdown Critical 11 / High 162 / Medium 170 / Low 69; top findings include Kubernetes cluster compromise and IAM credential compromise, both Critical*

> *These are GuardDuty's own sample and correlated findings rather than a hand-built incident like Lab 5's, since this lab is about the dashboarding layer, not about planting a fresh compromise. The numbers above are reported exactly as the console showed them.*

### Phase 5 - Building Custom Insights

Security Hub Insights let you save a filter-and-group combination as a reusable view, which is exactly what a SOC dashboard is built out of. I created three, each layered on the default active-and-unreviewed filters (Workflow status is NEW or NOTIFIED, Record state is ACTIVE).

Insight 1 grouped by resource type with an added Severity label = CRITICAL filter, to answer the question a triage analyst asks first: what is critical and unresolved right now. It returned a single AwsAccount resource, which lined up with the account-root finding concentration already seen in the Resources panel.

**[ SS9 - Custom Insight 1: Critical Findings by Resource Type ]**  

<img width="1917" height="907" alt="Screenshot 2026-08-24 202314" src="https://github.com/user-attachments/assets/f968c53f-a0a3-4098-b266-a554ab3b91ef" />


*Insight: Critical findings by resource type - filters Severity label is CRITICAL, Workflow status NEW/NOTIFIED, Record state ACTIVE; result: AwsAccount, count 1*

Insight 2 added a Product name = GuardDuty filter and grouped by severity label, to isolate GuardDuty's contribution from everything else feeding Security Hub. With Severity label = LOW also applied, it surfaced one finding tied to a temporary IAM access key resource, a much smaller and lower-stakes result than the raw GuardDuty console numbers in Phase 4, which is the point of narrowing a broad findings feed down to a specific saved view.

**[ SS10 - Custom Insight 2: GuardDuty Findings by Severity ]**  

<img width="1917" height="907" alt="Screenshot 2026-08-24 220425" src="https://github.com/user-attachments/assets/c204f557-2653-4ef4-a2c5-b4103569845f" />


*Insight: GuardDuty findings by severity - filters Severity label is LOW, Product name is GuardDuty, Workflow status NEW/NOTIFIED, Record state ACTIVE; result: LOW, count 1, resource type AwsIamAccessKey*

Insight 3 dropped the severity and product filters and grouped by product name instead, to see the overall split of new, unreviewed findings across every connected source. Security Hub's own posture checks accounted for 59 unreviewed findings against just 1 from GuardDuty at that moment, and the resource-type breakdown in the side panel showed the load spread across the account, both VPCs, an S3 bucket, a subnet, and a security group, roughly in that order.

**[ SS11 - Custom Insight 3: New Unreviewed Findings by Product ]**  

<img width="1917" height="900" alt="Screenshot 2026-08-24 220608" src="https://github.com/user-attachments/assets/72a74bc0-9fa1-4543-a6e0-69ae384a4359" />


*Insight: New unreviewed findings by product - filters Workflow status NEW/NOTIFIED, Record state ACTIVE; result: Security Hub 59, GuardDuty 1; resource type breakdown led by AwsAccount and the two VPCs*

---

## 6. Findings

This lab is a posture snapshot rather than an incident, so the findings here are about the state of the account's security configuration, not a set of planted IOCs.

- Overall Security Hub CSPM score: 52% (27 controls passed, 25 failed) across two enabled standards.
- AWS Foundational Security Best Practices v1.0.0: 60% (26 passed, 17 failed).
- CIS AWS Foundations Benchmark v1.2.0: 47% (8 passed, 9 failed) - the weaker of the two standards on this account.
- GuardDuty carried the heaviest raw finding volume: 412 findings, 11 of them Critical, concentrated into 11 correlated attack-sequence detections rather than spread evenly.
- The account root (712934828848) was consistently the top resource by finding count across the Trends, Assets, and Insight views, largely tied to an EC2 snapshot public-access check.
- IAM.6 (Hardware MFA for the root user) already showed a PASSED, RESOLVED check on first inspection, evaluated in eu-north-1.
- Two standards available to enable, AI Security Best Practices and AWS Resource Tagging Standard, were left off for this lab and are not reflected in the 52% score.

---

## 7. How Security Hub Aggregates Findings

Security Hub CSPM sits a layer above the individual detection services rather than replacing any of them. GuardDuty keeps generating its own threat findings on its own timeline; AWS Config keeps recording resource configuration and evaluating it against managed rules; IAM Access Analyzer keeps watching for externally shared resources. Security Hub's job is to pull all three feeds into one findings table, map the relevant subset against the FSBP and CIS control catalogs, and roll the pass/fail count up into a single percentage score.

- Ingestion happens automatically once a service is connected - there was no manual accept-findings step required in the current console, unlike the integration flow the original lab script described.
- Each control (like IAM.6) maps to one or more underlying checks, which can carry their own status and history independently of whether the account-wide score has finished generating yet.
- Findings and controls carry a workflow status (NEW, NOTIFIED, RESOLVED, and so on) and a record state (ACTIVE, ARCHIVED), which is what the Insight filters in Phase 5 were built around.
- In a live SOC, CloudTrail would be the next stop after Security Hub - filtering by account or resource ID to see exactly which API calls produced a given finding or changed a given control's compliance state.

---

## 8. Deviations from the Planned Lab Script

In the interest of an accurate report rather than a tidy one, here is what changed between the original lab plan and what I actually did and captured:

- Integrations step: the newer Security Hub CSPM console links GuardDuty, Config, and Access Analyzer automatically on enablement. There was no separate manual accept-findings screen to screenshot.
- Remediation step: the original plan called for manually fixing four specific findings (S3 public access, CloudTrail log validation, CloudTrail multi-region, Access Analyzer) and re-running checks to show a before-and-after score. I did not capture that remediation cycle - what I have instead is the score changing naturally as evaluation completed (Phase 3), plus one control (IAM.6) already passing on first check without any manual fix.
- Security Hub-side GuardDuty filter: rather than filtering Security Hub's own Findings page by Product name = GuardDuty, I used GuardDuty's own Summary page, which gave a clearer picture of severity and finding-type distribution for this report.
- Workflow status update: I did not manually change a finding's workflow status (for example, marking one Notified or Resolved) and screenshot the change, so that step is not part of this walkthrough.
- Final recap dashboard: this report closes on the Phase 3 score screenshot (52% overall) as the last full-dashboard view captured, rather than a separate closing summary screenshot.

> *None of these adjustments change the core lab objective. The dashboard was built, the score was reviewed, controls were inspected individually, GuardDuty's output was cross-checked, and three working custom Insights exist in the account. The gaps above are places where the console's current behavior did not match the original written steps, and I have chosen to report what happened over what was planned.*

---

## 9. Key Learnings

- A freshly enabled Security Hub is not immediately useful. The 20 to 25 minute gap before the first score generates is a real operational detail, not a lab artifact - a new account onboarded into Security Hub will show No Data before it shows Passed or Failed, and that gap should be expected during any real rollout.
- Individual control checks can resolve independently of the account-wide score. IAM.6 already had a PASSED result while the account rollup still said No Data, which means a control detail page is sometimes more current than the summary dashboard.
- GuardDuty's attack-sequence correlation matters more than raw finding counts. 412 individual findings sound alarming, but 11 correlated attack sequences is the number that actually tells you how many distinct incidents are being represented.
- The account root being the top resource by finding count across three different views (Trends, Assets, Insights) is a pattern worth remembering: account-level checks (like snapshot public-access settings) tend to dominate finding counts even when no single EC2 instance or S3 bucket is individually compromised.
- Custom Insights are cheap to build and genuinely useful. Three filter-and-group combinations took a few minutes each and immediately turned a noisy 412-finding GuardDuty feed and a 100-plus finding Security Hub feed into three specific, answerable questions.
- AWS product interfaces move faster than static lab guides. The gap between the written Security Hub Integrations flow and the CSPM console I actually used is a reminder to verify current console behavior before assuming a script written even a few months earlier is still accurate.

---

## 10. Conclusion

This lab built the dashboarding layer that Lab 5's incident response work would eventually be triaged through. Where Lab 5 was about chasing one host end to end, this one was about standing up the tool that decides which host gets chased first: CloudTrail, GuardDuty, Config, and Access Analyzer running underneath, Security Hub CSPM rolling all of it into a single 52% compliance score and a findings feed, and three custom Insights turning that feed into specific, repeatable triage questions.

The most useful takeaway is not the 52% score itself, it is watching how that score got there: a 20-plus minute evaluation delay, a control that resolved before the account rollup did, and a findings feed dominated by one correlated attack sequence rather than hundreds of unrelated alerts. Those are the details a real SOC analyst learns to expect from Security Hub, and they only show up if you watch the console honestly instead of assuming it behaves exactly like the write-up says it should.

> *Key takeaway: a posture dashboard is only as useful as the questions you build into it. Enabling Security Hub gets you a score; building Insights on top of it is what turns that score into something an analyst can act on every morning.*

*End of Lab Report - AWS Security Hub Unified Dashboard: Enablement, Posture Review & Custom Insights*
