# Interactive Brokers Gateway Docker

## Status: Archived (June/2021)!
* Not used in my system anymore! 
* Not updated anymore!
* Not maintained anymore!
* The docker container may still work as it is, but use it at your own risk. 
* I have moved past IB because of the the poor API and the notorious unstable gateway.

### Based on the following work:

* [IB Controller](https://github.com/ib-controller/ib-controller/) and VNC
* [ib-gateway-docker](https://github.com/mvberg/ib-gateway-docker)
* [Interactive Brokers Gateway Docker Image](https://hub.docker.com/r/mvberg/ib-gateway-docker)
* All credits to the respective contributor(s). 

### Versions:
* TWS Gateway: v974.4g
* IB Controller: v3.2.0

### Modifications:
* Script refactoring
* Smaller container (~100 MB less than initial Ubuntu build)
* Kubernetes deployment configurations
* Updated Readme

### Key features:

* Dockerized & fully pre-configured 
* Starts IB Gateway automatically after boot
* Log in into the gateway fully automatically
* Clicks the dialog box automatically
* Keeps the gateway up and running permanently when using default settings.
* VNC remote access to the running docker instance.
* Kubernetes deployment descriptor for GCP (no VNC access)  
* Cloud Build auto deployment descriptor for GCP 

### Known issues:

* Password must be plaintext 
* No VNC access to Kubernetes deployment 
* A bit dated gateway version but it runs very stable.

### TODO's

The password could be encrypted through some IB command line tool,
but the exact details needs to be figured out.  

### Observations;
* IB Gateway runs remarkable stable on the default settings
* Clients must close any connection correctly otherwise a manual restart becomes required to terminated open connections.
* It seems that the daily reset also closes possibly unterminated connections and keeps memory usage in check.
* No manual intervention needed as long as all clients connect and disconnect correctly.
* In case a client crashes without a proper connection close, usually a reconnect fails.
* Either use a new connection-ID and restart the gateway manually or, if feasible and non-time critical, 
wait some 24 hours to see if the auto-reset did the trick. 
* From experience, a number of clients works well as long as each connection terminates correctly. 

### IB System Status

https://www1.interactivebrokers.com/en/index.php?f=en/software/systemStatus.php

SERVER RESET TIMES	NORTH AMERICA
Saturday - Thursday	23:45 - 00:45 ET 
Friday	23:00 - 03:00 ET

During the Friday evening reset period, all services will be unavailable 
in all regions for the duration of the reset.


## Getting Started with docker

1) Clone repo: git clone

2) Edit docker-compose.yml
    * Set TRADING_MODE=paper # or live 
    * Set TWSUSERID=ib_username # i.e. jobo1960
    * Set TWSPASSWORD=secret # Must be plaintext, no quotes 

3) Build & run docker:  

```bash
> cd ib-gateway-docker
> docker build .
> docker-compose up
```

#### Expected output

```bash
Creating ibgatewaydocker_tws_1 ...
Creating ibgatewaydocker_tws_1 ... done
Attaching to ibgatewaydocker_tws_1
tws_1  | Starting virtual X frame buffer: Xvfb.
tws_1  | find: '/opt/IBController/Logs': No such file or directory
tws_1  | stored passwd in file: /.vnc/passwd
tws_1  | Starting x11vnc.
tws_1  |
tws_1  | +==============================================================================
tws_1  | +
tws_1  | + IBController version 3.2.0
tws_1  | +
tws_1  | + Running GATEWAY 960
tws_1  | +
tws_1  | + Diagnostic information is logged in:
tws_1  | +
tws_1  | + /opt/IBController/Logs/ibc-3.2.0_GATEWAY-960_Tuesday.txt
tws_1  | +
tws_1  | +
tws_1  | Forking :::4001 onto 0.0.0.0:4003\n
```

You will now have the IB Gateway app running on port 4003 and VNC on 5901.

See [docker-compose.yml](docker-compose.yml) for configuring VNC password, accounts and trading mode.

Please do not open your box to the internet.

### Testing VNC

* localhost:5901

![vnc](docs/ib_gateway_vnc.jpg)

### Demo Accounts

It seems that all of the `demo` accounts are dead for good:

* edemo
* fdemo
* pmdemo

### Troubleshooting

Sometimes, when running in non-daemon mode, you will see this:

```java
Exception in thread "main" java.awt.AWTError: Can't connect to X11 window server using ':0' as the value of the DISPLAY variable.
```

You will have to remove the container `docker rm container_id` and run `docker-compose up` again.


## Getting Started with Kubernetes

1) Clone repo: git clone

2) Edit deployment.yaml
    * Set TRADING_MODE=paper # or live 
    * Set TWSUSERID=ib_username # i.e. jobo1960
    * Set TWSPASSWORD=secret # Must be plaintext, no quotes 

3) Edit cloudbuild.yaml
    * Set cluster name & location for auto-deployment
    * Leave image name as it is otherwise Deployment.yaml needs to be updated as well.

4) Manual Deployment (for testing):
    * Ensure docker image is in registry
    * kubectl apply -f deployment.yaml 

5) Auto Deployment on GCP:
    * Setup a new git repo on GCP
    * Change master to new origin
    * Set build trigger in cloud-build
    * Push to repo 
    * Let cloud-build build, store, and deploy the container following each push.

## Localhost access to the gateway:

> kubectl get services 

Expected output on a new cluster:
```bash
>kubectl get services
NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
ib-gateway-service   ClusterIP   10.4.15.215   <none>        4003/TCP,5901/TCP   2d18h
kubernetes           ClusterIP   10.4.0.1      <none>        443/TCP             5d18h
```

> kubectl port-forward service/ib-gateway-service 4003


Expected output:
```bash
Forwarding from 127.0.0.1:4003 -> 4003
Forwarding from [::1]:4003 -> 4003
```

Thereafter, any connection to 127.0.0.1:4003 gets routed through SSL to
the cluster service. Kubernetes resolves the DNS name automatically and routes from the 
virtual service name to the matching pod in the cluster.

Programming the IB gateway on Kubernetes:

The only applicable rule is:

"Always connect to the service name"

> ib-gateway-service:4003 

Kubernetes assigns IP addresses dynamically, therefore any hard coded
IP addresses almost certainly will be invalidated after a new deployment leading
to connection timout errors. At the same time, kubernetes updates the internal DNS 
system dynamically so whenever a pod got replaced, the new IP already is in the DNS
resolver at the moment the pod comes online. Therefore, *always connect to the service name*

