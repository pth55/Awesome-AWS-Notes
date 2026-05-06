# Part 7: ALB & NLB - Hands-On Practice Tasks

---

## Quick Theory (Just the Important Bits)

### What is a Load Balancer?

A Load Balancer sits in front of your servers and distributes incoming traffic across them. If one server dies, traffic goes to the healthy one. Simple.

### ALB (Application Load Balancer) - Layer 7

- Works at **HTTP/HTTPS level** (Layer 7 of OSI)
- Can read the **URL path**, **headers**, **query strings** - basically understands HTTP
- Can route based on path: `/home` goes to Server A, `/api` goes to Server B
- **Slower** than NLB (because it inspects HTTP content)
- Gets a **DNS name** (not a static IP)
- **Use when**: You need smart routing based on URL paths, host headers, or HTTP content

### NLB (Network Load Balancer) - Layer 4

- Works at **TCP/UDP level** (Layer 4 of OSI)
- Does NOT read HTTP content - just forwards raw TCP packets
- **Ultra fast** - millions of requests per second, ultra-low latency
- Gets a **static IP** per AZ (or you can attach Elastic IPs)
- **Preserves client IP** (ALB replaces source IP with its own - you need `X-Forwarded-For` header)
- **Use when**: You need raw speed, static IPs, non-HTTP protocols (gaming, IoT, gRPC), or TCP passthrough

### Key Difference in One Line

> ALB = smart routing, understands HTTP, path-based rules
> NLB = dumb-fast forwarding, TCP level, static IPs, preserves client IP

### Target Groups

Both ALB and NLB route traffic to **Target Groups**. A Target Group is just a collection of targets (EC2 instances, IPs, Lambda functions) + a health check config.

- **ALB Target Group**: Protocol is HTTP/HTTPS, health check is HTTP
- **NLB Target Group**: Protocol is TCP/UDP/TLS, health check can be TCP or HTTP

### Important Options You'll See in Console

| Option | What It Means |
|--------|--------------|
| **Scheme: Internet-facing** | LB gets a public IP, accessible from internet |
| **Scheme: Internal** | LB gets private IP only, for internal services |
| **Listeners** | Port + Protocol the LB listens on (e.g., HTTP:80) |
| **Rules** | Conditions that decide which Target Group gets the traffic |
| **Health Check** | LB pings your instances periodically to check if they're alive |
| **Deregistration Delay** | How long to wait before cutting traffic to a draining target (default 300s, set to 30s for testing) |
| **Stickiness** | Same client always goes to same target (disabled by default) |
| **Cross-zone LB** | Distribute traffic evenly across all targets in all AZs (ALB: always on, NLB: off by default) |

---

### ALB Health Check Settings - Deep Dive

When you create a Target Group for an ALB, you configure **health checks**. These are how the ALB decides if an instance is alive and ready to receive traffic. If an instance fails health checks, the ALB **stops sending traffic to it** — this is the core of automatic failover.

Here's every health check field explained using our Todo app as an example:

```
Health Check Configuration (Target Group)
══════════════════════════════════════════════════════════════════════════════

  Health check protocol:    HTTP
  Health check path:        /getTodos
  Health check port:        traffic-port
  Interval:                 30 seconds
  Timeout:                  5 seconds
  Healthy threshold:        5
  Unhealthy threshold:      2
  Success codes:            200

══════════════════════════════════════════════════════════════════════════════
```

#### 1. Health Check Protocol: `HTTP`

The protocol the ALB uses when pinging your instance. Since our app is a plain HTTP server (no HTTPS/SSL), we use `HTTP`. Options:

| Protocol | When to Use |
|----------|-------------|
| **HTTP** | Your app serves plain HTTP (most common for backends behind ALB) |
| **HTTPS** | Your app has its own SSL certificate and terminates TLS itself |

> For ALB Target Groups, you'll almost always use HTTP here. The ALB itself handles HTTPS termination from the internet side — your backend EC2s typically talk plain HTTP to the ALB.

#### 2. Health Check Path: `/getTodos`

The **URL path** the ALB hits on your instance to check if it's healthy. The ALB sends a GET request to this path.

```
What the ALB does every health check cycle:
──────────────────────────────────────────────────────────────────

    ALB  ──── GET http://<instance-IP>:8080/getTodos ────>  EC2

    EC2  ──── 200 OK + JSON response ────────────────────>  ALB

    ALB thinks: "Got 200, this instance is healthy ✓"

──────────────────────────────────────────────────────────────────
```

**Why `/getTodos` instead of `/home`?**
- `/home` only proves the server process is running
- `/getTodos` proves the server is running AND the todo data/logic is accessible
- In real apps, you'd pick a path that tests a meaningful part of your application (e.g., a path that queries the database) so the health check catches more failure modes

**Choosing a good health check path:**
- Pick a path that's **lightweight** (doesn't do heavy computation or slow DB queries)
- Pick a path that **exercises your critical dependencies** (DB connection, cache, etc.)
- NEVER use a path that **modifies data** (no POST-like side effects from a GET)
- NEVER use a path that **requires authentication** (the ALB sends a plain GET, no tokens)
- Common patterns: `/health`, `/healthz`, `/status`, or a lightweight GET endpoint like `/getTodos`

#### 3. Health Check Port: `traffic-port`

Which port the ALB sends the health check request to.

| Option | Meaning |
|--------|---------|
| **traffic-port** (default) | Use the same port that the Target Group is configured to receive traffic on |
| **Override (custom port)** | Specify a different port number |

