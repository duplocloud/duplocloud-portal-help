---
name: gen-help-yaml
description: This skill should be used when the user asks to "generate help yaml", "add tooltip for a form", "create yaml for a portal form", "add help content for a new page", or mentions generating a .yml file for duplocloud-portal-help. It looks up the actual Angular form field names from the duplo-ui GitHub repo and creates the YAML file, then regenerates en.json.
---

# Generate Help YAML for DuploCloud Portal

## Purpose

Generate tooltip/help YAML files for the `duplocloud-portal-help` repo by:
1. Looking up the real Angular form field `name` attributes from the `duplo-ui` source
2. Writing the YAML file(s) with correct field IDs and meaningful tooltip text
3. Optionally running `python3 consolidate.py` to regenerate `en.json` (user is asked)

## Repo and Conventions

- **Help repo working directory**: `/duplocloud-portal-help`
- **duplo-ui source**: `https://github.com/duplocloud-internal/duplo-ui` (private, use `gh api`)
- **Reference commit**: use `99c36c488a87e0ce2699522eb60eae8e89a4cb5a` (the `main` ref returns 404 on this repo — always use this commit hash unless the user specifies another)
- **YAML file naming** — resolve in this order:
  1. `<form name="...">` attribute on the `<form>` element
  2. `name="..."` on `<sidebar-modal>` or `<app-sidebar-generic-form>` if form name is absent
  3. URL path segment or page `<h5>` heading as last resort
- **YAML format**: one line per field — `fieldName: tooltip text` (see [AddManagedSql.yml](../../../../../development/duplocloud-portal-help/forms/AddManagedSql.yml))

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  /duplo:gen-help-yaml                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │  Step 0 — Ask user            │
          │  "Portal URL or form name?"   │
          └───────────────┬───────────────┘
                          │ user provides input
                          ▼
          ┌───────────────────────────────┐
          │  Step 1 — Check gh auth       │
          │  gh auth status               │
          │  not authed → gh auth login   │
          │  then stop, ask user to retry │
          └───────────────┬───────────────┘
                          │ authenticated
                          ▼
          ┌───────────────────────────────┐
          │  Step 2 — Locate HTML         │
          │  Derive path from URL segment │
          │  Browse via gh api (commit    │
          │  99c36c4...)                  │
          │  Or tree search by keyword    │
          └───────────────┬───────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │  Step 3 — Extract field IDs   │
          │  · template → name="..."      │
          │  · reactive → formControlName │
          │  YAML filename priority:      │
          │  1. <form name="...">         │
          │  2. <sidebar-modal name="...">│
          │  3. URL / page heading        │
          └───────────────┬───────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │  Step 4 — Write YAML          │
          │  Does forms/<name>.yml exist? │
          └───────────┬───────────┬───────┘
                      │           │
                   yes│           │no
                      ▼           ▼
            ┌──────────────┐  ┌──────────────────────┐
            │ For each     │  │ Create new file with  │
            │ extracted    │  │ all extracted fields  │
            │ field:       │  └──────────┬────────────┘
            │              │             │
            │ · missing?   │             │
            │   → append   │             │
            │              │             │
            │ · exists with│             │
            │   content?   │             │
            │   → ask user │             │
            │   "Update    │             │
            │    field X?" │             │
            │   y→ update  │             │
            │   n→ skip    │             │
            └──────┬───────┘             │
                   └──────────┬──────────┘
                              ▼
          ┌───────────────────────────────┐
          │  Patterns:                    │
          │  · Simple     → one line      │
          │  · Multi-opt  → bullets       │
          │  · YAML/JSON  → code block    │
          │  · Pattern    → backtick      │
          │  · Docs       → help link     │
          └───────────────┬───────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │  Step 5 — Ask user            │
          │  "Run consolidate.py now?"    │
          └───────────┬───────────┬───────┘
                      │           │
                     y│           │n
                      ▼           ▼
            ┌──────────────┐  ┌──────────────────────┐
            │ Run          │  │ "Run python3          │
            │ consolidate  │  │  consolidate.py later"│
            │ .py + verify │  └──────────┬────────────┘
            └──────┬───────┘             │
                   └──────────┬──────────┘
                              ▼
          ┌───────────────────────────────┐
          │  Step 6 — Report              │
          │  · File(s) created            │
          │  · en.json status             │
          └───────────────────────────────┘
```

## Step-by-Step Workflow

### Step 0 — Ask the user

Ask the user:

> "Which form do you want to generate help YAML for? Please provide either:
> - A **portal URL** (e.g. `https://app.duplocloud.net/...`) — the path will be used to locate the component, or
> - A **form name** (e.g. `AddS3TableBucket`) — the Angular form `name` attribute"

Wait for the user's response before proceeding.

### Step 1 — Check gh authentication

Run:
```bash
gh auth status
```

- If **authenticated** → proceed to Step 2.
- If **not authenticated** → tell the user:
  > "`gh` is not logged in. Please authenticate by running the following in your terminal:"
  > ```bash
  > gh auth login
  > ```
  > "Select GitHub.com → HTTPS → authenticate with browser. Once done, re-run `/duplo:gen-help-yaml`."
  
  Then stop and wait for the user to re-invoke the skill.

### Step 2 — Locate the component HTML in duplo-ui

Use `gh api` to browse the repo tree and find the relevant component HTML file.

Derive the repo path from the portal URL:
- `aws/storage/s3-tables` → `portal/src/app/main/devops/aws/aws-storage/s3-table-bucket/`
- `aws/glue/databases` → `portal/src/app/main/devops/aws/aws-glue/glue-database/`
- `k8s/.../hosts/asg` → `ui/angular/src/app/deploy/hosts/hosts-aws/hosts-aws-add-host-asg/`

