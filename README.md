# cluster_management


This repository contains an Ansible build for the CompNeuro compute cluster.



- LDAP 
    - Lives within the ldap_client role
- Slurm 
    - sss 
- lmod 
    - ll
- OnDemand (OOD)
    - Lives within the ood role
    - setup on the login node

    - a web front-end that shells out to existing cluster tools 
    - Every feature in OOD is the same commands a user would type by hand just triggered by the web interface
    - OOD does not run a shared web server for everyone, it spawens a separate process as each user, so a job submitted through the portal runs with that person's acutal Unix permissions, not some shared "ondemand" service account. 
    - Apps are just templated Slurm scripts  
- BeeGFS


Cluster setup:

Habenula: Management: slurmctld, slurmdbd, MariaDB, BeeGFS mgmt + compute (CoreSpecCount reserved)

Retina: Login
Insula, Amygdala : BeeGFS metadata + compute + client
Cortex, Cerebellum, Cochlea, Thalamus, Hypothalamus + PFC : Compute + BeeGFS storage targets + client

Storage Tiers:
- BeeGFS scratch: - 7.2 TB raw across seven nodes, no mirroring, no backup, converged with compute. 
- Local storage server: > 400 TB, backedup, slow writes
- NAS - 50 TB, backed up 

- The BeeGFS needs a documented age-based cleanup 
- Mounting Storage mount as read-only on compute nodes



- Webservices
    - https://slurm-web.com/#features

How to get TLS Certificates for the webservices?