With `traffic-port`: If your Target Group forwards traffic to port `8080`, the health check also hits port `8080`. This is what you want 99% of the time — you're checking the same port that serves real traffic.

**When would you override?** If your app exposes a dedicated health endpoint on a separate port (e.g., main app on 8080 but health/metrics on 9090). This is uncommon for simple setups.

#### 4. Interval: `30 seconds`

How often the ALB sends a health check request to each target.

```
Timeline with 30-second interval:
──────────────────────────────────────────────────────────────────

 0s       30s      60s      90s      120s     150s
 │         │        │        │         │        │
 ▼         ▼        ▼        ▼         ▼        ▼
 PING     PING     PING     PING     PING     PING
 200✓     200✓     200✓     FAIL✗    FAIL✗    200✓
                                       │
                              Unhealthy threshold (2) reached!
                              ALB stops sending traffic here

──────────────────────────────────────────────────────────────────
```

- **30 seconds** is the default — ALB pings once every 30 seconds
- **Lower interval** (e.g., 10s) = faster detection of failures, but more load on your instances
- **Higher interval** (e.g., 60s) = less load, but slower to detect failures
- For production: 30s is a good balance
- For testing/labs: 10s so you don't wait forever to see health status changes

> **Math**: If interval = 30s and unhealthy threshold = 2, it takes **minimum 60 seconds** (2 × 30s) to mark an instance unhealthy after it starts failing.

#### 5. Timeout: `5 seconds`

How long the ALB waits for a response before considering that single health check **failed**.

```
What happens on each health check ping:
──────────────────────────────────────────────────────────────────

  ALB sends GET /getTodos
       │
       ├── Response within 5s?  →  Check the status code (200 = pass)
       │
       └── No response in 5s?   →  This check is a FAIL ✗
                                    (counts toward unhealthy threshold)

──────────────────────────────────────────────────────────────────
```

- **Timeout must be LESS than interval** (5s timeout < 30s interval — otherwise the next ping starts before the current one finishes)
- If your app is fast (like our Node.js Todo app), 5s is generous — it responds in milliseconds
- If your health check path queries a database or external service, it might need more time
- **Don't set timeout too low** (e.g., 1s) — network latency spikes could cause false failures

#### 6. Healthy Threshold: `5`

How many **consecutive successful** health checks an instance needs before the ALB considers it "healthy" and starts sending it traffic.

```
Instance recovering after being unhealthy (threshold = 5):
──────────────────────────────────────────────────────────────────

  Check 1:  200 ✓  (1/5 — not healthy yet)
  Check 2:  200 ✓  (2/5 — not healthy yet)
  Check 3:  200 ✓  (3/5 — not healthy yet)
  Check 4:  200 ✓  (4/5 — not healthy yet)
  Check 5:  200 ✓  (5/5 — NOW HEALTHY! ALB starts sending traffic)

  With interval = 30s, this takes: 5 × 30s = 150 seconds (2.5 minutes)

──────────────────────────────────────────────────────────────────
```

**Why set it to 5?**
- **Higher threshold = more conservative**. The instance must prove it's consistently healthy, not just a one-off success. This prevents a flapping instance (up, down, up, down) from receiving traffic prematurely.
- **Lower threshold** (e.g., 2) = faster recovery. Instance gets traffic sooner after coming back up.
- **Trade-off**: Higher threshold is safer but means longer recovery time. In production with `healthy threshold = 5` and `interval = 30s`, a recovered instance waits 2.5 minutes before getting traffic.
- For testing/labs: Use 2 so you're not waiting minutes for targets to become healthy

#### 7. Unhealthy Threshold: `2`

How many **consecutive failed** health checks before the ALB considers the instance "unhealthy" and **stops sending it traffic**.

```
Instance failing (threshold = 2):
──────────────────────────────────────────────────────────────────

  Check 1:  200 ✓   (healthy, normal)
  Check 2:  200 ✓   (healthy, normal)
  Check 3:  FAIL ✗  (1/2 failures — still healthy, could be a blip)
  Check 4:  FAIL ✗  (2/2 failures — UNHEALTHY! ALB stops sending traffic)

  With interval = 30s, detection takes: 2 × 30s = 60 seconds

──────────────────────────────────────────────────────────────────
```

**Why set it to 2?**
- **Lower threshold = faster failure detection**. As soon as 2 checks fail in a row, the instance is pulled from rotation. Users stop getting errors quickly.
- **Threshold of 1** would be too aggressive — a single network hiccup or GC pause could cause a blip, and you'd unnecessarily pull a healthy instance.
- **Threshold of 2** is the sweet spot: tolerates one transient failure but catches real outages within 2 check cycles.

> **Notice the asymmetry**: Unhealthy threshold (2) < Healthy threshold (5). This is intentional! You want to **pull unhealthy instances out FAST** (2 checks = 60s) but **bring them back SLOWLY** (5 checks = 150s). It's the same philosophy as a circuit breaker: fail fast, recover cautiously.

#### 8. Success Codes: `200`

The HTTP status code(s) the ALB considers a "successful" health check response.

| Setting | Meaning |
|---------|---------|
| `200` | ONLY 200 OK counts as healthy |
| `200-299` | Any 2xx status code counts as healthy |
| `200,301` | Either 200 or 301 counts as healthy |

**Why just `200`?**
- Our `/getTodos` endpoint returns `200` when everything is working
- If it returns `500` (server error), `503` (unavailable), or `404` — that means something is wrong and the instance should be marked unhealthy
- Being strict with `200` means any unexpected behavior is caught

