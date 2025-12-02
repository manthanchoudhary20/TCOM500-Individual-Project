## Project overview

This project builds a lab of four VyOS routers running as Docker containers, with their management plane reachable from an Ansible control host over SSH. It demonstrates how to use Ansible to push static route configuration changes and validate routing state before and after the change using `show ip route` outputs saved as artifacts in the repo.

## Topology and repo layout

The lab topology is:

- 4 VyOS routers (r1–r4) running as Docker containers.
- One management network (`172.30.0.0/24`) used for SSH/Ansible.
- Three point-to-point data-plane networks:
  - `172.31.12.0/29` (r1–r2)
  - `172.31.23.0/29` (r2–r3)
  - `172.31.34.0/29` (r3–r4)

Key files in this repository:

- `docker-compose.yml` – defines the r1–r4 containers and their Docker networks.
- `inventories/hosts.ini` – Ansible inventory with the `[routers]` group (r1–r4 management IPs).
- `group_vars/routers.yml` – group variables defining the initial static routes (`routes_init`) and the route‑change (`routes_change`) for each router.
- `roles/baseline/` – baseline configuration role (hostname, SSH, and common system settings).
- `roles/routes/` – routes role that generates `set protocols static route ...` commands from variables and pushes them using `vyos-commands-to-config`.
- `playbooks/baseline.yml` – applies the baseline config to all routers.
- `playbooks/routes_init.yml` – pushes the initial static routes for baseline connectivity.
- `playbooks/route_change.yml` – applies an additional static route advertising a non‑adjacent subnet.
- `playbooks/verify.yml` – runs `show ip route` on all routers and saves outputs under `outputs/before/` and `outputs/after/`.
- `outputs/before/` – captured routing tables before the route change.
- `outputs/after/` – captured routing tables after the route change.

## Prerequisites and setup

Controller requirements:

- Docker and Docker Compose installed.
- Python 3 and `venv` available.
- Ansible installed in a virtual environment.
- `sshpass` installed (required for Ansible SSH with passwords).

Example setup on the control host(for someone cloning this repo):

``` git clone https://github.com/https://github.com/Xanquisher/Network-Automation.git vyos-static-route-lab```
``` cd vyos-static-route-lab/ansible```
``` python3 -m venv venv```
``` source venv/bin/activate```
``` pip install ansible```
``` sudo apt update```
``` sudo apt install -y sshpass```

Bring up the VyOS lab:
``` cd vyos-static-route-lab/ansible```
``` docker compose up -d```
``` docker compose ps```

At this point r1–r4 should be running, each with a management IP on `172.30.0.0/24`.

## Running the lab

All commands below assume the current directory is the Ansible project root:

```cd ~/vyos-project/ansible```


### 1. Apply baseline configuration (optional but recommended)

Baseline covers common system settings such as hostname and SSH service.

```ansible-playbook playbooks/baseline.yml```

### 2. Push initial static routes (baseline connectivity)

Use the `routes` role with the `routes_init` variables to configure initial static routes across r1–r4.

```ansible-playbook playbooks/routes_init.yml -e "route_phase=init"```

This playbook generates `set protocols static route <prefix> next-hop <address>` commands from `group_vars/routers.yml` and applies them on each router using `vyos-commands-to-config`.

### 3. Capture BEFORE routing state

Run the verify playbook to capture `show ip route` on all routers before the route change:

```ansible-playbook playbooks/verify.yml -e verify_phase=before```

This will create:

- `outputs/before/r1_route.txt`
- `outputs/before/r2_route.txt`
- `outputs/before/r3_route.txt`
- `outputs/before/r4_route.txt`

### 4. Apply route change (non-adjacent subnet)

Apply the route-change playbook to add a static route for a non‑adjacent subnet (for example `10.4.99.0/24` behind r4 that r1 reaches via r2 and r3):

```ansible-playbook playbooks/route_change.yml -e "route_phase=change"```

### 5. Capture AFTER routing state

Run verify again to capture `show ip route` on all routers after the route change:

ansible-playbook playbooks/verify.yml -e verify_phase=after

This will create:

- `outputs/after/r1_route.txt`
- `outputs/after/r2_route.txt`
- `outputs/after/r3_route.txt`
- `outputs/after/r4_route.txt`

## Interpreting the outputs

The `outputs/` directory contains plain-text snapshots of the routing table on each router before and after the change:

- `outputs/before/*.txt` – routing tables after `routes_init.yml` but before applying `route_change.yml`.
- `outputs/after/*.txt` – routing tables after applying `route_change.yml`.

For example, to inspect r1:

```cat outputs/before/r1_route.txt```
```cat outputs/after/r1_route.txt```


The `routes_init.yml` playbook installs static routes that provide baseline reachability between subnets attached to r1 and r4. The `route_change.yml` playbook adds a new static route for a non‑adjacent subnet (`10.4.99.0/24`) learned via the r2–r3–r4 chain. The Ansible play logs show the exact `set protocols static route 10.4.99.0/24 next-hop ...` commands executed on r1–r3, and the `outputs/after/*.txt` files show the post‑change routing state on each router.

> Note: This lab uses a rolling VyOS Docker image whose `vyos-op-run ping` helper currently returns an internal error. For simplicity and reliability, routing validation in this lab is done via `show ip route` snapshots and Ansible logs, rather than automated ping/traceroute.
