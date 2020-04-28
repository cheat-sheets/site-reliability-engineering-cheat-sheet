# Site Reliability Engineering Cheat Sheet

**Site Reliability Engineering (SRE)** is a discipline that incorporates aspects of software engineering and applies 
them to infrastructure and operations problems. The main goals are to create **scalable** and **highly reliable** 
software systems. According to Ben Treynor, founder of Google's Site Reliability Team, 
SRE is "what happens when a software engineer is tasked with what used to be called operations."

SRE is closely related to DevOps.

**DevOps** is a set of practices that combines software development (Dev) and information-technology operations (Ops) 
which aims to shorten the systems development life cycle and provide continuous delivery with high software quality.

Goals of DevOps:

- Improved deployment frequency;
- Faster time to market;
- Lower failure rate of new releases;
- Shortened lead time between fixes;
- Faster mean time to recovery;

DevOps is more holistically defined and has many goals including quality, reliability, faster time to market, etc., 
while SRE focuses primarily on reliability and everything else is implied. SRE is deeper in this sense and DevOps
is broader.

## Foundations

### 1. Embracing Risk (SLOs, Error Budgets, Monitoring)

- Service failures can have many potential effects, including user dissatisfaction, harm, or loss of trust; 
direct or indirect revenue loss; brand or reputational impact; and undesirable press coverage.
- Unreliable systems can quickly erode users' confidence, so we want to reduce the chance of system failure.
- Cost does not increase linearly as reliability increments 
  (an incremental improvement in reliability may cost 100x more than the previous increment).
- It's **important to identify the appropriate level of reliability** by performing cost/benefit analysis to determine
  where on the **(nonlinear) risk continuum** a product should be placed.
- User experience is dominated by less reliable systems. Ex. mobile network is 99% reliable.

