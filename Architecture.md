# Hailstorm Architecture

The demo has a layered Architecture which looks as follows:

                             +---------+
                             |Container|
                             |(12)     |                                  Layer 4
                             +---------+--+
                             |OpenShift   |
                             |V3.x (11)   |
                             +----------------+ +-----+                   Layer 3
                             | Virtual Guests | |Cloud|                  
                             | / Instances (9)| |Forms|(10)
     +---------+ +---------+ +----------+------------+ +-------+ +---+
     |Satellite| |Infra-   | |Nested    | |Nested    | |RH     | |RH |
     |         | |structure| |Hypervisor| |Hypervisor| |Storage| |IDM|    Layer 2
     |      (3)| |      (4)| |1      (5)| |2      (6)| |    (7)| |(8)|
    ++-----------+-------------------------------------+--------------+
    |                        RHEL (KVM Host)     (2)                  |   Layer 1
    +-----------------------------------------------------------------+
    |                         Hardware           (1)                  |   Layer 0
    +-----------------------------------------------------------------+


From bottom to top:

1. Hardware:
  - A single, portable 2 HU server
  - plenty of CPU cores
  - plenty of RAM
  - RAID6 protected storage
  - Fast SSD storage
2. RHEL (KVM Host):
  - Base Operating System, to be defined whether RHEL or RHEL-Atomic Host.
  - Configured to allow nested virtualization.
  - KVM based virtualization allows to partition the hardware
  - Multiple internal VLANs
    - How? OpenSwitch virtual switches?
  - NAT enables a private IP space which can be connected into any LAN
    - To be discussed: How to enable DNS + Mail in NAT scenario
  - Runtime environment for scripts / playbooks / ... which
    1. configure the OS itself (Kernel Params, OVS config, NAT config, ...)
    2. bring up the KVM instances running on top of it
  - There is NO explicit GUI / WebUI on this layer
3. Satellite:
  - Core service (not in nested virt)
  - Subscription Management for ALL virtual machines on Layer 2 and up
  - Content Views for ALL virtual machines on Layer 2 and up
4. Infrastructure:
  - Core service (not in nested virt)
  - may be one or more virtual machines
  - DNS server
  - To be discussed: DHCP server
  - Mail (SMTP+IMAP) server
  - NFS Server (e.g. for RHEV)
5. Nested Hypervisor 1: classic HV for Mode 1 workloads
  - one RHEV-M
  - two RHEV-H
6. Nested Hypervisor 2: OpenStack for Mode 2 workloads
  - one RHEL-OSP installer/director
  - three RHEL-OSP controllers
  - three RHEL-OSP nodes
  - additional empty VMs to simulate rollout of new OSP instance
7. Red Hat Storage (OPTIONAL):
  - Optional component to provide NFS, Block or Object Storage
  - Augments or replaces NFS server in 4. Infrastructure
8. Red Hat Identity Management and/or KeyCloak (OPTIONAL):
  - Single point of authentication
  - Primarily for web authentication
  - LDAP containing SSH public keys so we don't have to copy&paste keys
9. Virtual Guests / Instances
  - Image-based deployment of RHEL and Windows
  - (potentially) Subscribed in 3. Satellite
10. CloudForms
  - Guest on RHEV (Nested Hypervisor 1)
  - Manages underlying Hypervisors
  - Manages OpenShift (Note: even though the graphic suggests otherwise, CF and OpenShift are on the same layer (3)).
11. OpenShift V3
  - One or three Masters
  - two Nodes
  - Potentially rolled out via CloudForms
  - Potentially rolled out via OpenStack HEAT
  - Potentially rolled out just via Ansible Script
12. Container
  - Any containerized workload
  - Developed in OpenShift or externally

## Principles
- Dependencies may only go down or sideways, never up (e.g. from Layer 2 to Layer 3)
- Componentization / Design for upgradability: Strive for self-contained components so that they can be upgraded independently of each other. For example:
  - RHEL-OSP with default SDN can be replaced whith RHEL-OSP with 3rd party SDN without affecting any other component
  - A subset of the demo with smaller footprint can be extracted easily, e.g. just OpenStack (running on bare metal) + CloudForms + OpenShift
- Immutable Infrastructure: No manual configuration, EVERYTHING needs to be scripted.
  - "Scripts" in this context means code that is checked into a repository,
  be it puppet classes, ansible playbooks, shell scripts or similar
  - Events to create scripts for:
    - Creation
    - Deletion
    - Startup
    - Shutdown
    - Data Export (e.g. satellite subscription data, openstack base images, ....)
    - Data Import
    - Reset to known state (demo reset)
  - This will reduce the need for backup / restore: Instead of backup up/restoring a known good configuration, we can recreate it from scratch
- Test (Driven/Augmented) Development: For all use cases, tests should be added
to the repository so a correct behavior can be validated. This includes testing the scripts to set up / tear down components as well as testing demo use cases.
  - To be discussed:
    - Level/Rigor of testing
    - Test Automation / Continuous Build