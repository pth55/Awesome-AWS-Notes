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

## Pre-Setup: The App + EC2 Instances + Security Groups

> **Do this ONCE. All 3 tasks reuse these EC2 instances.**

### Step 0: The Simple Node.js Todo App

This is the app you'll run on BOTH EC2 instances. It has 3 routes:

```javascript
// app.js
const http = require('http');
const os = require('os');

// Get instance's private IP
const INSTANCE_IP = os.networkInterfaces()['eth0']?.[0]?.address || 'unknown';

// Simple in-memory todos
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
```

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
   | SSH | TCP | 22 | Your IP (My IP) | SSH access for setup |

   > **IMPORTANT**: For the port 8080 rule, the Source is the **Security Group ID** of `alb-sg`, NOT an IP address. This means only traffic originating from the ALB (which uses `alb-sg`) can reach your EC2s on 8080. This is the proper way to secure backend instances.

6. **Outbound Rules:** Leave default (All traffic)
7. Click **Create**

---

### Step 2: Launch Two EC2 Instances

#### EC2-1 (in subnet: `my-vpc-1-public-1`)

1. Go to **EC2 Console > Launch Instance**
2. **Name**: `app-server-1`
3. **AMI**: Amazon Linux 2023 (free tier eligible)
4. **Instance Type**: `t2.micro`
5. **Key Pair**: Select your existing key pair (or create one)
6. **Network Settings** - Click **Edit**:
   - VPC: `my-vpc-1`
   - Subnet: `my-vpc-1-public-1` (`subnet-0f27153bbe1b8f0d5`)
   - Auto-assign Public IP: **Enable**
   - Security Group: Select **existing** > `ec2-app-sg`
7. Click **Launch Instance**

#### EC2-2 (in subnet: `my-vpc-1-public-2`)

1. Same as above but:
   - **Name**: `app-server-2`
   - Subnet: `my-vpc-1-public-2` (`subnet-0e4c3c8ec084b7756`)
   - Everything else identical

---

### Step 3: Deploy the App on BOTH EC2s

SSH into **each** EC2 and run:

```bash
# Install Node.js
sudo yum install -y nodejs

# Create app directory
mkdir ~/todoapp && cd ~/todoapp

# Create the app file
cat << 'EOF' > app.js
const http = require('http');
const os = require('os');

const INSTANCE_IP = os.networkInterfaces()['eth0']?.[0]?.address || 'unknown';

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
     - `my-vpc-1-public-1` (`subnet-0f27153bbe1b8f0d5`)
     - `my-vpc-1-public-2` (`subnet-0e4c3c8ec084b7756`)
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
3. Copy the **DNS Name** (looks like `my-basic-alb-123456.us-east-1.elb.amazonaws.com`)
4. Go to **Target Groups > tg-basic-alb > Targets tab**
   - Wait until both targets show **Status: healthy**
   - If they show `unhealthy`, check:
     - Is the app running? (SSH in and run `curl localhost:8080/home`)
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

**To verify client IP preservation** (SSH into an EC2):
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
1. **Multi-AZ Setup**: Place subnets in different AZs (us-east-1a, us-east-1c) for high availability
2. **HTTPS/TLS**: Add certificates and HTTPS listeners
3. **Auto Scaling Groups**: Let AWS automatically add/remove EC2s based on traffic
4. **NLB + ALB Combo**: Use NLB for static IPs in front, ALB behind for smart routing
5. **Cross-Region**: Use Route 53 + multiple ALBs across regions for global traffic management