A few issues to consider when determining target level of availability:
    - What level of service will the users expect?
    - Does this service tie directly to revenue (either our revenue, or our customers' revenue)?
    - Is this a paid service, or is it free?
    - If there are competitors in the marketplace, what level of service do those competitors provide?
    - Is this service targeted at consumers, or at enterprises?

#### SLIs, SLOs, SLAs 

- **SLI (Service Level Indicator)** - a carefully defined quantitative measure of some aspect of the level of 
service that is provided. Examples: request latency, error rate, system throughput.
- **SLO (Service Level Objective)** a target value or range of values for a service level that is measured by an SLI. 
A natural structure for SLOs is thus **SLI ≤ target**, or **lower bound ≤ SLI ≤ upper bound**. 
SLOs are the tool by which you measure your service's reliability.
- **SLA (Service Level Agreement)** - an explicit or implicit contract with your users that includes consequences 
of meeting (or missing) the SLOs they contain. The consequences are most easily recognized when they are 
financial — a rebate or a penalty — but they can take other forms.

It's recommended to choose SLIs that can be represented as the ratio of two numbers: 
**the number of good events divided by the total number of events**. 
SLIs of this form have a couple of particularly useful properties:
- The SLI ranges from 0% to 100%, where 0% means nothing works, and 100% means nothing is broken.
- This style lends itself easily to the concept of an error budget: the SLO is a target percentage and the error 
budget is 100% minus the SLO.
- Making all of your SLIs follow a consistent style allows you to take better advantage of tooling: you can write 
alerting logic, SLO analysis tools, error budget calculation, and reports to expect the same inputs: 
numerator, denominator, and threshold. Simplification is a bonus here.

There are two types of SLOs:
- **Request-based SLOs** - based on an SLI that is defined as the ratio of the number of good requests to 
the total number of requests. 
    - **"Latency is below 100 ms for at least 95% of requests."** A good request is one with a response time less 
    than 100 ms, so the measure of compliance is the fraction of requests with response times under 100 ms. 
- **Windows-based SLOs** - based on an SLI defined as the ratio of the number of measurement intervals that meets 
some goodness criterion to the total number of intervals.
    - **"The 95th percentile latency metric is less than 100 ms for at least 99% of 10-minute windows"**. 
    A good measurement period is a 10-minute span in which 95% of the requests have latency under 100 ms.
    
There are two types of compliance periods for SLOs:
- **Calendar-based** periods (from date to date).
- **Rolling periods** (from n days ago to now, where n ranges from 1 to 30 days).
 
### [Example SLO document](https://landing.google.com/sre/workbook/chapters/slo-document/)

**Example SLIs and SLOs for a service API**:

<table style="table-layout:fixed">
  <colgroup>
    <col style="width:50%"/>
    <col style="width:50%"/>
    <col style="width:50%"/>
  </colgroup>
  <tr>
    <td><b>Category</b></td>
    <td><b>SLI</b></td>
    <td><b>SLO</b></td>
  </tr>
  <tr>
      <td><b>Availability</b></td>
      <td>
        <p>The proportion of successful requests, as measured from the load balancer metrics:</p> 
        <p>count of http_requests which do not have a 5XX status code divided by count of all http_requests.</p>
      </td>
      <td>97% success</td>
  </tr>
  <tr>
        <td><b>Latency</b></td>
        <td>
          <p>The proportion of sufficiently fast requests, as measured from the load balancer metrics:</p> 
          <p>count of http_requests with a duration less than or equal to "X" milliseconds divided by count of all
           http_requests.</p>
        </td>
        <td>
          <p>90% of requests < 400 ms</p>
          <p>99% of requests < 850 ms</p>
        </td>
    </tr>
</table>

**Example SLIs and SLOs for a data pipeline**:

<table style="table-layout:fixed">
  <colgroup>
    <col style="width:50%"/>
    <col style="width:50%"/>
    <col style="width:50%"/>
  </colgroup>
  <tr>
    <td><b>Category</b></td>
    <td><b>SLI</b></td>
    <td><b>SLO</b></td>
  </tr>
  <tr>
      <td><b>Freshness</b></td>
      <td>
        <p>The proportion of records read from the league table that were updated recently:</p> 
        <p>count of all data_requests for with freshness less than or equal to X minutes divided by count of all data_requests</p>
      </td>
      <td>
        <p>90% of reads use data written within the previous 1 minute.</p>
        <p>99% of reads use data written within the previous 10 minutes.</p>
      </td>
  </tr>
  <tr>
      <td><b>Correctness</b></td>
      <td>
        <p>The proportion of records injected into the state table by a correctness prober that result in the correct data being read from the league table.
        A correctness prober injects synthetic data, with known correct outcomes, and exports a success metric:
        </p> 
        <p>count of all data_requests which were correct divided by count of all data_requests.</p>
      </td>
      <td>
        <p>99.99999% of records injected by the prober result in the correct output.</p>
      </td>
  </tr>
</table>

#### Error Budgets

**Error budgets** are a tool for balancing reliability with other engineering work, and a great way to decide 
which projects will have the most impact. 
Changes are a major source of instability, representing roughly 70% of our outages, and development work for features 
competes with development work for stability. 
The error budget forms a control mechanism for diverting attention to stability as needed.

An error budget is 1 minus the SLO of the service. A 99.9% SLO service has a 0.1% error budget.
If our service receives 1,000,000 requests in four weeks, a 99.9% availability SLO gives us 
a budget of 1,000 errors over that period.

### [Example Error Budget Policy](https://landing.google.com/sre/workbook/chapters/error-budget-policy/):

**Goals**:
- Protect customers from repeated SLO misses
- Provide an incentive to balance reliability with other features

**SLO Miss Policy**:
- If the service is performing at or above its SLO, then releases (including data changes) will proceed according to the release policy.
- If the service has exceeded its error budget for the preceding four-week window, we will halt all changes and releases other than P01 issues or security fixes until the service is back within its SLO.
- Depending upon the cause of the SLO miss, the team may devote additional resources to working on reliability instead of feature work.
- The team must work on reliability if:
    - A code bug or procedural error caused the service itself to exceed the error budget.
    - A postmortem reveals an opportunity to soften a hard dependency.
    - Miscategorized errors fail to consume budget that would have caused the service to miss its SLO.
- The team may continue to work on non-reliability features if:
    - The outage was caused by a company-wide networking problem.
    - The outage was caused by a service maintained by another team, who have themselves frozen releases to address their reliability issues.
    - The error budget was consumed by users out of scope for the SLO (e.g., load tests or penetration testers).
    - Miscategorized errors consume budget even though no users were impacted.
    
**Outage Policy**:
- If a single incident consumes more than 20% of error budget over four weeks, then the team must conduct a postmortem. 
  The postmortem must contain at least one P0 action item to address the root cause.
- If a single class of outage consumes more than 20% of error budget over a quarter, the team must have a P0 item on 
  their quarterly planning document2 to address the issues in the following quarter.

**Escalation Policy**:
- In the event of a disagreement between parties regarding the calculation of the error budget or the specific 
  actions it defines, the issue should be escalated to the CTO to make a decision.
  
- Don't over-rely on 3rd party services. Chubby had too good of SLIs and users over-relied on it. 
  They had to make planned service outages to prevent it.

#### Alerting on SLOs**:

https://landing.google.com/sre/workbook/chapters/alerting-on-slos/

Your goal is to be notified for a significant event: **an event that consumes a large fraction of the error budget**.

Consider the following attributes when evaluating an alerting strategy:
- **Precision** - the proportion of events detected that were significant. The fewer false positives the higher the precision.
- **Recall** - the proportion of significant events detected. The fewer false negatives the higher the recall.
- **Detection time** - how long it takes to send notifications in various conditions. 
  Long detection times can negatively impact the error budget.
- **Reset time** - how long alerts fire after an issue is resolved. 
  Long reset times can lead to confusion or to issues being ignored.
  
**Burn rate** is how fast, relative to the SLO, the service consumes the error budget. 
- Burn rate of 1 means that it's consuming error budget at a rate that leaves you with exactly 0 budget at the end 
  of the SLO's time window.
- With 100% downtime the burn rate is `1 / ((100% - SLO) * 100)`.

| Burn rate | Error rate for a 99.9% SLO | Time to exhaustion |
| --- | --- | ---  |
| 1 | 0.1% | 30 days |
| 2 | 0.2% | 15 days |
| 10 | 1% | 3 days |
| 1000 | 100% | 43 minutes |
  
Ways to alert on significant events:

- 1: Target Error Rate ≥ SLO Threshold. For example, if the SLO is 99.9% over 30 days, alert if the error rate over the previous 10 minutes is ≥ 0.1%:
    - `expr: job:slo_errors_per_request:ratio_rate10m{job="myjob"} >= 0.001`.
    - Cons: Precision is low: The alert fires on many events that do not threaten the SLO. 
      A 0.1% error rate for 10 minutes would alert, while consuming only 0.02% of the monthly error budget.
- 2: Increased Alert Window. To keep the rate of alerts manageable, you decide to be notified only if an event 
consumes 5% of the 30-day error budget — a 36-hour window.
    - `expr: job:slo_errors_per_request:ratio_rate36h{job="myjob"} > 0.001`.
    - Cons:
        - Very poor reset time: In the case of 100% outage, an alert will fire shortly after 2 minutes, 
          and continue to fire for the next 36 hours.
        - Calculating rates over longer windows can be expensive in terms of memory or I/O operations, 
          due to the large number of data points.
- 3: Incrementing Alert Duration. Most monitoring systems allow you to add a duration parameter to the alert criteria 
so the alert won’t fire unless the value remains above the threshold for some time.
    - `expr: job:slo_errors_per_request:ratio_rate1m{job="myjob"} > 0.001; for: 1h'`.
    - Cons:
        - Poor recall and poor detection time: Because the duration does not scale with the severity of the incident, 
          a 100% outage alerts after one hour, the same detection time as a 0.2% outage.
        - If the metric even momentarily returns to a level within SLO, the duration timer resets. 
          An SLI that fluctuates between missing SLO and passing SLO may never alert.
- 4: Alert on Burn Rate. 
    - `job:slo_errors_per_request:ratio_rate1h{job="myjob"} > (36*0.001)` 
    - Cons:
        - Low recall: A 35x burn rate never alerts, but consumes all of the 30-day error budget in 20.5 hours.
        - Reset time: 58 minutes is still too long.
- 5: Multiple Burn Rate Alerts.
    - ```expr: (
                 job:slo_errors_per_request:ratio_rate1h{job="myjob"} > (14.4*0.001)
               or
                 job:slo_errors_per_request:ratio_rate6h{job="myjob"} > (6*0.001)
               )
         severity: page
         
         expr: job:slo_errors_per_request:ratio_rate3d{job="myjob"} > 0.001
         severity: ticket```
    - Cons:
        - More numbers, window sizes, and thresholds to manage and reason about.
        - An even longer reset time, as a result of the three-day window.
        - To avoid multiple alerts from firing if all conditions are true, you need to implement alert suppression.
- 6: Multiwindow, Multi-Burn-Rate Alerts.
    - ``` expr: (
               job:slo_errors_per_request:ratio_rate1h{job="myjob"} > (14.4*0.001)
             and
               job:slo_errors_per_request:ratio_rate5m{job="myjob"} > (14.4*0.001)
             )
           or
             (
               job:slo_errors_per_request:ratio_rate6h{job="myjob"} > (6*0.001)
             and
               job:slo_errors_per_request:ratio_rate30m{job="myjob"} > (6*0.001)
             )
       severity: page
       
       expr: (
               job:slo_errors_per_request:ratio_rate24h{job="myjob"} > (3*0.001)
             and
               job:slo_errors_per_request:ratio_rate2h{job="myjob"} > (3*0.001)
             )
           or
             (
               job:slo_errors_per_request:ratio_rate3d{job="myjob"} > 0.001
             and
               job:slo_errors_per_request:ratio_rate6h{job="myjob"} > 0.001
             )
       severity: ticket``` 
    - Cons:
        - Lots of parameters to specify, which can make alerting rules hard to manage.   

**Fast-burn / slow-burn** strategy described here https://cloud.google.com/monitoring/service-monitoring/alerting-on-budget-burn-rate#burn-rates
corresponds to strategy 5 above.
  

**Notes**:
- GCP's Monitoring service allows alerting on burn rate: 
https://cloud.google.com/monitoring/service-monitoring/alerting-on-budget-burn-rate,
https://cloud.google.com/monitoring/service-monitoring#slo-types.
- Extreme availability SLOs. With target monthly availability of 99.999% a 100% outage would exhaust its budget
in 26 seconds - not enough to react. The only way to defend this level of reliability is to design the system so that
the chance of a 100% outage is extremely low. For example if you roll out a change to 1% of the machines you will have 
43 minutes before you exhaust your error budget. 
- [Availability Table](https://landing.google.com/sre/sre-book/chapters/availability-table/#appendix_table-of-nines).


#### Monitoring

The 4 golden signals: 
- **Latency**. The time it takes to service a request. 
- **Traffic**. A measure of how much demand is being placed on your system, 
measured in a high-level system-specific metric. E.g. HTTP requests per second, network I/O rate, concurrent sessions.
- **Errors**. The rate of requests that fail, either explicitly (e.g., HTTP 500s), 
implicitly (for example, an HTTP 200 success response, but coupled with the wrong content), 
or by policy (for example, "If you committed to one-second response times, any request over one second is an error").
- **Saturation**. How "full" your service is. A measure of your system fraction, emphasizing the resources that are 
most constrained (e.g., in a memory-constrained system, show memory; in an I/O-constrained system, show I/O). 
Note that many systems degrade in performance before they achieve 100% utilization, so having a utilization target 
is essential.

**Monitoring symptoms vs causes**:
- Symptoms are metrics that are visible to users of the service or application. For example low latency, many errors. 
- Causes are metrics that are not visible to users. For example high CPU usage on the web server, 
high rate of refused connections in the database.  

**Best practices**:
- **Monitor the versions of the components**. Monitor the command-line flags, especially when you use these flags to 
enable and disable **features** of the service. If configuration data is pushed to your service dynamically, 
**monitor the version of this dynamic configuration**. When you're trying to correlate an outage with a rollout, 
it's much easier to look at a graph/dashboard linked from your alert than to trawl through your CI/CD 
system logs after the fact.
- Even if your service didn't change, any of its dependencies might change or have problems, 
so you should also monitor responses coming from direct dependencies.
- Monitor saturation: 
    - resources with hard limits: RAM, disk, or CPU quota; 
    - resources without hard limits: open file descriptors, active threads in any thread pools, waiting times in queues, 
      or the volume of written logs.
    - programming language specific metrics: the heap and metaspace size for Java, the number of goroutines for Go.  
- Test alerting logic: https://landing.google.com/sre/workbook/chapters/monitoring/#testing-alerting-logic.


### 2. Eliminating Toil


### 3. Simplicity


## Practices

1. On-call, incident response, post-mortem culture.
2. Managing Load.
3. Configuration Management.
4. Reliability Testing.

## Processes

1. Organizational Change
2. 



**Notes**:

- Reliability deals with behaviour of failure rate over a long period of operation, while quality control
deals with percent of defectives based on performance specifications at a certain point in time. 
- Hacker puts hosting service Code Spaces out of business: 
https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/.
- Stalking the Wily Hacker: http://pdf.textfiles.com/academics/wilyhacker.pdf.  
- The DevOps handbook https://www.oreilly.com/library/view/the-devops-handbook/9781457191381/.
