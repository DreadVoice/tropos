# Tropos

Cloud infrastructure context for Jira issues. An Atlassian Forge app with a Java backend built for [Codegeist 2026](https://codegeist2026.devpost.com/). 

## What this is

Tropos reads an Azure subscription and puts what it finds onto the Jira issues where people are already working.

Open a bug saying checkout is slow, and the issue shows you which Azure resources checkout actually runs on, what changed on them in the hours before the complaint, who owns them, and which earlier issues touched those same resources.

It only reads from Azure. It never starts, stops, resizes or deletes anything. The only writes it makes are to Jira: issues, comments and panels.

## The problem it goes after

When something breaks, the information you need is already written down somewhere. It is just scattered across places that do not talk to each other. The ticket is in Jira. The servers are in the Azure portal. The deploy that caused it is in an activity log. The person who knows is on leave. The last time this happened is in a ticket from March that nobody remembers.

So people spend the first twenty minutes of every incident reassembling context by hand, in browser tabs.

Tropos assembles it once, on a schedule, and keeps it attached to the work.

## How it works

Three pieces.

**The Forge app** is the part that lives inside Atlassian. It holds the manifest, the Jira issue panel, an admin configuration page, a Rovo agent and an MCP module. It contains no business logic of its own, it just calls the backend and renders what comes back.

**The backend** is Spring Boot on an OVHCloud instance, reached through Forge Remote. It talks to Azure, stores inventory snapshots in Postgres, does the correlation work, and calls back into the Jira API. It lives outside Atlassian for a plain reason: the Azure SDK for Java cannot run inside a Forge function, which is JavaScript only. The ingest and correlation work also runs longer than Forge's 25 second function limit allows.

**Azure** is read through a service principal holding only the built-in Reader role, scoped to one subscription.

Requests from Forge arrive carrying a Forge Invocation Token, which the backend verifies against Atlassian's JWKS before doing anything. Tenant state is keyed on the installation ID.

### The three things it does

1. **Attribution.** Pull the full resource inventory, then work out what each resource belongs to. Resource groups and tags map to services, services map to Jira projects, teams and people. This answers the question nobody can usually answer: who owns this, and what work is it attached to. Those resources also get pushed into Atlassian's Teamwork Graph, so infrastructure becomes part of the same connected picture as goals and issues.

2. **Context at the moment.** On an issue, show the resources behind the service, what changed on them recently, how their metrics moved, and which prior issues touched them.

3. **Findings that turn into work.** Risk and waste checks produce real Jira issues with the evidence attached and an owner assigned, rather than a dashboard nobody opens.

## Scope

### In for the hackathon

Azure only. Read only on the cloud side. Jira as the product surface.

### Deliberately left out

AWS and GCP. The original version of this project covered all three clouds, and it will again after December, but three shallow integrations are worth less here than one that goes deep. Also out: every write action against cloud infrastructure, which is both a security problem and a trust problem for anyone installing this to try it.

## Constraints worth keeping in mind

**Judges have no Azure subscription.** The app has to show something real within thirty seconds of being installed by someone who has never touched Azure. So there are two data sources behind the same interface: a shared read only demo subscription, and a path to connect your own. Demo data goes through the same code as live data, never a separate display path.

**The backend has to survive January.** Judging runs 5 January to 3 February 2027, six weeks after submission closes. The OVH box and its certificate have to still be working then, without anyone watching. That is an infrastructure obligation, not a coding one, and it gets planned for in week one rather than hoped about in February.

**Teamwork Graph is the biggest unknown.** It is new and the documentation may not cover what we need. It gets a hard two day timebox and a fallback.

## Development checklist

Starting 17 November 2026. Target submission 18 December, which leaves four days of slack before the deadline.

Risky and unfamiliar work goes first. Design gets its own week, because it carries the same weight in judging as the engineering does.

### Week 1, 17 to 22 November: prove the plumbing

No features this week. One thin slice through every layer, carrying fake data.

- [ ] Create the Azure account and convert it to pay as you go straight away, so it does not get disabled when the trial credit runs out
- [ ] Register the Entra application, assign Reader on the subscription, set a budget alert
- [ ] Make three real Azure calls (a Resource Graph query, a VM list, one Monitor metric) and save the raw responses as fixture files
- [ ] Sort out the Forge developer license and run `forge create`
- [ ] Spring Boot skeleton with Forge Invocation Token verification, following `atlassian/forge-remote-spring-boot` rather than hand rolling JWT validation
- [ ] Postgres wired up under Testcontainers
- [ ] Domain types, the `InventoryProvider` interface, and a fixture backed implementation of it
- [ ] Forge manifest with the `remotes` block set to `operations: [compute]`
- [ ] A resolver that calls `invokeRemote`, and one Jira issue panel that renders what comes back
- [ ] `forge tunnel` loop working comfortably
- [ ] Deploy to OVH: HTTPS with automatic certificate renewal, systemd unit set to restart always
- [ ] Point the manifest at OVH and install into a real dev Jira site

**Done when:** a panel on a real Jira issue shows fixture data, served from OVH, with the token verified. If that is not true by Sunday night, everything after this slips.

### Week 2, 23 to 29 November: real Azure data

- [ ] `AzureInventoryProvider` built against WireMock serving spec examples, then switched to live calls
- [ ] Resource Graph ingest into a Postgres snapshot schema
- [ ] Scheduled pull running on the Spring side, not as a Forge trigger
- [ ] Attribution engine: resource groups and tags to services, services to Jira projects and owners
- [ ] Admin page for configuring the mapping rules
- [ ] Blob storage inspection, developed against Azurite

**Done when:** real inventory from Azure is in Postgres, mapped to Jira projects, and showing in the panel.

### Week 3, 30 November to 6 December: the part that matters

Everything before this is plumbing. This week is the actual product.

- [ ] Blast radius view: from an issue, resolve the service to its resources
- [ ] Recent Activity Log changes and metric movement for those resources, within the relevant window
- [ ] Prior Jira issues that touched the same resources
- [ ] Push resources into Teamwork Graph (two day timebox, fall back to reading from the TWG API instead)
- [ ] Three or four finding rules only: untagged resource, orphaned disk, public blob container, idle VM
- [ ] Findings create Jira issues with evidence attached and an owner set

**Done when:** opening an incident issue shows what changed and what happened last time, and findings land in the backlog as real issues.

### Week 4, 7 to 13 December: agents, MCP and design

- [ ] `rovo:agent` with actions wired to the existing backend endpoints
- [ ] `rovo:mcp` module exposing the read only tools
- [ ] A `skill.md` so the same tools work with third party agents
- [ ] Empty states, loading states, error states
- [ ] Onboarding that offers demo data or connect your own, and makes the choice obvious
- [ ] Consistent typography and spacing across every surface
- [ ] First install path hardened and walked through end to end

**Done when:** every bonus prize hook is either in or consciously dropped, and the app reads as a product rather than a proof of concept.

### Week 5, 14 to 18 December: freeze and ship

- [ ] Monday: feature freeze
- [ ] Stand up the demo subscription: around twenty deallocated resources across a few resource groups, tagged like a plausible company
- [ ] Read only service principal for the shared demo
- [ ] Fresh install rehearsal on a clean Jira site, following the written instructions exactly as a judge would
- [ ] Write the setup and configuration instructions
- [ ] Script and record the demo video, under five minutes, public on YouTube
- [ ] Devpost submission: description, technology explanation, category (Apps for Software Teams), bonus prize write ups, installation link
- [ ] Submit to the Marketplace for listing, and keep the review ticket link
- [ ] Submit on Thursday 18th. Drafts stay editable, so there is no reason to sit on it

### Buffer, 19 to 22 December

Held back for whatever goes wrong. Hard stop at 12:00 PT on 22 December, which is 01:30 IST on the 23rd.

### Before walking away on the 18th

Judging runs through to 3 February and nobody will be watching the box.

- [ ] Uptime monitoring with alerts to phone
- [ ] Forced certificate renewal dry run
- [ ] Azure budget alert confirmed working
- [ ] Subscription confirmed as pay as you go, not trial
- [ ] Weekly calendar reminder to check the box, through early February

## If time runs short

Cut in this order: the Teamwork Graph push first (drop to reading from the API, or drop it), then the cost and waste side of findings, then the extra finding rules.

Keep the MCP module and the Rovo agent whatever happens. Once the backend endpoints exist, those are mostly module declarations over work already done, and they are worth more than they cost.

Do not cut the issue panel, the design week, or the video.

## Stack

- Atlassian Forge (manifest, UI Kit, Rovo agent, MCP module)
- Forge Remote to a self hosted backend
- Java 21, Spring Boot
- Postgres
- Azure SDK for Java: `azure-identity`, `azure-resourcemanager`, Resource Graph
- Testcontainers, WireMock, Azurite for testing without a live subscription
- OVHCloud instance, systemd, automatic TLS

## After December

The multi cloud version this started as. GCP and AWS behind the same `InventoryProvider` interface, which is most of the reason that interface exists.

## License

Apache 2.0. See [LICENSE](LICENSE).
