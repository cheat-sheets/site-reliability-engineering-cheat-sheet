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

**Monitoring**

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
    - resources without hard limits: open file descriptors, active threads in any thread pools, waiting times in queues, or the volume of written logs.
    - programming language specific metrics: the heap and metaspace size for Java, the number of goroutines for Go.  
- Test alerting logic: https://landing.google.com/sre/workbook/chapters/monitoring/#testing-alerting-logic.

**Alerting on SLOs**:


### 2. Eliminating Toil



### 3. Simplicity


**Notes**:

- Reliability deals with behaviour of failure rate over a long period of operation, while quality control
deals with percent of defectives based on performance specifications at a certain point in time. 
- Hacker puts hosting service Code Spaces out of business: 
https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/.
- Stalking the Wily Hacker: http://pdf.textfiles.com/academics/wilyhacker.pdf.  
- The DevOps handbook https://www.oreilly.com/library/view/the-devops-handbook/9781457191381/.
