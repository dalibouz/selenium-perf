# Load Testing with 1,000 Real Browsers: How I Built a Selenium Grid Load Generator on AKS

Most load tests tell you how fast your servers answer. Your users don't experience server response times. They experience pages that load, render, run JavaScript and finally become usable.

For a former client, I needed a performance-testing setup that measured the second thing, at scale. So I built a load generator that drives up to 1,000 concurrent real browsers, orchestrated by a Node.js application running on Azure Kubernetes Service (AKS). Here is how it works, what it measured, and where I would take it next.

## Why real browsers?

Protocol-level tools such as JMeter or k6 are excellent at hammering an API with HTTP requests. But they don't render pages or execute front-end code, so they miss a whole category of problems: heavy JavaScript bundles, slow third-party scripts, layout shifts, requests chained by the client.

Driving real browsers with Selenium closes that gap. Each virtual user logs in, clicks and navigates like a person would. The trade-off is cost: a browser needs far more CPU and memory than an HTTP client, so you need many machines, and something to manage them. That "something" is what I built.

## The architecture

The solution has three building blocks, all deployed on AKS:

- **Selenium hubs**: up to 10 of them, each one routing sessions to its own pool of browsers.
- **Selenium nodes**: from 10 to 100 per hub, each running a browser that plays one virtual user.
- **A Node.js orchestrator**: an application I developed, packaged as a Docker image and deployed next to the hubs and nodes. It deploys the grid, distributes the virtual users, runs the user scenarios and collects every metric.

With 10 hubs × 100 nodes, the platform scales from 0 to 1,000 concurrent users, all connecting to the application under test and simulating real user actions.

Spreading the browsers across several hubs, rather than one giant grid, keeps each hub at a manageable size and lets the load grow hub by hub.

## One YAML file per test

The orchestrator runs as a job on AKS, and the job's YAML file is the only thing you edit to define a test:

- **test type**: baseline, smoke, peak / scalability, or stress;
- **number of virtual users**;
- **test duration**;
- and the other run parameters.

Each test type answers a different question:

- **Smoke**: does the scenario work at all, under minimal load?
- **Baseline**: what does "normal" look like? This is the reference for every other run.
- **Peak / scalability**: does the application hold the expected peak, and how does it behave as the load grows?
- **Stress**: where does it break, and how does it recover?

Because the whole test is described in a version-controlled YAML file, a run is repeatable: same scenario, same load profile, comparable results.

## Watching the test live

During the run, the orchestrator streams metrics to **Grafana** in real time:

- the **response time of every user action**;
- **CPU and memory usage of every hub and node**;
- the **orchestrator's own metrics**.

Monitoring the load generator itself matters more than it sounds. If a node runs out of CPU, its browser slows down and the measured response times go up, even though the application is fine. Watching the grid's resources next to the application's response times is how you tell a slow application from a saturated test rig.

## Measuring what users actually feel

Response times are only part of the picture. Through the **Chrome DevTools Protocol (CDP)**, the orchestrator also collected the **Core Web Vitals**, the metrics Google uses to describe loading speed, interactivity and visual stability, as seen from inside the browser.

During the tests, we also took **HAR snapshots**. A HAR file records every network request the browser made, with its timings. Comparing snapshots taken at low and high load shows which requests slow down, fail or pile up when the application is under pressure.

## The results report

At the end of each run, the orchestrator generates a results file with, for every transaction:

- **min**, **max** and **average** response time;
- the **90th percentile (p90)**.

The p90 is the number I look at first. An average can look healthy while one user in ten waits far too long. The p90 tells you what that tenth user experiences.

## Correlating with the application's monitoring

On the application side, monitoring was split: roughly half in **Splunk**, half in **Grafana**, plus several internal tools. That is common in large organisations, and it means the story of a test is spread across several places: browser-side metrics in one dashboard, server-side logs and metrics in others.

## Running it on a budget

Not every client can afford a large Kubernetes cluster for load testing. The same solution can also run on a **Raspberry Pi cluster**, which makes browser-based load testing accessible when the budget is limited.

## What's next: let AI read the results

Today, a test produces Grafana dashboards, Splunk data, internal monitoring, a results file, HAR snapshots and Core Web Vitals. A human has to connect the dots.

The next feature I have in mind is to **consolidate all of it in one place and analyse it with AI**: correlate a p90 spike with a resource limit on the grid, a slow request in a HAR file and an error in the server logs, then explain what happened in plain language.

## Key takeaways

- **Test what users experience.** Real browsers catch front-end and network issues that protocol-level tests miss.
- **Orchestration is the hard part.** Scaling to 1,000 browsers needs automation: hubs, nodes, scenarios and metrics managed by code.
- **Monitor your load generator.** Otherwise you can't trust your response times.
- **Make tests repeatable.** One YAML file per test turns load testing into a routine.
- **Look at percentiles.** The p90 tells the truth that the average hides.
