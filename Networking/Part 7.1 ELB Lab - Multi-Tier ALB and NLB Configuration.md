# Part 7.1: ELB Lab — Multi-Tier ALB + NLB Configuration

---

## Lab Overview

This is a real lab task where you configure **both an ALB and NLB** in a multi-tier architecture. Everything is pre-created (EC2 instances, load balancers, target groups, security groups) — you just need to wire them together.

### Architecture

```
                        INTERNET
                           │
                    ┌──────┴──────┐
                    │     ALB     │  Internet-facing, Port 80
                    │  (Layer 7)  │  Path-based routing
                    └──┬───────┬──┘
                       │       │
              /customers      /orders
                       │       │
           ┌───────────┘       └───────────┐
           │                               │
  ┌────────┴────────┐           ┌──────────┴──────────┐
  │ Customers Server │           │   Orders Server     │
  │ Port 8080        │           │   Port 8080         │
  └────────┬─────────┘           └──────────┬──────────┘
           │                                │
           └──────────┐   ┌────────────────┘
                      │   │
               ┌──────┴───┴──────┐
               │      NLB        │  Internal, Layer 4
               │  TCP:3000       │  No path awareness
               │  UDP:7788       │  Raw packet forwarding
               └──┬──────────┬───┘
                  │          │
         TCP:3000 │          │ UDP:7788
                  │          │
     ┌────────────┘          └────────────┐
     │                                    │
┌────┴────────────┐          ┌────────────┴─────┐
│  TCP Backend    │          │  UDP Backend      │
│  Service        │          │  Service          │
│  Port 3000      │          │  Port 7788        │
└─────────────────┘          └──────────────────┘
```

### What's Pre-Created (Don't Touch These)

| Resource | Role |
|----------|------|
| **Customers Web Server** | Handles `/customers` requests on HTTP port 8080 |
| **Orders Web Server** | Handles `/orders` requests on HTTP port 8080 |
| **TCP Backend Service** | Handles traffic on TCP port 3000 |
| **UDP Backend Service** | Handles traffic on UDP port 7788 |
| **Application Load Balancer** | Internet-facing ALB (already exists, no listener yet) |
| **Network Load Balancer** | Internal NLB (already exists, no listeners yet) |
| **Customers Target Group** | ALB target group for customers path |
| **Orders Target Group** | ALB target group for orders path |
| **TCP Target Group** | NLB target group for TCP traffic |
| **UDP Target Group** | NLB target group for UDP traffic |
| **4 Security Groups** | One per EC2 instance |

> All instances have **port 5001** open for health checks. Region: `eu-west-1`.

---

## What YOU Need To Do

Three things:
1. **Register targets** — Attach EC2 instances to their target groups
2. **Create NLB listeners** — TCP on 3000, UDP on 7788
3. **Configure ALB listener** — Port 80 with path-based routing + redirect default

---
---

## Step 1: Register EC2 Instances to Target Groups

You have 4 instances and 4 target groups. Each instance goes into exactly one target group.

### 1A: Register Customers Server to Customers Target Group

1. Go to **EC2 Console > Target Groups**
2. Find and click on the **Customers target group** (`cmtr-...-tg-cust`)
3. Go to the **Targets** tab
4. Click **Register targets**
5. In the instance list, find the **Customers Web Server** instance
6. Check its checkbox
7. **Port**: `8080`
   > The customers server listens on 8080, so the target group must forward to 8080
8. Click **Include as pending below**
9. Click **Register pending targets**

### 1B: Register Orders Server to Orders Target Group

1. Go back to **Target Groups**
2. Click the **Orders target group** (`cmtr-...-tg-orders`)
3. **Targets tab > Register targets**
4. Select the **Orders Web Server** instance
5. **Port**: `8080`
6. Include as pending > Register

### 1C: Register TCP Backend to TCP Target Group

1. Click the **TCP target group** (`cmtr-...-tg-tcp`)
2. **Targets tab > Register targets**
3. Select the **TCP Backend Service** instance
4. **Port**: `3000`
   > This backend runs on port 3000, NOT 8080
5. Include as pending > Register

### 1D: Register UDP Backend to UDP Target Group

1. Click the **UDP target group** (`cmtr-...-tg-udp`)
2. **Targets tab > Register targets**
3. Select the **UDP Backend Service** instance
4. **Port**: `7788`
   > UDP backend uses port 7788
5. Include as pending > Register

---

### Verify Step 1

Go to each target group's **Targets** tab and confirm:

| Target Group | Instance | Port | Expected Status |
|-------------|----------|------|-----------------|
| Customers TG | Customers Web Server | 8080 | Healthy (after health checks pass) |
| Orders TG | Orders Web Server | 8080 | Healthy |
| TCP TG | TCP Backend Service | 3000 | Healthy |
| UDP TG | UDP Backend Service | 7788 | Healthy |

> Health checks use **port 5001** on all instances. Give it 30-60 seconds for targets to become healthy.

---
---

## Step 2: Create NLB Listeners

The Network Load Balancer exists but has **no listeners**. You need to create two: one for TCP and one for UDP.

### 2A: Create TCP Listener (Port 3000)

1. Go to **EC2 Console > Load Balancers**
2. Find and select the **Network Load Balancer** (`cmtr-...-nlb`)
3. Go to the **Listeners** tab
4. Click **Add listener**
5. Configure:

```
Listener Configuration:
────────────────────────────────────────────
Protocol:        TCP
Port:            3000
Default action:  Forward to → TCP Target Group (cmtr-...-tg-tcp)
────────────────────────────────────────────
```

6. Click **Add**

### 2B: Create UDP Listener (Port 7788)

1. Still on the NLB's **Listeners** tab
2. Click **Add listener** again
3. Configure:

```
Listener Configuration:
────────────────────────────────────────────
Protocol:        UDP
Port:            7788
Default action:  Forward to → UDP Target Group (cmtr-...-tg-udp)
────────────────────────────────────────────
```

4. Click **Add**

---

### Verify Step 2

Your NLB should now show 2 listeners:

| Protocol | Port | Target Group |
|----------|------|-------------|
| TCP | 3000 | TCP TG → TCP Backend (port 3000) |
| UDP | 7788 | UDP TG → UDP Backend (port 7788) |

> **Why NLB for these?** TCP and UDP are Layer 4 protocols. ALB only understands HTTP/HTTPS (Layer 7). For raw TCP/UDP traffic, you MUST use an NLB. The NLB doesn't inspect packet contents — it just forwards them to the right target group based on the port.

---
---

## Step 3: Configure ALB Listener (Path-Based Routing)

The ALB already exists and has a listener on port 80. You need to configure its routing rules.

### 3A: Find the ALB Listener

1. Go to **EC2 Console > Load Balancers**
2. Find and select the **Application Load Balancer** (`cmtr-...-lb`)
3. Go to the **Listeners** tab
4. You should see an **HTTP:80** listener already there
5. Click on the listener to open its rules

### 3B: Set the Default Action (Redirect to /orders)

The default action catches any request that doesn't match a specific rule. We want it to redirect to `/orders` with a 302.

1. In the listener rules view, find the **Default rule** (it's the last one, can't be deleted)
2. Click **Edit rule** (or select it and click **Actions > Edit rule**)
3. Change the action to:

```
Default Rule Action:
────────────────────────────────────────────
Action type:     Redirect to
Protocol:        HTTP
Port:            80
Host:            #{host}
Path:            /orders
Query:           #{query}
Status code:     HTTP_302
────────────────────────────────────────────
```

> **What this does**: Any request that doesn't match `/customers` or `/orders` gets a 302 redirect to `/orders`. So hitting `/` or `/random` or `/anything` sends the user to `/orders`.

4. Click **Save changes**

### 3C: Add Rule for /customers Path

1. In the listener rules view, click **Add rule** (or **Insert Rule**)
2. **Name/Tag**: `customers-rule`
3. **Priority**: `1` (evaluated first)
4. **Add condition**:

```
Condition:
────────────────────────────────────────────
Condition type:   Path
Path pattern:     /customers
────────────────────────────────────────────
```

> Use exact path `/customers` without wildcard — the lab validator expects exact match

5. **Add action**:

```
Action:
────────────────────────────────────────────
Action type:      Forward to target group
Target group:     Customers Target Group (cmtr-...-tg-cust)
────────────────────────────────────────────
```

6. Click **Create rule**

### 3D: Add Rule for /orders Path

1. Click **Add rule** again
2. **Name/Tag**: `orders-rule`
3. **Priority**: `2`
4. **Add condition**:

```
Condition:
────────────────────────────────────────────
Condition type:   Path
Path pattern:     /orders
────────────────────────────────────────────
```

5. **Add action**:

```
Action:
────────────────────────────────────────────
Action type:      Forward to target group
Target group:     Orders Target Group (cmtr-...-tg-orders)
────────────────────────────────────────────
```

6. Click **Create rule**

---

### Verify Step 3

