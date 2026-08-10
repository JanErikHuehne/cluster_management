# cluster_management


This repository contains an Ansible build for the CompNeuro compute cluster.



- LDAP 
    - sss
- Slurm 
    - sss 
- lmod 
    - ll
- OnDemand (OOD)
    - a web front-end that shells out to existing cluster tools 
    - Every feature in OOD is the same commands a user would type by hand just triggered by the web interface
    - OOD does not run a shared web server for everyone, it spawens a separate process as each user, so a job submitted through the portal runs with that person's acutal Unix permissions, not some shared "ondemand" service account. 
    - Apps are just templated Slurm scripts  
- BeeGFS
