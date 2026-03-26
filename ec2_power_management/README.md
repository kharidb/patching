# EC2 Power Management – Ansible Automation Platform Template

Start or stop one or more AWS EC2 instances by Instance ID, driven by
an AAP Job Template Survey.

---

## Files

| File | Purpose |
|---|---|
| `ec2_power_management.yml` | Main Ansible playbook |
| `aap_survey.yml` | AAP Survey definition (import via UI or `awx` CLI) |
| `requirements.yml` | Galaxy collection dependencies |

---

## Prerequisites

### 1. AWS Credentials in AAP
Create a **Credential** of type **Amazon Web Services** in AAP
(Automation Controller → Credentials → Add) and attach it to the Job Template.

Required IAM permissions for the service account:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Execution Environment
The playbook requires the `amazon.aws` collection (≥ 7.0.0).

Option A – Install into an existing EE:
```bash
ansible-galaxy collection install -r requirements.yml
```

Option B – Build a custom EE using `ansible-builder` and reference
`requirements.yml` inside your `execution-environment.yml`.

---

## AAP Job Template Setup

| Field | Value |
|---|---|
| **Name** | EC2 Power Management |
| **Job Type** | Run |
| **Inventory** | *Any inventory with localhost* (or use the built-in localhost) |
| **Project** | *Your SCM project containing this playbook* |
| **Playbook** | `ec2_power_management.yml` |
| **Credentials** | Your AWS credential |
| **Survey** | Enable → import `aap_survey.yml` |
| **Extra vars** | *(leave empty – survey populates all vars)* |

---

## Survey Prompts (at launch)

| Prompt | Variable | Type | Default |
|---|---|---|---|
| AWS Region | `aws_region` | Choice | `us-east-1` |
| EC2 Instance IDs | `ec2_instance_ids` | Text | *(required)* |
| Action | `ec2_action` | Choice | `stop` |
| Wait for target state? | `wait_for_state` | Choice | `true` |
| Wait timeout (seconds) | `wait_timeout` | Integer | `300` |

---

## Running Without AAP (CLI)

```bash
# Stop an instance
ansible-playbook ec2_power_management.yml \
  -e "ec2_instance_ids=i-0abc123def456789a" \
  -e "ec2_action=stop" \
  -e "aws_region=us-east-1"

# Start multiple instances
ansible-playbook ec2_power_management.yml \
  -e "ec2_instance_ids=i-0abc123def456789a,i-0xyz987cba654321b" \
  -e "ec2_action=start" \
  -e "aws_region=us-west-2"
```

---

## Playbook Flow

```
pre_tasks
  ├── Assert instance IDs provided
  ├── Assert action is 'start' or 'stop'
  ├── Parse & validate instance ID list
  └── Display run parameters

tasks
  ├── Gather current instance states (ec2_instance_info)
  ├── Display current states
  ├── [if stop] Stop instances → wait for 'stopped'
  └── [if start] Start instances → wait for 'running'

post_tasks
  ├── Re-gather instance states
  ├── Display final states
  └── Assert all instances reached expected state
```

---

## Notes

* **Multiple instances** – pass a comma-separated list:
  `i-0abc123def456789a, i-0xyz987cba654321b`
* **Waiting** – set `wait_for_state=false` for fire-and-forget launches.
* **Idempotency** – already-running instances will be ignored when
  `ec2_action=start`, and already-stopped instances when `ec2_action=stop`.