Your ALB listener rules should look like this (in priority order):

```
ALB HTTP:80 Listener Rules
══════════════════════════════════════════════════════════════════
 Priority │ Condition          │ Action
──────────┼────────────────────┼─────────────────────────────────
    1     │ Path = /customers  │ Forward → Customers TG (8080)
    2     │ Path = /orders     │ Forward → Orders TG (8080)
  Default │ (no match)         │ Redirect 302 → /orders
══════════════════════════════════════════════════════════════════
```

---
---

## Step 4: Verification

### Test the ALB (do this in browser or curl)

ALB DNS: `cmtr-58ir3aht-ec2-es-lb-1228625124.eu-west-1.elb.amazonaws.com`

```bash
# Should return Customer service response
curl http://cmtr-58ir3aht-ec2-es-lb-1228625124.eu-west-1.elb.amazonaws.com/customers

# Should return Orders service response
curl http://cmtr-58ir3aht-ec2-es-lb-1228625124.eu-west-1.elb.amazonaws.com/orders

# Should 302 redirect to /orders, then return Orders response
curl -L http://cmtr-58ir3aht-ec2-es-lb-1228625124.eu-west-1.elb.amazonaws.com/

# See the redirect header directly
curl -I http://cmtr-58ir3aht-ec2-es-lb-1228625124.eu-west-1.elb.amazonaws.com/
# Look for: Location: http://cmtr-..../orders
```

### Expected Results

| URL | Expected Behavior |
|-----|------------------|
| `/customers` | Customer service response (from Customers Web Server, port 8080) |
| `/orders` | Orders service response (from Orders Web Server, port 8080) |
| `/` | 302 redirect → `/orders` → Orders service response |
| `/anything-else` | 302 redirect → `/orders` → Orders service response |

### Test the NLB (internal — can't hit from internet)

The NLB is **internal**, so you'd test it from within the VPC (e.g., from one of the EC2 instances):

```bash
# From inside the VPC (SSH/SSM into one of the web servers)
curl http://<NLB-DNS>:3000    # Should reach TCP backend
# UDP testing requires a UDP client, not curl
```

---
---

## How the Full Flow Works

```
USER BROWSER
     │
     │  GET /customers
     ▼
┌─────────────────────────────────────────────────────────┐
│  ALB (Internet-facing, Port 80)                         │
│                                                         │
│  Evaluates rules:                                       │
│    Rule 1: /customers* matches? YES → forward           │
│    Rule 2: /orders* matches? (not checked)              │
│    Default: redirect (not reached)                      │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │  Customers TG        │
                  │  → Customers Server  │
                  │  → Port 8080         │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │  Customers Server    │
                  │  processes request   │
                  │  returns response    │
                  └──────────────────────┘
```

For the NLB (internal, between tiers):

```
CUSTOMERS SERVER (inside VPC)
     │
     │  TCP connection to NLB:3000
     ▼
┌─────────────────────────────────────────┐
│  NLB (Internal, Layer 4)                │
│                                         │
│  Listener TCP:3000 → forward to TCP TG  │
│  (no path inspection, just port match)  │
└────────────────────┬────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  TCP Backend Service │
          │  Port 3000           │
          └──────────────────────┘
```

---

## Key Takeaways from This Lab

1. **ALB does path routing** — it reads the HTTP request path and sends traffic to different target groups based on rules
2. **NLB does port routing** — it just maps a port to a target group, no content inspection
3. **ALB can redirect** — the default rule sends unmatched paths to `/orders` with a 302
4. **NLB handles TCP + UDP** — ALB can only do HTTP/HTTPS
5. **Internal vs Internet-facing** — ALB faces the internet (users hit it), NLB is internal (servers talk to each other through it)
6. **Health checks on port 5001** — All instances expose a health endpoint on a dedicated port separate from the app port
7. **Target groups are the glue** — They connect load balancers to instances with a specific port

---

## ALB vs NLB Side-by-Side (This Lab)

| | ALB | NLB |
|--|-----|-----|
| **Scheme** | Internet-facing | Internal |
| **Layer** | 7 (HTTP) | 4 (TCP/UDP) |
| **Listener Port** | 80 | 3000 (TCP), 7788 (UDP) |
| **Routing Logic** | Path rules: /customers, /orders | Port-based only |
| **Default Action** | 302 redirect to /orders | N/A (just forwards) |
| **Target Ports** | 8080, 8080 | 3000, 7788 |
| **Who calls it?** | External users/browsers | Internal servers within VPC |
| **Health Check Port** | 5001 | 5001 |
