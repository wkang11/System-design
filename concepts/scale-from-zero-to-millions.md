# Scale from zero to millions of users

Source: https://bytebytego.com/courses/system-design-interview/scale-from-zero-to-millions-of-users

Use this as a ladder, not a diagram to dump on the board. Start with one machine. Name the limit you just hit. Add one mechanism. Say what it unlocked and what new failure you accepted. Stop when the requirements are met.

## What to say before you draw

- Agree on the request path: name lookup, then HTTP, then a response (HTML for a browser, usually JSON for a mobile client).
- Separate "more capacity" from "still up when a box dies." Many steps do only one of those.
- Call out the single point of failure that remains after each step.

## Stages

### 1. One server

Everything shares a machine: app, database, cache. A DNS provider (not your box) turns the hostname into an address, and the client calls that address.

**Adds:** a system you can explain in one minute.

**Say:** "This ships a product. It dies with the machine, and it cannot grow past that machine's CPU, memory, and disk."

### 2. Split the app from the database

Put request handling on one tier and storage on another so they grow and fail separately.

Default to a relational database. Reach for a non-relational store when the data has no real relationships, you mostly store and fetch documents, latency has to be extremely low, or one SQL box cannot hold the volume.

**Adds:** independent scaling of traffic and data. Still one database.

**Say:** "The web tier can now grow without resizing the database. The database is the new single point of failure, and it is still doing every read and write."

### 3. Bigger machine vs more machines

A bigger machine is the right move while traffic is small: one box, no fleet to operate. It has a hard ceiling, and there is no spare. Once availability or capacity matters, add machines.

**Adds:** a decision rule, not a new component.

**Say:** "I scale up only while one machine is enough. I do not treat a larger box as redundancy."

### 4. Load balancer in front of the web tier

Clients use the balancer's public address. App servers stay on private addresses and are not reachable from the internet. If one app server dies, the others take its traffic. If traffic grows, add a server to the pool.

**Adds:** web-tier failover and a way to add app capacity without giving clients a new address.

**Say:** "The website no longer depends on one app server. The database is still one machine."

### 5. Replicate the database

One primary accepts writes. Replicas copy that data and serve reads. Reads usually dwarf writes, so replicas outnumber primaries.

You get parallel reads, a surviving copy if a machine is lost, and a place to send queries when one database is down. A dead replica: send reads to the other replicas, or briefly to the primary. A dead primary: promote a replica, then repair the gap, because a replica can be behind.

**Adds:** read scale and a database failover story. Writes still land on one primary.

**Say:** "I will not claim a read sees a write that has not reached the replica yet. Promotion is an operations problem, not a config flag."

### 6. Cache the hot reads

Keep frequent, expensive reads in memory. The app looks there first; on a miss it reads the database and fills the cache. A separate cache tier can grow without the database.

Use it when data is read often and written rarely. Memory is not the system of record: a restart drops it. Set a lifetime. Too short and the database gets hammered; too long and users see stale data. Cache updates and database updates are not one transaction, and that gets harder across regions. One cache node is a single point of failure, so run more than one and leave memory headroom. When the cache is full, evict. Least-recently-used is the usual default; least-frequently-used or first-in-first-out fit other access patterns.

**Adds:** lower latency and a lighter database, in exchange for staleness and another fleet to run.

**Say:** "I name what is cached, the lifetime, and what the app does when the cache is cold or down."

### 7. Put static files on a CDN

Images, CSS, JavaScript, and video come from an edge server near the user. A miss fetches the file from your origin (app server or object storage), then the edge holds it until the lifetime expires. The next viewer nearby is served from that edge.

Skip the CDN for files nobody requests. If the CDN is down, clients must be able to fall back to origin. To change a file before it expires, invalidate it or change the URL (a version query is enough).

**Adds:** static bytes leave your web servers, and distant users stop paying a cross-country trip for every image.

**Say:** "The app still renders dynamic responses. The CDN is for bytes that are the same for many users."

### 8. Make the web tier stateless

If a user's session lives on one app server, every later request must find that same server. Sticky sessions can force that, and they make adding, removing, or losing a server painful.

Move session data to a shared store (relational, Redis-style cache, or a scalable NoSQL store). Any app server can handle any user, so the pool can grow and shrink with load.

**Adds:** horizontal scaling that actually works, because users are not pinned to a box.

**Say:** "Sticky sessions are a workaround. Shared session state is what lets me add a server without moving users."

### 9. More than one data center

Send users to a nearby site with location-aware DNS. If a site fails, send everyone to a healthy one.

Three problems show up immediately: directing traffic, copying data so the failover site has it, and shipping the same build everywhere. Cross-site copy is usually asynchronous, so a failover can serve slightly old data.

**Adds:** survival of a whole site, and lower latency for distant users. Not a clone-and-forget of one site.

**Say:** "I name how traffic moves, how far behind the other site is, and how a deploy reaches both."

### 10. Queue the slow work

Producers append a job. Consumers do it later. The queue holds the job if either side is down or busy. Image work is the usual picture: the web server enqueues a crop or filter, workers pull jobs, and you add workers when the queue grows, not when web traffic grows.

**Adds:** decoupling. The request path stays fast; heavy work scales on its own; a down worker does not drop the job.

**Say:** "The user gets an accepted response. Completion is asynchronous, and I say how they learn it finished."

### 11. Logs, metrics, and automation

Optional on a few servers. Required once you cannot inspect machines by hand.

- Logs: errors per host, then aggregated so you can search.
- Metrics: the machine (CPU, memory, disk), the tier (database, cache), and the product (active users, retention, revenue).
- Automation: build, test, and deploy on each change so every site runs the same thing.

**Adds:** a way to see and change a large system without heroics.

**Say:** "I add this when the fleet is too big to SSH. I mention one business metric, not only CPU."

### 12. Scale the data tier, then shard

A bigger database server goes surprisingly far, and some large sites have lived on one primary. It still has a hardware ceiling, a single point of failure, and a painful price.

Sharding splits rows across servers that share a schema. A key (often a hash of user id) picks the server. The key has to spread load.

Costs worth saying out loud:

- A shard fills up, or one shard gets hot because the key was uneven. You have to move rows. Consistent hashing cuts how much moves when the number of shards changes.
- One famous record can melt its shard. Give that key its own shard, or split it further.
- Joins across shards are a bad idea. Duplicate fields so the query hits one shard.

Move access patterns that do not need relational queries onto a non-relational store so the SQL primary is not doing that work.

**Adds:** data volume beyond one primary family. You pay with operational complexity.

**Say:** "I shard only after replication and cache are not enough. I name the key, the hot key, and the join I can no longer do."

### 13. Past a few million

Same loop, smaller pieces. Split a tier into services when it has more than one reason to scale. Keep redundancy at every tier, cache hot reads, serve static files from a CDN, shard data that outgrows one family, and watch the system with automation.

**Adds:** nothing magical. The interview point is that you iterate on the current bottleneck.

**Say:** "I would not redraw the company on the first pass. Tell me the next limit and I will add the next piece."

## Recite if they ask for the short list

- Stateless web servers
- A spare at every tier
- Cache hot reads
- More than one data center
- Static files on a CDN
- Shard the data tier when one family is the limit
- Split services when one tier has two scaling stories
- Metrics and automated deploys

## Interruptions worth being ready for

- Replica lag, and cache entries that disagree with the database
- What still dies if this one box dies
- Why this step, and why not the next one yet
