# Local Slurm cluster (for BIOMERO)

This is a multi-container Slurm cluster using docker-compose.  The compose file
creates named volumes for persistent storage of MySQL data files as well as
Slurm state and log directories.

## Quickstart

Clone this repository locally

    git clone https://github.com/Cellular-Imaging-Amsterdam-UMC/NL-BIOMERO-Local-Slurm

Change into the new directory

    cd NL-BIOMERO-Local-Slurm

Copy your public SSH key into this directory, to allow SSH access

    cp ~/.ssh/id_rsa.pub .

Build and run the Slurm cluster containers

    docker-compose up -d --build

Now you can access Slurm through SSH:

    ssh -i ~/.ssh/id_rsa -p 2222 -o StrictHostKeyChecking=no slurm@host.docker.internal

Or:

    ssh -i ~/.ssh/id_rsa -p 2222 -o StrictHostKeyChecking=no slurm@localhost

Done.

For example, run:

    sbatch -n 1 --wrap "hostname > lolcow.log && singularity run docker://godlovedc/lolcow >> lolcow.log"

This should say "Submitted batch job 1"
Then let's tail the logfile:

    tail -f lolcow.log

First we see the slurm node that is computing, and later we will see the funny cow.

```bash
[slurm@slurmctld data]$ tail -f lolcow.log
c1
 _______________________________________
/ Must I hold a candle to my shames?    \
|                                       |
| -- William Shakespeare, "The Merchant |
\ of Venice"                            /
 ---------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Exit logs with `CTRL+C`, and the container with `exit`, and enjoy your local Slurm cluster.

## New Features

We added the following features to this (forked) cluster:
- Running SSH on the SlurmCTLD, you can connect to it at `host.docker.internal:2222`
- Running Singularity, you can fire off any singularity container now, e.g.:

    `sbatch -n 1 --wrap "hostname > lolcow.log && singularity run docker://godlovedc/lolcow >> lolcow.log"`

This should give a funny cow in lolcow.log and the host farm on which the cow was grazing.

Note: Like always be sure to run Slurm commands from `/data`, the shared folder/volume. Otherwise it won't be able to share the created logfile.


We have not added:
- GPU support

## Docker specifics 

To stop the cluster:

    docker-compose down

N.B. Data is stored on Docker volumes, which are not automatically deleted when you down the setup. Convenient.

To remove volumes as well:

    docker-compose down --volumes

To rebuild a single container (while running your cluster):

    docker-compose up -d --build <name>

To attach to a running container:

    docker-compose exec <name> /bin/bash

Where `<name>` is e.g. `slurmctld` or `c1`

Exit back to your commandline by typing `exit`.

Or check the logs

    docker-compose logs -f 

Exit with CTRL+C (only exits the logs, does not shut down the container)

## Containers and Volumes

The compose file will run the following containers:

* mysql
* slurmdbd
* slurmctld
* c1 (slurmd)
* c2 (slurmd)

The compose file will create the following named volumes:

* etc_munge         ( -> /etc/munge     )
* etc_slurm         ( -> /etc/slurm     )
* slurm_jobdir      ( -> /data          )
* var_lib_mysql     ( -> /var/lib/mysql )
* var_log_slurm     ( -> /var/log/slurm )

## Slurm specifics

### Register the Cluster with SlurmDBD

To register the cluster to the slurmdbd daemon, run the `register_cluster.sh`
script:

```console
./register_cluster.sh
```

> Note: You may have to wait a few seconds for the cluster daemons to become
> ready before registering the cluster.  Otherwise, you may get an error such
> as **sacctmgr: error: Problem talking to the database: Connection refused**.
>
> You can check the status of the cluster by viewing the logs: `docker-compose
> logs -f`

### Accessing the Cluster

Use `docker exec` to run a bash shell on the controller container:

```console
docker exec -it slurmctld bash
```

From the shell, execute slurm commands, for example:

```console
[root@slurmctld /]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      2   idle c[1-2]
```

### Submitting Jobs

The `slurm_jobdir` named volume is mounted on each Slurm container as `/data`.
Therefore, in order to see job output files while on the controller, change to
the `/data` directory when on the **slurmctld** container and then submit a job:

```console
[root@slurmctld /]# cd /data/
[root@slurmctld data]# sbatch --wrap="uptime"
Submitted batch job 2
[root@slurmctld data]# ls
slurm-2.out
```
