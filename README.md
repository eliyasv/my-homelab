# MY DevOps Homelab (on-demand)

This repo is my **personal DevOps homelab** built on a spare laptop.

This is a rebuildable, on-demand bare-metal DevOps lab to exercise platform design decisions without cloud abstractions. Ansible is used only for immutable host and cluster bootstrap, after which Argo CD owns all Kubernetes state via GitOps.
The stack deliberately uses k3s, Traefik, and MetalLB to surface failure modes that cloud providers typically abstract away. This lab prioritizes rebuildability over uptime and reconciliation over manual intervention. It is expected to be stopped, broken, and rebuilt.

The lab is **on-demand**. I don’t keep it running all the time. I start it when I want to practice, and shut it down when I’m done.

---

## Hardware

- Laptop (repurposed)
- AMD Ryzen 3 3050U
- 12 GB RAM
- Wi-Fi (dynamic IP)
- Used like a server, but not always running

---

## Stack (current)

- **OS:** Ubuntu Desktop (used headless)
- **Automation:** Ansible
- **Containers:** Docker
- **Kubernetes:** k3s (single node)
- **Ingress:** Traefik
- **LoadBalancer:** MetalLB (L2 mode)
- **GitOps:** Argo CD

Planned:
- GitHub Actions (CI)
- cert-manager (HTTPS)

---

## On-demand workflow

The lab is not always on.

### Start
```
start-homelab

```
Starts Docker, k3s, and converges state with Ansible.

### Stop
```
stop-homelab

```
Stops k3s and Docker. MetalLB releases the IP cleanly.

## Current state

- Kubernetes cluster up and stable

- Traefik ingress working

- MetalLB assigning real IPs

- Argo CD accessible behind Traefik

- Apps and infra managed via GitOps

- Services reachable via hostname

- Lab can be stopped and started anytime

## What’s next

- Install Argo CD

- Move app deployments to GitOps

- Add CI with GitHub Actions

- Break things on purpose and learn to fix them

## Why i build this

This is not a “perfect” setup.
It’s a learning lab.

The goal is to understand DevOps culture:

automation

ownership

debugging

iteration

## Some early hurdles I ran into

This lab wasn’t smooth, and that’s kind of the point.
Here are some of the real issues I hit while building it.

### Ubuntu Server vs Desktop reality
- Wi-Fi didn’t work reliably on Ubuntu Server on this laptop
- NetworkManager wasn’t available by default
- Spent time debugging networking before realizing Desktop was the practical choice
- Switched to Ubuntu Desktop and ran it headless

---

### SSH hardening broke access (temporarily)
- Disabled password auth and root login
- Didn’t have keys fully sorted on all machines
- Locked myself out once and had to recover locally

---

### Ansible hanging on “Gathering Facts”
- Playbooks would hang for minutes with no output
- Turned out to be privilege escalation + sudo prompt issues
- Fixed inventory and `become` usage

---

### Sudoers mistakes are unforgiving
- Accidentally broke sudo by misreading include paths
- Nearly lost admin access
- Had to recover carefully without rebooting into chaos

---

### Docker vs Kubernetes networking confusion
- Assumed Docker networks and Kubernetes networking behaved similarly
- Learned the hard way they are completely different layers

---

### k3s kubeconfig permissions
- `kubectl` kept trying to read `/etc/rancher/k3s/k3s.yaml`
- Permission denied errors even though kubeconfig existed
- Learned how `KUBECONFIG` resolution actually works

---

### Ingress does nothing without a controller
- Created Ingress resources and expected them to “just work”
- Nothing happened
- Realized Ingress is only rules, not functionality

---

### DNS mistakes break everything silently
- Pointed hostname to `127.0.1.1`
- Ingress never worked, no obvious error
- Fixed by pointing to node / LB IP

---

### NodePort ≠ port 80
- Expected ingress to respond on port 80
- Found out NodePorts map to high ports only
- Spent time understanding why this is intentional by design, for safety and practicality

---

### nginx-ingress removal left broken webhooks
- Removed nginx-ingress but admission webhooks stayed
- New Ingress objects failed with cryptic errors
- Had to manually delete webhook configs

---

### Traefik Helm values are not intuitive
- Tried to change ports with Helm values
- Traefik kept listening on 8000/8080
- Learned to read chart schemas instead of guessing

---

### Bare-metal Kubernetes is different from cloud (cloud abstractions hide a lot.)
- LoadBalancer services don’t work magically
- `EXTERNAL-IP: <pending>` is normal
- Learned why MetalLB exists

---

### MetalLB + Wi-Fi considerations
- Needed an IP pool that wouldn’t conflict with DHCP
- Chose a small, safe range
- Verified ARP-based L2 mode works fine on Wi-Fi

---

### MetalLB IP assigned but unreachable
- MetalLB showed an IP, but nothing responded
- Root cause - no Service of type LoadBalancer was using it
- Fixed by patching Traefik Service to LoadBalancer

---

### Traefik returning 404 despite working networking
- Requests reached Traefik but returned 404
- Traefik ignored the Ingress
- Fixed by explicitly setting ingressClassName: traefik

---

### Infinite redirect loop accessing Argo CD
- Browser showed ERR_TOO_MANY_REDIRECTS
- TLS was terminated twice- Traefik + Argo CD
- Fixed by running Argo CD behind a reverse proxy (server.insecure=true) (serves http)

---

### 502 Bad Gateway after fixing redirects
- Traefik forwarded HTTPS to an HTTP backend
- Protocol mismatch caused 502
- Fixed by explicitly telling Traefik to use HTTP for the backend

---
