# Part 1: YAML Fundamentals for CloudFormation

---

## Table of Contents

1. [What is YAML?](#1-what-is-yaml)
2. [YAML vs JSON — Which to Use for CloudFormation](#2-yaml-vs-json--which-to-use-for-cloudformation)
3. [Creating and Editing YAML Files](#3-creating-and-editing-yaml-files)
4. [Comments](#4-comments)
5. [Key-Value Pairs — The Building Block](#5-key-value-pairs--the-building-block)
6. [Data Types](#6-data-types)
7. [Lists](#7-lists)
8. [Mappings — Nested Key-Value Pairs](#8-mappings--nested-key-value-pairs)
9. [Lists of Mappings](#9-lists-of-mappings)
10. [Multiline Strings](#10-multiline-strings)
11. [Anchors and Aliases — Avoiding Repetition](#11-anchors-and-aliases--avoiding-repetition)
12. [Indentation Rules — The Most Common Source of Errors](#12-indentation-rules--the-most-common-source-of-errors)
13. [Common YAML Mistakes and How to Fix Them](#13-common-yaml-mistakes-and-how-to-fix-them)
14. [VS Code Setup for YAML](#14-vs-code-setup-for-yaml)
15. [YAML Cheat Sheet](#15-yaml-cheat-sheet)
16. [Practice Tasks](#16-practice-tasks)

---

## 1. What is YAML?

YAML stands for **YAML Ain't Markup Language** (a recursive acronym). It is a human-readable data serialization format — think of it as a cleaner, more readable alternative to JSON or XML. Rather than curly braces and square brackets, YAML uses indentation and simple punctuation to define structure.

AWS CloudFormation accepts templates in both YAML and JSON. YAML is almost always the better choice for writing and reading — it is less verbose, supports comments, and feels natural compared to deeply nested JSON.

```
JSON (hard to read at scale):            YAML (same data, cleaner):
──────────────────────────────           ──────────────────────────
{                                        Server:
  "Server": {                              Name: web-01
    "Name": "web-01",                      OS: Amazon Linux 2
    "OS": "Amazon Linux 2",                RAM: 8
    "RAM": 8,                              Production: true
    "Production": true
  }
}
```

### Where YAML is used in AWS

| Service | YAML files used for |
|:--------|:--------------------|
| CloudFormation | Infrastructure templates |
| CodeBuild | `buildspec.yml` — build instructions |
| CodePipeline | Pipeline definitions |
| ECS | Task definitions (also JSON) |
| GitHub Actions / CodePipeline | CI/CD workflow files |
| Kubernetes (EKS) | Deployments, services, config maps |
| SAM (Serverless Application Model) | Serverless app templates |

YAML is the universal language for "describing what infrastructure should exist." Master it once, use it everywhere.

---

## 2. YAML vs JSON — Which to Use for CloudFormation

| Feature | YAML | JSON |
|:--------|:-----|:-----|
| Comments | Yes (`#`) | No |
| Readability | High | Lower (lots of `{}`, `[]`, `"`) |
| Multiline strings | Clean (`|` and `>`) | Messy (`\n` escapes) |
| Verbosity | Concise | Verbose |
| CloudFormation support | Full | Full |
| Tooling support | Excellent | Excellent |
| Error messages | Slightly harder to debug | Line/column errors are precise |

> **Rule: Always write CloudFormation in YAML.** The only reason to use JSON is if you are generating templates programmatically from code that produces JSON natively. For human-authored templates, YAML wins.

One important caveat: YAML is a **superset of JSON**. Valid JSON is valid YAML. This means tools that parse YAML can also parse JSON — the reverse is not true.

---

## 3. Creating and Editing YAML Files

### File extension

Use `.yaml` or `.yml` — both are accepted everywhere. CloudFormation accepts either. The convention in this notes series is `.yaml`.

### Creating a YAML file

```
In VS Code:
  File → New File → Save As → template.yaml
  OR
  Ctrl+Shift+P → "YAML: New File"

In terminal:
  touch template.yaml
  code template.yaml
```

### The first line of any YAML file

YAML files can optionally begin with `---` (three dashes). This is the "document start" marker. CloudFormation templates do not require it, but it is a common convention.

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"
Description: My first CloudFormation template
```

### Encoding

Always save YAML files as **UTF-8** without BOM. VS Code does this by default. Never use tabs — only spaces.

---

## 4. Comments

Comments in YAML start with `#`. Everything after `#` on the same line is ignored.

```yaml
# This is a full-line comment — the parser ignores this entire line

Name: web-server  # This is an inline comment — ignored from # onward

# You can comment out a key by adding # before it:
# InstanceType: t3.large   ← this line is disabled / commented out
InstanceType: t3.micro      ← this is the active value
```

### VS Code tip: comment/uncomment blocks

Select multiple lines → `Ctrl+/` (Windows/Linux) or `Cmd+/` (Mac) to toggle comments for the entire selection.

### Why comments matter in CloudFormation

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro  # Free tier eligible
      # ImageId is the Amazon Linux 2 AMI for us-east-1
      # Update this if you are in a different region
      ImageId: ami-0c02fb55956c7d316
      # The key pair must already exist in your account
      KeyName: my-keypair
```

Comments save you and your teammates hours of confusion when revisiting templates months later.

---

## 5. Key-Value Pairs — The Building Block

Every piece of data in YAML is expressed as a **key-value pair** (also called a mapping entry).

```
key: value
```

The colon `:` separates the key from the value. **There must be at least one space after the colon.**

```yaml
# Correct
Name: web-server
Port: 80
Enabled: true

# WRONG — no space after colon
Name:web-server   ← YAML parser error

# WRONG — tab instead of space
Name:	web-server  ← YAML does not allow tabs
```

### A complete example

```yaml
Fruit:
  Name: mango
  Color: yellow
  Taste: sweet
  Origin: India
  Calories: 60
  InSeason: true
```

The key `Fruit` maps to a **nested mapping** (a group of key-value pairs). Each sub-key is indented by 2 spaces under `Fruit`. More on nesting in Section 8.

---

## 6. Data Types

YAML automatically detects data types based on the value. You rarely need to declare them explicitly.

### Strings

```yaml
# Unquoted string — most common, works when value has no special characters
Name: web-server
Region: us-east-1

# Double-quoted string — use when value contains special characters or colons
Description: "This is a: test server"
Message: "Hello, World!"

# Single-quoted string — takes the value literally, no escape sequences
Pattern: 'c:\users\pavan'
Literal: '${AWS::StackName}'  # prevents YAML from interpreting ${}
```

**When must you quote a string?**

| Situation | Example |
|:----------|:--------|
| Value contains a colon followed by a space | `"host: port"` |
| Value starts with `{`, `[`, `"`, `'` | `"[a, b, c]"` |
| Value looks like a number but should be a string | `"08080"` |
| Value looks like a boolean but should be a string | `"true"` or `"yes"` |
| Value contains `#` (would be treated as comment) | `"color: #ff0000"` |

### Numbers

```yaml
Port: 8080          # integer
Price: 9.99         # float
NegativeTemp: -273  # negative integer
Scientific: 1.5e10  # scientific notation
```

### Booleans

YAML recognizes several words as boolean. **This causes bugs.** Be aware:

```yaml
# These are ALL parsed as true:
Enabled: true
Enabled: True
Enabled: TRUE
Enabled: yes      ← dangerous — looks like a string but isn't!
Enabled: Yes
Enabled: on

# These are ALL parsed as false:
Enabled: false
Enabled: False
Enabled: no
Enabled: off
```

> **CloudFormation tip:** In CFT, boolean values should be written as `true` or `false` (lowercase). If a property expects the string `"true"` not the boolean `true`, quote it.

### Null

```yaml
OptionalValue: null    # explicit null
OptionalValue: ~       # tilde is shorthand for null
OptionalValue:         # empty value also means null
```

### Summary table

```
Type        Example             Notes
──────────  ──────────────────  ────────────────────────────────────────
String      Name: web-01        Unquoted unless special chars present
String      Tag: "yes"          Quoted to prevent boolean interpretation
Integer     Port: 443           Plain number
Float       Cost: 0.0058        Decimal
Boolean     Enabled: true       Only use true/false in CFT
Null        Value: null         Or ~, or empty
```

---

## 7. Lists

A list (also called a sequence) is an ordered collection of items. YAML has two ways to write lists.

### Block list (most readable — use this)

Each item starts with `- ` (dash + space), indented under the key:

```yaml
Fruits:
  - mango
  - apple
  - banana
  - grape

EC2InstanceTypes:
  - t3.micro
  - t3.small
  - t3.medium
  - m5.large
```

### Inline list (flow style)

All items on one line inside square brackets, separated by commas:

```yaml
Fruits: [mango, apple, banana, grape]

EC2InstanceTypes: [t3.micro, t3.small, t3.medium, m5.large]

AllowedPorts: [80, 443, 8080]
```

Both formats produce the exact same data structure. Block style is preferred for readability in CloudFormation templates.

### When you will see lists in CloudFormation

```yaml
# Security group inbound rules — a list of rule objects
SecurityGroupIngress:
  - IpProtocol: tcp
    FromPort: 80
    ToPort: 80
    CidrIp: 0.0.0.0/0
  - IpProtocol: tcp
    FromPort: 443
    ToPort: 443
    CidrIp: 0.0.0.0/0

# Subnet IDs for a load balancer — a list of strings
Subnets:
  - !Ref PublicSubnet1
  - !Ref PublicSubnet2

# Allowed values for a parameter
AllowedValues:
  - t3.micro
  - t3.small
  - t3.medium
```

---

## 8. Mappings — Nested Key-Value Pairs

A mapping is a key whose value is itself a set of key-value pairs. Nesting creates the hierarchical structure that CloudFormation uses to describe complex resources.

### Single-level nesting

```yaml
Database:
  Engine: MySQL
  Version: "8.0"
  Port: 3306
  MultiAZ: false
```

`Database` is the key. Its value is a mapping with 4 key-value pairs. Each sub-key is indented 2 spaces.

### Two-level nesting

```yaml
EC2Instance:
  InstanceType: t3.micro
  BlockDeviceMappings:
    DeviceName: /dev/xvda
    Ebs:
      VolumeSize: 20
      VolumeType: gp3
      DeleteOnTermination: true
```

### Deep nesting example — CloudFormation resource

```yaml
WebServerInstance:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: ami-0c02fb55956c7d316
    InstanceType: t3.micro
    NetworkInterfaces:
      - DeviceIndex: "0"
        SubnetId: !Ref PublicSubnet
        AssociatePublicIpAddress: true
        GroupSet:
          - !Ref WebServerSG
    Tags:
      - Key: Name
        Value: web-server
      - Key: Environment
        Value: production
```

### Visualizing nesting depth

```
WebServerInstance          ← depth 0 (top-level key)
  Type                     ← depth 1
  Properties               ← depth 1
    ImageId                ← depth 2
    InstanceType           ← depth 2
    NetworkInterfaces      ← depth 2
      - DeviceIndex        ← depth 3 (list item → mapping)
        SubnetId           ← depth 3
        GroupSet           ← depth 3
          - !Ref ...       ← depth 4 (list item)
    Tags                   ← depth 2
      - Key                ← depth 3
        Value              ← depth 3
```

Each level of nesting = 2 more spaces. Never mix 2-space and 4-space indentation in the same file.

---

## 9. Lists of Mappings

The most common pattern in CloudFormation is a **list of mappings** — a list where each item is itself a key-value mapping. This is used for tags, security group rules, route table entries, listener rules, and more.

```yaml
# Each tag is a mapping with Key and Value
Tags:
  - Key: Name
    Value: web-server
  - Key: Environment
    Value: production
  - Key: Team
    Value: platform

# Each rule is a mapping with IpProtocol, FromPort, ToPort, CidrIp
SecurityGroupIngress:
  - IpProtocol: tcp
    FromPort: 22
    ToPort: 22
    CidrIp: 10.0.0.0/8
  - IpProtocol: tcp
    FromPort: 80
    ToPort: 80
    CidrIp: 0.0.0.0/0
  - IpProtocol: tcp
    FromPort: 443
    ToPort: 443
    CidrIp: 0.0.0.0/0
```

### The dash `- ` starts a new list item

Every time you see `- ` at the same indentation level, it is a new item in the list. The mapping properties that follow are indented further under the dash.

```
SecurityGroupIngress:        ← key pointing to a list
  - IpProtocol: tcp          ← item 1: the dash starts the mapping
    FromPort: 22             ← still item 1: same indentation as IpProtocol
    ToPort: 22               ← still item 1
  - IpProtocol: tcp          ← item 2: new dash = new list item
    FromPort: 80             ← still item 2
    ToPort: 80               ← still item 2
```

---

## 10. Multiline Strings

Sometimes a value is too long for one line, or it contains newlines (like a shell script or user data for EC2). YAML provides two multiline string styles.

### Literal block scalar `|` — preserves newlines

Use when line breaks matter (shell scripts, SQL, config files):

```yaml
UserData:
  |
  #!/bin/bash
  yum update -y
  yum install -y httpd
  systemctl start httpd
  systemctl enable httpd
  echo "Hello from CloudFormation" > /var/www/html/index.html
```

The content is taken exactly as written, with newlines preserved. Each line of the script appears on its own line.

### Folded block scalar `>` — folds newlines into spaces

Use when you have a long sentence that you want to wrap across lines for readability, but it should actually be a single line:

```yaml
Description: >
  This template creates a highly available web application
  with an Application Load Balancer, Auto Scaling Group,
  and RDS database in a multi-AZ configuration.
```

The above produces a single string with spaces instead of newlines.

### In CloudFormation: UserData with Base64

EC2 `UserData` must be Base64-encoded. CloudFormation has an intrinsic function for this:

```yaml
UserData:
  Fn::Base64: |
    #!/bin/bash
    yum update -y
    amazon-linux-extras install -y nginx1
    systemctl start nginx
    systemctl enable nginx
```

The `|` preserves the script's newlines exactly as needed for a shell script.

---

## 11. Anchors and Aliases — Avoiding Repetition

YAML anchors (`&`) let you define a value once and reuse it elsewhere with aliases (`*`). This is useful when the same tags or config block repeats in your template.

### Define an anchor

```yaml
# Define an anchor named "common-tags"
CommonTags: &common-tags
  Project: my-app
  Environment: production
  Owner: platform-team
```

### Use an alias

```yaml
WebServer:
  Tags:
    <<: *common-tags        # merge in all keys from common-tags
    Name: web-server        # add an extra key on top

Database:
  Tags:
    <<: *common-tags        # same common tags
    Name: database          # different Name
```

`<<:` is the merge key — it expands the anchor's contents into the current mapping.

> **CloudFormation caveat:** CloudFormation's template validator does not always support YAML anchors. They are resolved by the YAML parser before the template is sent to CloudFormation, so it works in practice — but some linters will warn. Use sparingly and test before relying on them heavily.

---

## 12. Indentation Rules — The Most Common Source of Errors

Indentation is YAML's syntax. A wrong indent means a different data structure or an error. These rules are non-negotiable.

### Rules

```
1. Use SPACES only — never tabs.
2. Be consistent — pick 2 spaces per level and never deviate.
3. A child key is indented more than its parent.
4. Sibling keys are at the same indentation level.
5. List items (- ) are at the same level as their siblings.
```

### Correct indentation

```yaml
Resources:               # level 0
  MyBucket:              # level 1 (2 spaces)
    Type: AWS::S3::Bucket  # level 2 (4 spaces)
    Properties:          # level 2
      BucketName: my-bucket  # level 3 (6 spaces)
      Tags:              # level 3
        - Key: Name      # level 4 (8 spaces) — list item
          Value: my-bucket  # level 4 — part of same list item
```

### Wrong indentation — common errors

```yaml
# ERROR: child at same level as parent
Resources:
MyBucket:              ← wrong — should be indented 2 spaces
  Type: AWS::S3::Bucket

# ERROR: inconsistent indentation
Properties:
  BucketName: my-bucket
    Tags:              ← wrong — Tags is a child of Properties, not BucketName
      - Key: Name

# ERROR: tab character
Properties:
	BucketName: my-bucket  ← tab here causes a YAML parse error
```

### VS Code helper: show whitespace

Enable **View → Render Whitespace** in VS Code so you can see dots for spaces and arrows for tabs. This makes indentation errors immediately visible.

---

## 13. Common YAML Mistakes and How to Fix Them

### Mistake 1: Missing space after colon

```yaml
# Wrong
Name:web-server

# Right
Name: web-server
```

### Mistake 2: String that looks like a boolean

```yaml
# Wrong — "yes" and "no" are booleans in YAML 1.1
Environment: yes    ← parsed as true, not the string "yes"

# Right
Environment: "yes"
Environment: production  ← or just use a proper string
```

### Mistake 3: Unquoted special characters

```yaml
# Wrong — colon in value confuses the parser
Description: This is: a test

# Right
Description: "This is: a test"
```

### Mistake 4: Wrong indentation on list items

```yaml
# Wrong
Tags:
- Key: Name      ← dash at same level as Tags, not under it
  Value: test

# Right
Tags:
  - Key: Name    ← dash indented 2 spaces under Tags
    Value: test
```

### Mistake 5: Mixing tabs and spaces

VS Code shows a `Tab Size` indicator at the bottom right. Set it to **Spaces: 2** for YAML files. If you see `Tab Size` instead of `Spaces`, click it and select **Convert Indentation to Spaces**.

### Mistake 6: String that looks like a number

```yaml
# Wrong — leading zero: YAML parses this as octal!
Port: 0443    ← parsed as the number 291 (octal 0443 = decimal 291)

# Right
Port: "0443"  ← quoted, preserved as string
Port: 443     ← or just don't use a leading zero
```

---

## 14. VS Code Setup for YAML

### Extensions to install

| Extension | Publisher | Purpose |
|:----------|:----------|:--------|
| **YAML** | Red Hat | Syntax highlighting, validation, auto-complete for YAML |
| **AWS Toolkit** | Amazon Web Services | CloudFormation schema validation, deploy from IDE |
| **CloudFormation Linter (cfn-lint)** | AWS | Validates CFT templates against the official resource schema |

### Install cfn-lint (run in terminal)

```bash
pip install cfn-lint
# or
brew install cfn-lint
```

### VS Code settings for YAML

Add to your `.vscode/settings.json` or user settings:

```json
{
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "files.trimTrailingWhitespace": true,
  "[yaml]": {
    "editor.tabSize": 2,
    "editor.insertSpaces": true,
    "editor.formatOnSave": true
  },
  "yaml.schemas": {
    "https://raw.githubusercontent.com/aws-cloudformation/cloudformation-resource-schema/main/CloudFormationSchema.json": "*.yaml"
  }
}
```

### What you get with the Red Hat YAML extension

```
✓ Real-time validation as you type
✓ Red underlines on syntax errors
✓ Hover documentation for known keys
✓ Auto-complete for indented keys
✓ Bracket matching
✓ Folding of nested blocks
✓ Schema validation when schema is configured
```

---

## 15. YAML Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║                    YAML QUICK REFERENCE                          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  COMMENTS                                                        ║
║  # full line comment                                             ║
║  Key: value  # inline comment                                    ║
║                                                                  ║
║  KEY-VALUE PAIRS                                                 ║
║  Key: value               (space after colon is mandatory)       ║
║  Key: "special: value"    (quote when value has special chars)   ║
║                                                                  ║
║  DATA TYPES                                                      ║
║  String:  Name: hello     Number:  Port: 8080                    ║
║  Bool:    On: true        Null:    Val: null  OR  Val: ~         ║
║                                                                  ║
║  LISTS (two formats, same result)                                ║
║  Block:          Inline:                                         ║
║  Items:          Items: [a, b, c]                                ║
║    - a                                                           ║
║    - b                                                           ║
║    - c                                                           ║
║                                                                  ║
║  NESTED MAPPING                                                  ║
║  Parent:                                                         ║
║    Child1: value1                                                ║
║    Child2: value2                                                ║
║                                                                  ║
║  LIST OF MAPPINGS                                                ║
║  Tags:                                                           ║
║    - Key: Name                                                   ║
║      Value: web                                                  ║
║    - Key: Env                                                    ║
║      Value: prod                                                 ║
║                                                                  ║
║  MULTILINE STRINGS                                               ║
║  Literal (preserves newlines):   Folded (newlines → spaces):     ║
║  Script: |                       Desc: >                         ║
║    line 1                          long sentence                 ║
║    line 2                          wrapped here                  ║
║                                                                  ║
║  ANCHORS AND ALIASES                                             ║
║  Base: &anchor                                                   ║
║    Key: value                                                    ║
║  Copy:                                                           ║
║    <<: *anchor   (merges all keys from anchor)                   ║
║                                                                  ║
║  INDENTATION RULES                                               ║
║  - Spaces only, never tabs                                       ║
║  - 2 spaces per level (consistent throughout file)               ║
║  - Child is always more indented than parent                     ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 16. Practice Tasks

### Task 1 — Write a YAML file from scratch

Create a file called `fruits.yaml` that describes 3 fruits. Each fruit should have: `name`, `color`, `taste`, `origin`, `calories` (number), and `inSeason` (boolean). Represent the fruits as a list of mappings.

**Expected structure:**

```yaml
fruits:
  - name: mango
    color: yellow
    taste: sweet
    origin: India
    calories: 60
    inSeason: true
  - name: ...
    ...
  - name: ...
    ...
```

---

### Task 2 — Identify and fix YAML errors

The following YAML has 5 errors. Find and fix all of them.

```yaml
ServerConfig:
Name: web-server
  Port:8080
  OS: "Amazon Linux 2
  Enabled: yes
  Disks:
  - /dev/xvda
    - /dev/xvdf
```

**Errors to find:**
1. `Name` is at the wrong indentation level (should be under `ServerConfig`)
2. Missing space after `:` in `Port:8080`
3. Unclosed string quote in `"Amazon Linux 2` (missing closing `"`)
4. `yes` should be quoted `"yes"` if it means the string yes, not the boolean true
5. Second list item `- /dev/xvdf` has wrong indentation — should be at same level as `- /dev/xvda`

---

### Task 3 — Describe an AWS environment in YAML

Write a `environment.yaml` file that describes a simple AWS setup (no actual CloudFormation yet — just practice writing YAML). Include:

- A `vpc` block: `cidr`, `name`, `region`
- A `subnets` list with 2 items: each has `name`, `cidr`, `type` (public/private), `az`
- An `ec2` block: `name`, `type`, `ami`, `keyPair`, `tags` (list with Name and Environment)
- A `database` block: `engine`, `version`, `instanceClass`, `multiAZ`

Use proper indentation, add comments explaining each section, and quote any value that could be misinterpreted.

---

### Key Points to Remember

- YAML is indentation-based — a wrong indent is a wrong structure, not just bad formatting
- Spaces only, never tabs — this is enforced by the YAML specification
- Colon + space (`: `) separates keys from values — the space is mandatory
- `yes`, `no`, `on`, `off`, `true`, `false` are all booleans — quote them if you mean a string
- `|` preserves newlines (scripts), `>` folds newlines into spaces (long descriptions)
- Lists use `- ` (dash + space) — each item at the same indent level
- Comments (`#`) do not exist in JSON — they are a YAML advantage you should use generously
- The Red Hat YAML extension in VS Code + cfn-lint give you real-time error feedback

---

## Resources

- [YAML Official Specification](https://yaml.org/spec/1.2.2/)
- [YAML Tutorial — Learn X in Y Minutes](https://learnxinyminutes.com/docs/yaml/)
- [Red Hat YAML Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)
- [cfn-lint GitHub](https://github.com/aws-cloudformation/cfn-lint)
- [CloudFormation in Telugu (first 30min)](https://www.youtube.com/watch?v=XlOxrLt3QL8&list=PLneBjIzDLECkYQ7dYpvWFSgEwTMeZkykF&index=54)
