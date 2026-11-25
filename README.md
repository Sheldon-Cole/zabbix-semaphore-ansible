# Semaphore UI by HTTP (`zbx_template_ansible_semaphore.yaml`)

Agentless monitoring of **Ansible Semaphore UI** via its REST API for **Zabbix 7.0+**. Dynamically discovers project templates via Low-Level Discovery (LLD), tracks last task execution status, task ID, start/end timestamps, playbook name, and runner app type, and alerts on API outages, template fetch failures, task failures, and hung running tasks.

---

## Requirements

- **Zabbix** version `7.0` or higher
- **Semaphore UI** instance reachable via HTTP/HTTPS from the Zabbix Server or Zabbix Proxy
- Semaphore API Bearer token configured in `{$SEMAPHORE.API.TOKEN}` (`SECRET_TEXT`)

---

## Create a Semaphore API Token

1. Log in to the Semaphore web UI with an account that has read access to the target project.
2. Navigate to **User → API Tokens → Create Token** (or generate one via CLI/API).
3. Copy the token and store it in the Zabbix host macro `{$SEMAPHORE.API.TOKEN}` as **Secret text**.

---

## Import & Host Setup

1. **Validate Locally**:
   ```bash
   ./scripts/validate-templates.py
   ```
2. **Import into Zabbix**:
   - Navigate to **Data collection → Templates → Import** and select `7.0/zbx_template_ansible_semaphore.yaml`.
3. **Create or Link a Host**:
   - Navigate to **Data collection → Hosts → Create host**:
     - **Host name:** e.g. `semaphore` (or `semaphore-project-1` when monitoring multiple projects)
     - **Host groups:** e.g. `Applications`
     - **Templates:** Link **Semaphore UI by HTTP**
     - **Interfaces:** None required (HTTP Agent items run agentlessly)
4. **Configure Host Macros**:
   - Set `{$SEMAPHORE.URL}`, `{$SEMAPHORE.API.TOKEN}` (`Secret text`), and `{$SEMAPHORE.PROJECT.ID}` on the host.

---

## Macros

### Required Host Macros

| Macro | Default in Template | Type | Description |
| :--- | :--- | :---: | :--- |
| `{$SEMAPHORE.API.TOKEN}` | *(blank)* | `SECRET_TEXT` | Semaphore API Bearer token used for REST authentication |
| `{$SEMAPHORE.URL}` | `http://localhost:3000` | `TEXT` | Base URL of the Semaphore UI API without trailing slash (e.g., `https://semaphore.example.com`) |
| `{$SEMAPHORE.PROJECT.ID}` | `1` | `TEXT` | Numeric project ID (visible in Semaphore URL `/project/<id>/`) |

### Optional / Threshold Macros