**When to use a range like `200-299`?**
- If your health check endpoint might return `204 No Content` (valid but no body)
- If your app uses various 2xx codes for success responses
- Generally, keep it strict (`200`) unless you have a reason to broaden it

#### Putting It All Together - The Full Health Check Lifecycle

```
═══════════════════════════════════════════════════════════════════════════
 FULL LIFECYCLE: Instance Failure and Recovery
 Settings: Interval=30s, Timeout=5s, Healthy=5, Unhealthy=2, Codes=200
═══════════════════════════════════════════════════════════════════════════

 PHASE 1: NORMAL OPERATION
 ─────────────────────────
   0:00  GET /getTodos → 200 ✓  (healthy, receiving traffic)
   0:30  GET /getTodos → 200 ✓  (healthy, receiving traffic)
   1:00  GET /getTodos → 200 ✓  (healthy, receiving traffic)

 PHASE 2: APP CRASHES (someone did `pkill node`)
 ────────────────────────────────────────────────
   1:30  GET /getTodos → TIMEOUT after 5s ✗  (1/2 failures)
         └─ Still marked healthy, still receiving traffic
   2:00  GET /getTodos → TIMEOUT after 5s ✗  (2/2 failures)
         └─ NOW UNHEALTHY! ALB stops sending traffic to this instance
         └─ All traffic goes to the other healthy instance(s)

 PHASE 3: APP RESTARTED (someone ran `node app.js` again)
 ─────────────────────────────────────────────────────────
   2:30  GET /getTodos → 200 ✓  (1/5 successes — still unhealthy)
   3:00  GET /getTodos → 200 ✓  (2/5 successes — still unhealthy)
   3:30  GET /getTodos → 200 ✓  (3/5 successes — still unhealthy)
   4:00  GET /getTodos → 200 ✓  (4/5 successes — still unhealthy)
   4:30  GET /getTodos → 200 ✓  (5/5 successes)
         └─ NOW HEALTHY AGAIN! ALB resumes sending traffic

 TIMELINE SUMMARY:
   Failure detection:  ~60 seconds  (2 checks × 30s interval)
   Recovery time:      ~150 seconds (5 checks × 30s interval)
   Total downtime for this instance: ~3.5 minutes

═══════════════════════════════════════════════════════════════════════════
```

> **Key Takeaway**: The ALB doesn't just blindly spray traffic — it continuously monitors every instance and only routes to healthy ones. If an instance dies, traffic shifts to survivors within ~60 seconds. When it comes back, the ALB cautiously waits ~150 seconds of consistent health before trusting it again. This is automatic — no human intervention needed.

---

## VPC Setup: Create `my-vpc-1` (Prerequisite for All Tasks)

> **Do this FIRST before anything else.** All 3 tasks run inside this VPC. If you already have `my-vpc-1` from a previous lab, skip to Pre-Setup.

This is a simpler VPC than the one in Part 2 — we only need **2 public subnets** across 2 AZs (no private subnets, no NAT Gateway). The ALB requires subnets in at least 2 different AZs.

### What We're Building

```
AWS REGION: ap-south-1 (Mumbai)
┌──────────────────────────────────────────────────────────────────┐
│  VPC: my-vpc-1  (10.0.0.0/16)                                   │
│                                                                  │
│  ┌─────────────────────────┐   ┌─────────────────────────┐      │
│  │  my-vpc-1-public-1      │   │  my-vpc-1-public-2      │      │
│  │  10.0.1.0/24            │   │  10.0.2.0/24            │      │
│  │  ap-south-1a            │   │  ap-south-1b            │      │
│  │                         │   │                         │      │
│  │  ┌───────────────────┐  │   │  ┌───────────────────┐  │      │
│  │  │  app-server-1     │  │   │  │  app-server-2     │  │      │
│  │  │  10.0.1.x         │  │   │  │  10.0.2.x         │  │      │
│  │  └───────────────────┘  │   │  └───────────────────┘  │      │
│  └─────────────────────────┘   └─────────────────────────┘      │
│                                                                  │
│                    ┌──────────────┐                               │
│                    │  IGW         │                               │
│                    │  my-vpc-1-igw│                               │
│                    └──────┬───────┘                               │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                        INTERNET
```

---

### Step 1: Create the VPC

1. Go to **VPC Console > Your VPCs > Create VPC**
2. Settings:

```
VPC Settings:
────────────────────────────────────────────
Resources to create:  VPC only
Name tag:             my-vpc-1
IPv4 CIDR block:      10.0.0.0/16
IPv6 CIDR block:      No IPv6 CIDR block
Tenancy:              Default
────────────────────────────────────────────
```

3. Click **Create VPC**

> **Why 10.0.0.0/16?** It gives 65,536 addresses — way more than we need, but it's the standard starting point for lab VPCs and leaves plenty of room for subnets. It also won't overlap with the Part 2 VPC (`192.168.1.0/24`) if you kept that.

---

### Step 2: Create Two Public Subnets

We need exactly 2 public subnets in **different Availability Zones**. ALBs require at least 2 AZs.

#### Subnet 1: `my-vpc-1-public-1`

1. Go to **VPC Console > Subnets > Create Subnet**
2. Settings:

```
Settings:
────────────────────────────────────────────────────
VPC ID:                  my-vpc-1
Subnet name:             my-vpc-1-public-1
Availability Zone:       ap-south-1a
IPv4 subnet CIDR block:  10.0.1.0/24
────────────────────────────────────────────────────
```

