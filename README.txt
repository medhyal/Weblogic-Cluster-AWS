1. On Manager/Admin VM
git clone https://github.com/oracle/docker-images.git

2. Download server JRE and WebLogic generic version, see github for more details
3. Build Oracle JRE first
4. CD to Weblogic
5. update domain.propertes
6. Comment line in Dockerfile
#HEALTHCHECK --start-period=10s --timeout=30s --retries=3  CMD curl -k -s --fail http://localhost:$ADMIN_LISTEN_PORT/weblogic/ready || exit 1

7. Build Weblogic generic 12.2.1.3 version
./buildDockerImage.sh -v 12.2.1.3 -g -s

8. CD to samples/12213-domain-home-in-image
9. Edit Dockerfile to chane to
FROM oracle/weblogic:12.2.1.3-generic

10. Update domain_security.properties
11. Update file security.properties
12 Run
. container-scripts/setEnv.sh ./properties/docker-build/domain.properties
docker build --force-rm=true --no-cache=true $BUILD_ARG -t 12213-domain-home-in-image .

13. export image
docker save -o 12213-domain-home-in-image.tar 12213-domain-home-in-image:latest

14. copy file 12213-domain-home-in-image.tar to worker nodes (ec2 instances)
 - Update known_hosts and authorized_keys accordingly to scp the file.
 
15. config docker swarm and network

On Admin node:
docker swarm init
docker network create -d overlay wls

On worker nodes:
docker swarm join --token SWMTKN-1-2optb5h86jmvf8tadbldvkxd32mo7b7erq31idtu8csgxfjpix-7x2dpniyspqhtilsgouegrfje 10.30.1.18:2377

16. Update below ports in EC2 securtity group for all Worker nodes
worker ports: 
tcp 2377
tcp 8001
tcp 7946
udp 7946
udp 4789

17. Update below ports in EC2 securtity group for master/admin nodes
master ports:
tcp 2377
tcp 7001
tcp 7946
udp 7946
udp 4789

18. On Master node run this to deploy service.

docker service create \
  --name wlsadmin \
  --network wls \
  --hostname wlsadmin \
  --replicas 1 \
  --publish published=7001,target=7001 \
  --no-healthcheck \
  --mount type=bind,source=/home/ec2-user/docker-images/OracleWebLogic/samples/12213-domain-home-in-image/properties/docker-run,destination=/u01/oracle/properties \
  12213-domain-home-in-image
  
  
docker service create \
  --name wlsms1 \
  --network wls \
  --replicas 1 \
  --publish published=8001,target=8001 \
  --env MANAGED_SERV_NAME=managed-server1 \
  --no-healthcheck \
  --mount type=bind,source=/home/ec2-user/docker-images/OracleWebLogic/samples/12213-domain-home-in-image/properties/docker-run,destination=/u01/oracle/properties \
  12213-domain-home-in-image startManagedServer.sh
  
  
optional:
docker service create \
  --name wlsms2 \
  --network wls \
  --replicas 1 \
  --publish published=8002,target=8001 \
  --env MANAGED_SERV_NAME=managed-server2 \
  --no-healthcheck \
  --mount type=bind,source=/home/ec2-user/docker-images/OracleWebLogic/samples/12213-domain-home-in-image/properties/docker-run,destination=/u01/oracle/properties \
  12213-domain-home-in-image startManagedServer.sh
  
19. Copy demo app from samples/12213-deploy-application
 copy to admin docker container
 
docker cp container-scripts/ 0cfa7d481c97:/u01/oracle
docker cp sample 0cfa7d481c97:/u01/oracle
docker cp build-archive.sh 0cfa7d481c97:/u01/oracle

20. Login to Admin container
Run /u01/oracle/build-archive.sh

21. Login to weblogic console
22. create deployment
23. once in prepared state, deploy from control tab