| Macro | Default | Type | Description |
| :--- | :--- | :---: | :--- |
| `{$SEMAPHORE.HTTP.TIMEOUT}` | `15s` | `TEXT` | HTTP agent timeout for Semaphore API endpoints |
| `{$SEMAPHORE.TRIGGER.ENABLED}` | `1` | `TEXT` | Enable task failure and long-running task alerts globally (`1` = enabled, `0` = disabled) |
| `{$SEMAPHORE.TRIGGER.ENABLED:"{#ID}"}` | *(inherits)* | `TEXT` | Per-template context override (e.g., set `{$SEMAPHORE.TRIGGER.ENABLED:"3"}=0` to silence template ID `3`) |
| `{$SEMAPHORE.TASK.RUNNING.MAX.TIME}` | `1h` | `TEXT` | Duration with unit (`1h`, `30m`) before a continuously running task alerts |
| `{$SEMAPHORE.TASK.RUNNING.MAX.TIME:"{#ID}"}` | *(inherits)* | `TEXT` | Per-template context override for legitimately long-running playbooks |
| `{$SEMAPHORE.LLD.FILTER.NAME.MATCHES}` | `.*` | `TEXT` | Regex of template names to include in LLD |
| `{$SEMAPHORE.LLD.FILTER.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | `TEXT` | Regex of template names to exclude from LLD |

---

## Collected Metrics & Discovery

### Master HTTP Agent Items

| Key | Endpoint | Interval | History | Description |
| :--- | :--- | :---: | :---: | :--- |
| `semaphore.ping` | `GET /api/ping` | `1m` | `7d` | Health check (`1` = Up on `"pong"`, `0` = Down) |
| `semaphore.info.raw` | `GET /api/info` | `1h` | `0` | Raw JSON payload for Semaphore & Ansible versions |
| `semaphore.raw` | `GET /api/project/{$SEMAPHORE.PROJECT.ID}/templates` | `5m` | `0` | Raw JSON payload for template LLD and dependent task items |

### Static Dependent Items (from `semaphore.info.raw`)

| Key | Value Type | Heartbeat | Description |
| :--- | :---: | :---: | :--- |
| `semaphore.version` | `CHAR` | `1d` | Semaphore UI software version (`$.version`) |
| `semaphore.ansible.version` | `CHAR` | `1d` | Installed Ansible version (`$.ansible`) |

### Discovery Rule (`semaphore.template.discovery`)

Dependent on `semaphore.raw`. Strips volatile `last_task` fields before applying a `1h` `DISCARD_UNCHANGED_HEARTBEAT` so LLD only updates when templates are added, renamed, or removed.

| LLD Macro | JSONPath | Description |
| :--- | :--- | :--- |
| `{#ID}` | `$.id` | Semaphore template ID |
| `{#NAME}` | `$.name` | Semaphore template display name |
| `{#APP}` | `$.app` | Runner application type (`ansible`, `terraform`, `bash`, etc.) |

### Item Prototypes (dependent on `semaphore.raw`)

| Key | Type / Units | Heartbeat | Description |
| :--- | :---: | :---: | :--- |
| `semaphore.task.status[{#ID}]` | `UNSIGNED` (`Semaphore task status`) | — | Last task status (`0`=Stopped, `1`=Success, `2`=Error, `3`=Running, `4`=Waiting, `5`=Unknown) |
| `semaphore.task.id[{#ID}]` | `UNSIGNED` | `1d` | Last task execution ID (`last_task.id`) |
| `semaphore.task.start_time[{#ID}]` | `UNSIGNED` (`unixtime`) | `1d` | Last task start timestamp in epoch seconds |
| `semaphore.task.end_time[{#ID}]` | `UNSIGNED` (`unixtime`) | `1d` | Last task end timestamp in epoch seconds (discarded while running) |
| `semaphore.task.playbook[{#ID}]` | `CHAR` | `1d` | Playbook filename from `last_task.tpl_playbook` |
| `semaphore.task.app[{#ID}]` | `CHAR` | `1d` | Runner app type (`app`) |

### Triggers & Trigger Prototypes

| Trigger Name | Problem Expression | Recovery Expression | Severity |
| :--- | :--- | :--- | :---: |
| `Semaphore: API is unreachable on {HOST.NAME}` | `max(/.../semaphore.ping,#3)=0 or nodata(/.../semaphore.ping,3m)=1` | `min(/.../semaphore.ping,#2)=1` | Average |
| `Semaphore: Failed to fetch project templates on {HOST.NAME}` | `nodata(/.../semaphore.raw,15m)=1` *(depends on API unreachable)* | *(auto on data)* | Warning |
| `Semaphore: Version has changed on {HOST.NAME}` | `last(#1)<>last(#2) and length(last())>0` | Manual close | Info |
| `Semaphore: Task "{#NAME}" failed on {HOST.NAME}` | `last(semaphore.task.status[{#ID}])=2 and {$SEMAPHORE.TRIGGER.ENABLED:"{#ID}"}=1` | `last(semaphore.task.status[{#ID}])=1 or {$SEMAPHORE.TRIGGER.ENABLED:"{#ID}"}=0` | Warning |
| `Semaphore: Task "{#NAME}" running longer than {$SEMAPHORE.TASK.RUNNING.MAX.TIME:"{#ID}"} on {HOST.NAME}` | `min(...,{$SEMAPHORE.TASK.RUNNING.MAX.TIME:"{#ID}"})=3 and count(...,{$SEMAPHORE.TASK.RUNNING.MAX.TIME:"{#ID}"})>=3 and {$SEMAPHORE.TRIGGER.ENABLED:"{#ID}"}=1` | `last(semaphore.task.status[{#ID}])<>3 or {$SEMAPHORE.TRIGGER.ENABLED:"{#ID}"}=0` | Warning |

---

## Value Maps

- **`Service state`**: `0` → `Down`, `1` → `Up`
- **`Semaphore task status`**: `0` → `Stopped`, `1` → `Success`, `2` → `Error`, `3` → `Running`, `4` → `Waiting`, `5` → `Unknown`

---

## Monitoring Multiple Semaphore Projects

Create one Zabbix host per Semaphore project and set `{$SEMAPHORE.PROJECT.ID}` individually per host.

---

> **Upstream attribution:** Adapted from [Garfieldttt/zabbix-semaphore-ansible](https://github.com/Garfieldttt/zabbix-semaphore-ansible/tree/zabbix).