#### Subnet 2: `my-vpc-1-public-2`

1. Same flow, different settings:

```
Settings:
────────────────────────────────────────────────────
VPC ID:                  my-vpc-1
Subnet name:             my-vpc-1-public-2
Availability Zone:       ap-south-1b          ← DIFFERENT AZ
IPv4 subnet CIDR block:  10.0.2.0/24
────────────────────────────────────────────────────
```

> **Why different AZs?** ALB places a load balancer node in each subnet/AZ you select. Having subnets in at least 2 AZs gives high availability — if one AZ fails, the ALB still routes traffic to the surviving AZ.

---

### Step 3: Create and Attach an Internet Gateway

1. Go to **VPC Console > Internet Gateways > Create Internet Gateway**
2. Name: `my-vpc-1-igw`
3. Click **Create**
4. Select `my-vpc-1-igw` > **Actions > Attach to VPC** > Select `my-vpc-1` > **Attach**

---

### Step 4: Create a Public Route Table and Associate Subnets

#### Create the route table

1. Go to **VPC Console > Route Tables > Create Route Table**
2. Name: `my-vpc-1-public-rt`
3. VPC: `my-vpc-1`
4. Click **Create**

#### Add the internet route

1. Select `my-vpc-1-public-rt` > **Routes tab > Edit Routes > Add Route**

```
Destination:   0.0.0.0/0
Target:        Internet Gateway → my-vpc-1-igw
```

2. Click **Save Changes**

#### Associate both public subnets

1. Select `my-vpc-1-public-rt` > **Subnet Associations tab > Edit Subnet Associations**
2. Check both:
   - `my-vpc-1-public-1`
   - `my-vpc-1-public-2`
3. Click **Save Associations**

---

### Step 5: Enable Auto-Assign Public IP on Both Subnets

Instances need public IPs so you can SSH into them and so the internet can reach the ALB.

1. Go to **VPC Console > Subnets**
2. Select `my-vpc-1-public-1` > **Actions > Edit Subnet Settings**
3. Check **Enable auto-assign public IPv4 address** > **Save**
4. Repeat for `my-vpc-1-public-2`

---

### Verify the VPC Setup

Before moving on, confirm:

| Component | Expected State |
|-----------|---------------|
| VPC `my-vpc-1` | Created, CIDR `10.0.0.0/16` |
| Subnet `my-vpc-1-public-1` | `10.0.1.0/24`, `ap-south-1a`, auto-assign public IP **ON** |
| Subnet `my-vpc-1-public-2` | `10.0.2.0/24`, `ap-south-1b`, auto-assign public IP **ON** |
| IGW `my-vpc-1-igw` | Attached to `my-vpc-1` |
| Route Table `my-vpc-1-public-rt` | `0.0.0.0/0 → IGW`, associated with both subnets |

> **Note:** You'll select these subnets by name when launching EC2 instances and creating the ALB in later steps.

---
---

## Pre-Setup: The App + EC2 Instances + Security Groups

> **Do this ONCE. All 3 tasks reuse these EC2 instances.**

### Step 0: The Simple Node.js Todo App

This is the app you'll run on BOTH EC2 instances. It has 3 routes:

```javascript
// app.js
const http = require('http');

// Fetch instance IP using IMDSv2 (EC2 metadata service)
// IMDSv2 requires a token - simple GET (IMDSv1) is blocked on newer instances
function getInstanceIP() {
  return new Promise((resolve) => {
    // Step 1: Get a token via PUT request
    const tokenReq = http.request('http://169.254.169.254/latest/api/token', {
      method: 'PUT',
      headers: { 'X-aws-ec2-metadata-token-ttl-seconds': '21600' }
    }, (tokenRes) => {
      let token = '';
      tokenRes.on('data', (d) => token += d);
      tokenRes.on('end', () => {
        // Step 2: Use token to fetch private IP
        const ipReq = http.request('http://169.254.169.254/latest/meta-data/local-ipv4', {
          headers: { 'X-aws-ec2-metadata-token': token }
        }, (ipRes) => {
          let ip = '';
          ipRes.on('data', (d) => ip += d);
          ipRes.on('end', () => resolve(ip || 'unknown'));
        });
        ipReq.on('error', () => resolve('unknown'));
        ipReq.end();
      });
    });
    tokenReq.on('error', () => resolve('unknown'));
    tokenReq.end();
  });
}

async function main() {
  const INSTANCE_IP = await getInstanceIP();

  const todos = [
    { id: 1, task: "Learn ALB", done: false },
    { id: 2, task: "Learn NLB", done: false },
    { id: 3, task: "Build multi-AZ setup", done: false }
  ];

  const server = http.createServer((req, res) => {
    res.setHeader('Content-Type', 'application/json');

    if (req.url === '/' || req.url === '/home') {
      res.end(JSON.stringify({
        message: "Hello from EC2!",
        instance_ip: INSTANCE_IP,
        route: req.url
      }, null, 2));
    }
    else if (req.url === '/getTodos') {
      res.end(JSON.stringify({
        todos: todos,
        instance_ip: INSTANCE_IP,
        route: "/getTodos"
      }, null, 2));
    }
    else {
      res.statusCode = 404;
      res.end(JSON.stringify({
        error: "Route not found",
        instance_ip: INSTANCE_IP,
        available_routes: ["/home", "/getTodos"]
      }, null, 2));
    }
  });

  server.listen(8080, () => {
    console.log(`Server running on port 8080 | Instance IP: ${INSTANCE_IP}`);
  });
}

main();
```

