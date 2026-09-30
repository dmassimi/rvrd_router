# TIBCO RENDEZVOUS: MULTI-SUBNET DOCKER ROUTING (RVRD) TEST

This project demonstrates a proof-of-concept for routing TIBCO Rendezvous (RV) messages between three completely isolated Docker bridge networks (Subnets A, B, and C) using a central Routing Daemon (rvrd) running on the Windows host machine.

Since standard TIBCO RV relies on UDP multicast/broadcast, it cannot natively cross the network boundaries established by Docker subnets. By configuring the containerized applications to connect via TCP directly to the host rvrd (acting as a central hub), we can bridge these subnets.


## ARCHITECTURE DIAGRAM
![Architecture](img/architecture.png)

The topology runs four containers: one publisher on Subnet A (`rv-sender`, service 7501), two replicas of the Subnet B subscriber (`rv-listener-a-1` and `rv-listener-a-2`, service 7502) and one Subnet C subscriber (`rv-listener-b`, service 7503).


## PREREQUISITES

* Docker Desktop: Installed and running on Windows.
* TIBCO Rendezvous (Host): Installed on the Windows host machine.
* TIBCO License File: Available at C:\Users\dmassimi\containers\resources\addons\license\dmassimi-vdi_ANY.bin
* TIBCO Docker Image: A working Docker image tagged tibco-rv:9.0.0 available locally.


## PROJECT FILES: docker-compose.yml

```yaml
services:
  # 1. Publisher Container on Subnet A (Service 7501)
  rv-sender:
    image: tibco-rv:9.0.0
    container_name: rv-sender
    hostname: localhost
    volumes:
      - C:\Users\dmassimi\containers\resources\addons\license:/data/license
    environment:
      - TIBRV_LICENSE=file:///data/license/dmassimi-vdi_ANY.bin
    command: >
      bash -c "tibrvlisten -service 7501 -daemon tcp:host.docker.internal:7500 TEST.ACK & sleep 2; while true; do echo 'Publishing to NET-A...'; tibrvsend -service 7501 -daemon tcp:host.docker.internal:7500 TEST.SUBJECT 'Hello from Subnet A!'; sleep 5; done"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      subnet_a:
        ipv4_address: 172.20.0.10

  # 2. Subscriber Containers on Subnet B (Service 7502)
  rv-listener-a:
    image: tibco-rv:9.0.0
    #container_name: rv-listener
    hostname: localhost
    volumes:
      - C:\Users\dmassimi\containers\resources\addons\license:/data/license
    environment:
      - TIBRV_LICENSE=file:///data/license/dmassimi-vdi_ANY.bin
    # Listens specifically for TEST.SUBJECT on Service 7502
    command: >
      bash -c "tibrvlisten -service 7502 -daemon tcp:host.docker.internal:7500 TEST.SUBJECT"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    # Two replicas: Docker creates rv-listener-a-1 and rv-listener-a-2
    deploy:
      replicas: 2
    networks:
      - subnet_b

  # 3. Second Subscriber Container on Subnet C (Service 7503)
  rv-listener-b:
    image: tibco-rv:9.0.0
    container_name: rv-listener-b
    hostname: localhost
    volumes:
      - C:\Users\dmassimi\containers\resources\addons\license:/data/license
    environment:
      - TIBRV_LICENSE=file:///data/license/dmassimi-vdi_ANY.bin
    command: >
      bash -c "tibrvlisten -service 7503 -daemon tcp:host.docker.internal:7500 TEST.SUBJECT"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      subnet_c:
        ipv4_address: 172.22.0.10

networks:
  subnet_a:
    ipam:
      config:
        - subnet: 172.20.0.0/16
  subnet_b:
    ipam:
      config:
        - subnet: 172.21.0.0/16
  subnet_c:
    ipam:
      config:
        - subnet: 172.22.0.0/16

```

## CONFIGURATION & EXECUTION STEPS


**Step 1: Start Host rvrd**

1. Open a command prompt on your Windows host.
2. Start the Routing Daemon listening on port 7500 with the HTTP interface enabled on 7580:
```bash
   rvrd -listen tcp:7500 -http 7580
```

**Step 2: Configure the rvrd Router & Local Networks**

Open http://localhost:7580 in your browser to access the RVRD Web UI. 
Navigate to Configuration -> Routers and create a router named CENTRAL-ROUTER (click Add Router) if it does not already exist. The three local network interfaces added below are attached to this router:

![RVRD Web UI - Routers Configuration](img/router.png)

Next, go to Configuration -> Routers -> Local Networks.
Click Add Local Network three times to create the following entries:
* Name: NET-A, Service: 7501, Daemon: tcp:7500
* Name: NET-B, Service: 7502, Daemon: tcp:7500
* Name: NET-C, Service: 7503, Daemon: tcp:7500

![RVRD Web UI - Local Network Interfaces Configuration (CENTRAL-ROUTER)](img/networks.png)

**Step 3: Configure Subject Routing Rules**

1. In the Local Networks table, click on NET-A:
   * Under Export Subject List, click Add Export Subject.
   * Type: TEST.SUBJECT
   * Ensure Export Default Policy is set to Discard.
2. Return to Local Networks, click on NET-B:
   * Under Import Subject List, click Add Import Subject.
   * Type: TEST.SUBJECT
   * Ensure Import Default Policy is set to Discard.
3. Return to Local Networks, click on NET-C:
   * Under Import Subject List, click Add Import Subject.
   * Type: TEST.SUBJECT
   * Ensure Import Default Policy is set to Discard.

**Step 4: Start Containers**

Open a terminal in the directory containing your docker-compose.yml file and run:
```bash
docker compose up -d --force-recreate
```

Note: `deploy.replicas` is only honored by the Compose V2 CLI (`docker compose`). The legacy `docker-compose` (V1) binary ignores it unless you add `--compatibility`.

All four containers should start and stay running (rv-sender, rv-listener-a-1, rv-listener-a-2 and rv-listener-b):

![Docker Desktop - rv-sender, rv-listener-a-1, rv-listener-a-2 and rv-listener-b running](img/list-containers.png)


## VERIFICATION

To verify that routing is working, observe the logs of the subscriber containers.

Terminal 1 (rv-listener-b on Subnet C):
`docker logs -f rv-listener-b`

Terminal 2 (first rv-listener-a replica on Subnet B):
`docker logs -f rvrd_docker_project-rv-listener-a-1`

Terminal 3 (second rv-listener-a replica on Subnet B):
`docker logs -f rvrd_docker_project-rv-listener-a-2`

Expected Output (identical in all three terminals):

```console
2026-09-30 07:29:53: subject=TEST.SUBJECT, message={DATA="Hello from Subnet A!"}
```

Since rv-listener-a has no `container_name`, Docker prefixes each replica with the Compose project name (rvrd_docker_project here - it defaults to the folder name). List the exact names with `docker ps --format "{{.Names}}"`.

In the logs, rv-sender publishes once while all three subscribers receive the same message:

![Docker Desktop logs - rv-sender publishing and the rv-listener-a replicas / rv-listener-b receiving TEST.SUBJECT](img/logs.png)

The test is successful if a single message published by rv-sender is simultaneously received by both rv-listener-a replicas (Subnet B) and by rv-listener-b (Subnet C), despite the containers running on completely isolated network subnets.