Start by listing the top-level area and drill down:
```bash
gh api "repos/duplocloud-internal/duplo-ui/contents/portal/src/app/main/devops/aws?ref=99c36c488a87e0ce2699522eb60eae8e89a4cb5a" --jq '.[].name'
```

For a broader search across the full tree:
```bash
gh api "repos/duplocloud-internal/duplo-ui/git/trees/99c36c488a87e0ce2699522eb60eae8e89a4cb5a?recursive=1" --jq '.tree[].path' | grep -i "<keyword>"
```

Navigate into subfolders until the correct `*.component.html` file is found.

To read a file:
```bash
gh api "repos/duplocloud-internal/duplo-ui/contents/<path>?ref=99c36c488a87e0ce2699522eb60eae8e89a4cb5a" --jq '.content' | base64 -d
```

### Step 3 — Extract field IDs

From the HTML, extract field IDs using either pattern depending on the form type:

- **Template-driven forms** (most common): use `name="..."` on `<input>`, `<ng-select>`, `<textarea>` elements. The `<form name="...">` attribute is the YAML filename.
- **Reactive forms** (`[formGroup]` on the `<form>` element): use `formControlName="..."` on each input element. The YAML filename comes from the portal URL path or the page heading (e.g. URL ends in `/asg/add/ASG` → check component `<h5>` title or use the Angular form class name from the `.component.ts` file).
- **Generic sidebar forms** (`<app-sidebar-generic-form>`): when no `<form name="...">` is present, use the `name` attribute on the `<sidebar-modal>` or `<app-sidebar-generic-form>` element as the YAML filename. Example:
  ```html
  <app-sidebar-generic-form name="AddGlueTable" ...>
  ```
  → YAML filename: `AddGlueTable.yml`

When in doubt, check the `.component.ts` file for the `FormGroup` definition to confirm all field names.

### Step 4 — Write the YAML file

Before writing, check if `forms/<FormName>.yml` already exists:

```bash
ls forms/<FormName>.yml
```

- **File exists** → for each extracted field, check if it already exists in the file:
  - **Field is missing** → append it automatically.
  - **Field exists with content** → ask the user:
    > "`<fieldName>` already has content. Would you like to update it? (y/n)"
    - **y** → replace only that field's value.
    - **n** → skip it, leave existing content unchanged.
  - Never overwrite or remove the full file.

- **File does not exist** → create it with all extracted fields.

Create or append to `forms/<FormName>.yml` in `/duplocloud-portal-help/` with one entry per field.

#### Tooltip style guide

**Simple fields** — one line, verb-first:
```yaml
Name: Specify a unique name. Use only lowercase letters, numbers, and hyphens.
status: Select whether the feature is enabled or disabled.
```

**Fields with multiple options or behaviors** — use a `|` block with numbered bullets:
```yaml
targetType: |
  Specifies how to route traffic to pods.

  1. **instance** — Routes traffic to all EC2 instances on NodePort. Service must be NodePort or LoadBalancer.
  2. **ip** — Routes directly to the pod IP. Required for sticky sessions with ALBs.
```

**Fields that accept YAML or JSON** — use a `|` block with a fenced code block:
```yaml
envVariables: |
  Environment variables in YAML format.
  ```yaml
  - Name: VARIABLE1
    Value: value1
  ```
```

**Fields with cron/regex/pattern syntax** — inline backtick highlight the example:
```yaml
Schedule: Specify schedule in Cron format. Example- `0 0 * * 0`. Runs once a week at midnight on Sunday. For help on cron expressions, click [here](https://crontab.guru/).
```

**Fields that link to AWS/Azure/GCP/Kubernetes docs** — include platform-specific links when the field is annotation-, label-, or policy-driven:
```yaml
ingressAnnotations: |
  Key-value pairs to annotate the Ingress Object. Each ingress controller defines supported annotations.
  Refer these links:
    1. **AWS ALB ingress controller** [here](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/)
    2. **Azure Application Gateway Ingress** [here](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#list-of-supported-annotations)
    3. **GCP Ingress** [here](https://cloud.google.com/kubernetes-engine/docs/how-to/load-balance-ingress#ingress_annotations)
```

**Help links** — add a `[Details](url)` or `click [here](url)` reference whenever the field maps to official Kubernetes, AWS, Azure, or GCP documentation.

**General rules:**
- Start with a verb: "Select", "Enter", "Specify", "Enable"
- Mention constraints (min/max, length, pattern) inline
- Use `|` block whenever the content is multi-line
- Match tone/style of existing files: `AddManagedSql.yml`, `AddIngress.yml`, `K8sJobBasicForm.yml`

### Step 5 — Ask user to regenerate en.json

After writing the YAML file(s), ask the user:

> "YAML file created. Would you like to run `python3 consolidate.py` now to incorporate the changes into `en.json`? (y/n)"

- **y** → run the following from the help repo root:
  ```bash
  cd /duplocloud-portal-help && python3 consolidate.py
  ```
  Then verify the new form key appears in `en.json`:
  ```bash
  python3 -c "import json; d=json.load(open('en.json')); print(list(d['data']['FORM'].keys()))" | grep <FormName>
  ```
- **n** → skip, tell the user "You can run `python3 consolidate.py` later to incorporate the changes into `en.json`.", and proceed to Step 6.

### Step 6 — Report

Confirm which YAML file(s) were created and whether `en.json` was updated.

## Notes

- For dynamic field names like `destinationArn_0`, `destinationArn_1` (indexed arrays), use the base name without the index: `destinationArn`.
- If `pyyaml` is not installed, run: `pip3 install pyyaml --break-system-packages`
- The `gh` CLI must be authenticated: `gh auth status`