> **Why IMDSv2?** Newer EC2 instances enforce IMDSv2 by default, which requires a token (PUT request first, then GET with that token). The old approach using `os.networkInterfaces()['eth0']` or simple GET to metadata service will return `unknown` or `401 Unauthorized`. This code handles it correctly.

Save this somewhere on your local machine - you'll paste it into both EC2s.

---

### Step 1: Create Security Groups

You need **2 Security Groups** in your VPC (`my-vpc-1`):

#### SG 1: `alb-sg` (for the Load Balancer)

1. Go to **VPC Console > Security Groups > Create Security Group**
2. Name: `alb-sg`
3. Description: `Security group for ALB - allows HTTP from internet`
4. VPC: `my-vpc-1`
5. **Inbound Rules:**

   | Type | Protocol | Port | Source | Description |
   |------|----------|------|--------|-------------|
   | HTTP | TCP | 80 | 0.0.0.0/0 | Allow HTTP from anywhere |

6. **Outbound Rules:** Leave default (All traffic - 0.0.0.0/0)
7. Click **Create**

#### SG 2: `ec2-app-sg` (for both EC2 instances)

1. Create another Security Group
2. Name: `ec2-app-sg`
3. Description: `Security group for app EC2s - only ALB can reach port 8080`
4. VPC: `my-vpc-1`
5. **Inbound Rules:**

   | Type | Protocol | Port | Source | Description |
   |------|----------|------|--------|-------------|
   | Custom TCP | TCP | 8080 | `alb-sg` (select the SG) | Only ALB can hit app port |

   > **IMPORTANT**: For the port 8080 rule, the Source is the **Security Group ID** of `alb-sg`, NOT an IP address. This means only traffic originating from the ALB (which uses `alb-sg`) can reach your EC2s on 8080. This is the proper way to secure backend instances.
   >
   > **No SSH rule needed** if you're using SSM Session Manager to connect (recommended). Add SSH rule only if you prefer SSH access.

6. **Outbound Rules:** Leave default (All traffic)
7. Click **Create**

---

### Step 2: Launch Two EC2 Instances

#### EC2-1 (in subnet: `my-vpc-1-public-1`)

1. Go to **EC2 Console > Launch Instance**
2. **Name**: `app-server-1`
3. **AMI**: Amazon Linux 2023 (free tier eligible)
4. **Instance Type**: `t2.micro`
5. **Key Pair**: Select your existing key pair (or create one — not needed if using SSM only)
6. **Network Settings** - Click **Edit**:
   - VPC: `my-vpc-1`
   - Subnet: `my-vpc-1-public-1`
   - Auto-assign Public IP: **Enable**
   - Security Group: Select **existing** > `ec2-app-sg`
7. **Advanced Details** (expand):
   - IAM instance profile: Select a role with `AmazonSSMManagedInstanceCore` policy (so you can connect via Session Manager without SSH)
8. Click **Launch Instance**

#### EC2-2 (in subnet: `my-vpc-1-public-2`)

1. Same as above but:
   - **Name**: `app-server-2`
   - Subnet: `my-vpc-1-public-2`
   - Everything else identical

---

### Step 3: Deploy the App on BOTH EC2s

Connect to **each** EC2 via **SSM Session Manager** (EC2 Console > Select instance > Connect > Session Manager) and run:

```bash
# Install Node.js
sudo yum install -y nodejs

# Create app directory
mkdir ~/todoapp && cd ~/todoapp

# Create the app file (uses IMDSv2 for fetching instance IP)
cat << 'EOF' > app.js
const http = require('http');

function getInstanceIP() {
  return new Promise((resolve) => {
    const tokenReq = http.request('http://169.254.169.254/latest/api/token', {
      method: 'PUT',
      headers: { 'X-aws-ec2-metadata-token-ttl-seconds': '21600' }
    }, (tokenRes) => {
      let token = '';
      tokenRes.on('data', (d) => token += d);
      tokenRes.on('end', () => {
        const ipReq = http.request('http://169.254.169.254/latest/meta-data/local-ipv4', {
          headers: { 'X-aws-ec2-metadata-token': token }
        }, (ipRes) => {
          let ip = '';
          ipRes.on('data', (d) => ip += d);
          ipRes.on('end', () => resolve(ip || 'unknown'));
        });
        ipReq.on('error', () => resolve('unknown'));
        ipReq.end();
      });
    });
    tokenReq.on('error', () => resolve('unknown'));
    tokenReq.end();
  });
}

async function main() {
  const INSTANCE_IP = await getInstanceIP();

  const todos = [
    { id: 1, task: "Learn ALB", done: false },
    { id: 2, task: "Learn NLB", done: false },
    { id: 3, task: "Build multi-AZ setup", done: false }
  ];

  const server = http.createServer((req, res) => {
    res.setHeader('Content-Type', 'application/json');

    if (req.url === '/' || req.url === '/home') {
      res.end(JSON.stringify({
        message: "Hello from EC2!",
        instance_ip: INSTANCE_IP,
        route: req.url
      }, null, 2));
    }
    else if (req.url === '/getTodos') {
      res.end(JSON.stringify({
        todos: todos,
        instance_ip: INSTANCE_IP,
        route: "/getTodos"
      }, null, 2));
    }
    else {
      res.statusCode = 404;
      res.end(JSON.stringify({
        error: "Route not found",
        instance_ip: INSTANCE_IP,
        available_routes: ["/home", "/getTodos"]
      }, null, 2));
    }
  });

  server.listen(8080, () => {
    console.log(`Server running on port 8080 | Instance IP: ${INSTANCE_IP}`);
  });
}

main();
EOF

# Run the app in background (survives terminal close)
nohup node app.js > app.log 2>&1 &

# Verify it's running
curl http://localhost:8080/home
```

You should see output like:
```json
{
  "message": "Hello from EC2!",
  "instance_ip": "10.0.x.x",
  "route": "/home"
}
```

> **Do this on BOTH EC2 instances.** Each will show its own private IP.

---

### Step 4: Verify Both Apps Are Running

From **EC2-1**:
```bash
curl http://localhost:8080/home
curl http://localhost:8080/getTodos
```

From **EC2-2**:
```bash
curl http://localhost:8080/home
curl http://localhost:8080/getTodos
```

Both should respond with their respective private IPs. You're ready for the tasks!

> **NOTE**: You CANNOT hit these from browser yet because `ec2-app-sg` only allows traffic from `alb-sg`, not from the internet. That's the whole point - only the load balancer can talk to them.

---
---

## TASK 1: Basic ALB - Round Robin Traffic Between Two EC2s

**Goal**: Create an ALB that listens on port 80 and distributes traffic evenly across both EC2 instances on port 8080. Hit refresh and watch the `instance_ip` alternate between the two.

---

### Step 1: Create a Target Group

1. Go to **EC2 Console > Target Groups > Create Target Group**
2. **Choose target type**: `Instances`
3. **Target group name**: `tg-basic-alb`
4. **Protocol**: `HTTP` | **Port**: `8080`
   > Why 8080? Because our app listens on 8080. The LB will forward to this port.
5. **VPC**: `my-vpc-1`
6. **Health check settings**:
   - Protocol: `HTTP`
   - Path: `/home`
   > The ALB will hit `/home` on each instance every 30s. If it gets a 200 OK, the instance is "healthy". If it fails multiple times, ALB stops sending traffic to it.
7. **Advanced Health Check Settings** (expand it):
   - Healthy threshold: `2` (2 consecutive successes = healthy)
   - Unhealthy threshold: `3` (3 consecutive failures = unhealthy)
   - Timeout: `5` seconds
   - Interval: `10` seconds (check every 10s - faster for testing)
   - Success codes: `200`
8. Click **Next**
9. **Register Targets**:
   - Select BOTH `app-server-1` and `app-server-2`
   - Click **Include as pending below**
   - Verify port is `8080` for both
10. Click **Create Target Group**

---

### Step 2: Create the ALB

1. Go to **EC2 Console > Load Balancers > Create Load Balancer**
2. Choose **Application Load Balancer** > Create
3. **Basic Configuration**:
   - Name: `my-basic-alb`
   - Scheme: `Internet-facing` (we want to access it from browser)
   - IP address type: `IPv4`
4. **Network Mapping**:
   - VPC: `my-vpc-1`
   - Mappings: Select **both subnets**:
     - `my-vpc-1-public-1`
     - `my-vpc-1-public-2`
   > **Why select subnets?** The ALB places a node in each selected subnet. Even though both are in the same AZ, the ALB needs at least one subnet selected. Selecting both means the ALB can reach instances in both subnets.
5. **Security Group**: Select `alb-sg` (remove the default SG if auto-selected)
6. **Listeners and Routing**:
   - Listener: `HTTP` : `80`
   - Default action: Forward to > `tg-basic-alb`
   > This means: "Any traffic that hits the ALB on port 80 gets forwarded to the target group `tg-basic-alb`"
7. Click **Create Load Balancer**

---

### Step 3: Wait & Test

1. Go to **Load Balancers**, select `my-basic-alb`
2. Wait for **State** to become `Active` (takes 2-3 minutes)
3. Copy the **DNS Name** (looks like `my-basic-alb-123456.ap-south-1.elb.amazonaws.com`)
4. Go to **Target Groups > tg-basic-alb > Targets tab**
   - Wait until both targets show **Status: healthy**
   - If they show `unhealthy`, check:
     - Is the app running? (Connect via SSM and run `curl localhost:8080/home`)
     - Is the SG correct? (ec2-app-sg must allow 8080 from alb-sg)

5. **Test in browser:**

```
http://<ALB-DNS-NAME>/home
http://<ALB-DNS-NAME>/getTodos
```

6. **Hit refresh 5-10 times** on `/home` - watch the `instance_ip` alternate between your two EC2 private IPs!

> **What's Happening?** The ALB uses **round-robin** algorithm by default. Each new request goes to the next target in the list. So you'll see the IP flip between the two instances.

---

### What to Observe

- Both `/home` and `/getTodos` work through the ALB
- The `instance_ip` in the response changes as ALB alternates between instances
- You're accessing port 80 (ALB) but the app runs on 8080 (EC2) - the ALB handles the port translation
- Try killing the app on one EC2 (`pkill node`) - after health checks fail, ALB will only send to the healthy one

---

### Cleanup for Task 1

> **Do NOT delete EC2 instances or Security Groups - you need them for Task 2 and 3!**

1. Delete the ALB: **Load Balancers > Select `my-basic-alb` > Actions > Delete**
2. Delete the Target Group: **Target Groups > Select `tg-basic-alb` > Actions > Delete**
   - (You must delete the ALB first, then the TG)

---
---

## TASK 2: ALB Path-Based Routing

**Goal**: Route `/` and `/home` to EC2-1 ONLY, and `/getTodos` to EC2-2 ONLY. No more round-robin - specific paths go to specific servers.

> **Why is this useful?** In real architectures, your frontend might run on one set of servers and your API on another. Path-based routing lets a single ALB handle both without needing separate domain names.

---

### Step 1: Create TWO Target Groups

You need separate target groups because each path points to a different set of instances.

#### Target Group A: `tg-home-routes`

1. **EC2 Console > Target Groups > Create Target Group**
2. Target type: `Instances`
3. Name: `tg-home-routes`
4. Protocol: `HTTP` | Port: `8080`
5. VPC: `my-vpc-1`
6. Health check path: `/home`
7. Advanced health check: Interval `10s` (same as before)
8. Next > Register targets:
   - Select ONLY `app-server-1`
   - Include as pending, verify port 8080
9. Create

#### Target Group B: `tg-todos-route`

1. Same process
2. Name: `tg-todos-route`
3. Protocol: `HTTP` | Port: `8080`
4. Health check path: `/getTodos`
5. Register targets:
   - Select ONLY `app-server-2`
6. Create

---

### Step 2: Create the ALB

1. **EC2 Console > Load Balancers > Create Load Balancer > ALB**
2. Name: `my-path-alb`
3. Scheme: `Internet-facing`
4. Network mapping: Select BOTH subnets (same as Task 1)
5. Security Group: `alb-sg`
6. Listener: `HTTP` : `80`
7. Default action: Forward to > `tg-home-routes`
   > The default action catches anything that doesn't match any specific rule. We'll point `/` and `/home` here.
8. Create Load Balancer

---

### Step 3: Add Path-Based Routing Rules

This is the key difference from Task 1!

1. Go to **Load Balancers > Select `my-path-alb`**
2. Go to **Listeners tab**
3. Click on the **HTTP:80** listener
4. You'll see the **Rules** section with one default rule

#### Add Rule for `/getTodos`

5. Click **Add Rule** (or **Manage Rules** depending on console version)
6. **Name**: `todos-route`
7. Click **Add Condition**:
   - Condition type: **Path**
   - Path pattern: `/getTodos`
   > This means: "If the request URL path matches `/getTodos`, apply this rule's action"
8. Click **Confirm**
9. Click **Add Action**:
   - Action type: **Forward to target group**
   - Target group: `tg-todos-route`
10. **Priority**: `1` (lower number = evaluated first)
11. Click **Create**

> **How Rules Work**: The ALB evaluates rules by priority (lowest number first). When a request comes in, it checks each rule's condition. The first rule that matches wins. If nothing matches, the **default action** is used.

> **Our Setup**:
> - Priority 1: Path = `/getTodos` --> Forward to `tg-todos-route` (EC2-2)
> - Default: Everything else --> Forward to `tg-home-routes` (EC2-1)

---

### Step 4: Wait & Test

1. Wait for ALB state = `Active`
2. Check **both target groups** have healthy targets
3. Copy the ALB DNS name

**Test:**

```
http://<ALB-DNS-NAME>/home       --> Should ALWAYS show EC2-1's IP
http://<ALB-DNS-NAME>/           --> Should ALWAYS show EC2-1's IP
http://<ALB-DNS-NAME>/getTodos   --> Should ALWAYS show EC2-2's IP
```

4. **Refresh each URL 10 times** - the IP should NOT change because each path is pinned to one specific instance!

---

### What to Observe

- `/home` and `/` ALWAYS return EC2-1's IP (no flip-flopping)
- `/getTodos` ALWAYS returns EC2-2's IP
- Compare this to Task 1 where IPs alternated on every refresh
- Try hitting a random path like `/random` - it goes to the default rule (EC2-1)
- This is how microservices work: one ALB, many backend services, routing by path

---

### Cleanup for Task 2

1. Delete ALB `my-path-alb`
2. Delete Target Group `tg-home-routes`
3. Delete Target Group `tg-todos-route`

---
---

## TASK 3: Network Load Balancer (NLB) - TCP Level Load Balancing

**Goal**: Create an NLB that forwards raw TCP traffic on port 80 to both EC2s on port 8080. Observe how NLB differs from ALB: no path awareness, preserves client IP, and you get a static IP.

> **Key Difference**: NLB doesn't understand HTTP. It just forwards TCP packets. So you can't do path-based routing with NLB. But it's faster and gives you static IPs.

---

### Step 1: Update Security Group for NLB

**IMPORTANT**: NLB works differently from ALB with security groups!

> **NLB does NOT have its own security group.** Unlike ALB, NLB is transparent at the network level. Traffic appears to come from the **original client IP** (NLB preserves source IP). This means your `ec2-app-sg` rule that says "allow 8080 from `alb-sg`" **WON'T WORK** for NLB because NLB doesn't use a security group.

You need to update `ec2-app-sg`:

1. Go to **VPC Console > Security Groups > `ec2-app-sg`**
2. **Edit Inbound Rules**
3. **Add a new rule** (don't remove the alb-sg rule - you might need it later):

   | Type | Protocol | Port | Source | Description |
   |------|----------|------|--------|-------------|
   | Custom TCP | TCP | 8080 | 0.0.0.0/0 | Allow NLB traffic (NLB preserves client IP) |

   > **Why 0.0.0.0/0?** Because NLB preserves the client's real IP. The traffic arriving at your EC2 looks like it came directly from the user's browser IP, not from the NLB. So you can't restrict by NLB's IP. In production, you'd use NLB's **private IP range** or a **CIDR of your VPC** instead of 0.0.0.0/0. For this lab, 0.0.0.0/0 is fine.

4. **Save Rules**

---

### Step 2: Create a Target Group (TCP this time!)

1. **EC2 Console > Target Groups > Create Target Group**
2. Target type: `Instances`
3. Name: `tg-nlb-tcp`
4. **Protocol: `TCP`** | Port: `8080`
   > Notice: TCP, not HTTP! NLB works at Layer 4.
5. VPC: `my-vpc-1`
6. **Health Check**:
   - Health check protocol: `HTTP`
     > Even though the TG protocol is TCP, you can still use HTTP health checks. This is useful because TCP health check only verifies the port is open, but HTTP health check verifies your app actually responds.
   - Path: `/home`
   - Advanced: Interval `10s`, Healthy threshold `2`, Unhealthy threshold `3`
7. Next > Register targets:
   - Select BOTH `app-server-1` and `app-server-2`
   - Port: `8080`
8. Create

---

### Step 3: Create the NLB

1. **EC2 Console > Load Balancers > Create Load Balancer**
2. Choose **Network Load Balancer** > Create
3. **Basic Configuration**:
   - Name: `my-nlb`
   - Scheme: `Internet-facing`
   - IP address type: `IPv4`
4. **Network Mapping**:
   - VPC: `my-vpc-1`
   - Select both subnets:
     - `my-vpc-1-public-1`
     - `my-vpc-1-public-2`
   > **Notice**: For each subnet/AZ, you'll see an option for **IPv4 address**: `Assigned by AWS` or `Use an Elastic IP address`. AWS assigns one static IP per selected subnet. This is a big NLB advantage - you get predictable IPs.
5. **Security Group**: NLB now supports optional SGs (newer feature). You can select `alb-sg` or skip it.
   > If you attach a SG to NLB, then it behaves like ALB (source IP becomes NLB's IP). If you DON'T attach a SG, client IP is preserved. **For this task, do NOT attach a security group** so you can observe client IP preservation.
6. **Listeners**:
   - Protocol: `TCP` | Port: `80`
   - Default action: Forward to > `tg-nlb-tcp`
7. Click **Create Load Balancer**

---

### Step 4: Wait & Test

1. Wait for NLB state = `Active` (NLB can take 2-5 minutes)
2. Check target group `tg-nlb-tcp` - both targets should be `healthy`
3. Copy the NLB **DNS Name**

**Test in browser:**

```
http://<NLB-DNS-NAME>/home
http://<NLB-DNS-NAME>/getTodos
http://<NLB-DNS-NAME>/
```

4. **Refresh multiple times** - watch the `instance_ip` alternate (round-robin, same as Task 1)

---

### Step 5: Observe the Differences from ALB

| Observation | ALB (Task 1) | NLB (Task 3) |
|-------------|-------------|-------------|
| **DNS Name** | Long DNS, no static IP | DNS name, but also has **static IPs** per subnet |
| **Path routing possible?** | Yes (Task 2) | No - NLB doesn't understand HTTP paths |
| **Client IP preserved?** | No (ALB replaces with its own IP) | Yes (your app sees real client IP) |
| **Speed** | Good | Faster (no HTTP parsing) |
| **SG on LB** | Required | Optional (newer feature) |
| **Protocols** | HTTP/HTTPS only | TCP, UDP, TLS |

**To see the static IPs:**
1. Go to **Load Balancers > `my-nlb`**
2. Under **Network mapping**, you'll see the assigned IPv4 address for each subnet
3. You can actually use these IPs directly in the browser instead of the DNS name!

**To verify client IP preservation** (connect to an EC2 via SSM):
```bash
# Check app log to see incoming connections
tail -f ~/todoapp/app.log

# Or modify app.js to log request headers:
# In the request handler, add:
# console.log('Client IP:', req.socket.remoteAddress);
```

---

### Step 6: Try Something Cool - Use Static IP

```
http://<NLB-STATIC-IP>/home
```

This works! With ALB, you can only use the DNS name. With NLB, you can use the actual IP address. This is critical for:
- DNS-less environments
- Whitelisting by IP in firewalls
- IoT devices that can't do DNS resolution

---

### Final Cleanup

1. Delete NLB: `my-nlb`
2. Delete Target Group: `tg-nlb-tcp`
3. **Revert Security Group**: Remove the `0.0.0.0/0` port 8080 rule from `ec2-app-sg`
4. If you're done with everything:
   - Terminate both EC2 instances
   - Delete `ec2-app-sg` and `alb-sg`

---
---

## Quick Reference: ALB vs NLB Decision Matrix

```
Need path-based routing?        --> ALB
Need host-based routing?        --> ALB
Need WebSocket support?         --> ALB (or NLB with TCP)
Need static IP?                 --> NLB
Need ultra-low latency?         --> NLB
Need to preserve client IP?     --> NLB (without SG)
Non-HTTP protocol (TCP/UDP)?    --> NLB
Need both smart routing + 
  static IP?                    --> NLB in front, ALB behind (advanced pattern)
```

## What's Next?

After mastering these single-AZ tasks, the next steps are:
1. **Multi-AZ Setup**: Place subnets in different AZs (ap-south-1a, ap-south-1c) for high availability
2. **HTTPS/TLS**: Add certificates and HTTPS listeners
3. **Auto Scaling Groups**: Let AWS automatically add/remove EC2s based on traffic
4. **NLB + ALB Combo**: Use NLB for static IPs in front, ALB behind for smart routing
5. **Cross-Region**: Use Route 53 + multiple ALBs across regions for global traffic management
