# Elastic Stack for Raspberry Pi

## Background
I wanted to be able to run a complete Elastic Stack on a Raspberry Pi (Low power) to act as a SIEM as well monitor my network. Elasticsearch itself has a build for Arm64 however Kibana did not, however after a little playing around I was able to get it to successfully run as a container as well as installed from a package. [My repo for this Kibana Arm can be found here](https://github.com/jamesgarside/kibana-arm).
I chose to deploy the Elastic Stack on Docker because that way I could take advantage of easy deployment to multiple nodes by utilising Docker Swarm. This will also run on a single node using Docker-Compose (TBC) or Swarm.

## To Do
- [x] Docker Swarm Config
- [ ] Enhance Docker Swarm Script
  - [ ] Add package checking
  - [ ] Automatic pre-deploy image pulling 
  - [ ] Migrate from Internal Monitoring to Metricbeat
- [ ] Docker Compose Config
- [ ] Script to install through APT for Debian/Ubuntu
- [ ] Script to install through Yum for RHFL/CentOS

## Requirements
- Raspberry Pi 4 (4GB+)
- Docker Compose or Docker Swarm

## Usage

### Quick deploy
1. Ensure Docker Swarm is formed
2. Update hostname constraints in `resources/docker-compose-swarm.default.yml` to that of the Swarm nodes.
3. Pull `sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:7.10.0` on each of the Swarm nodes
4. Run `build-stack.yml`
5. Access kibana using the elastic user in stack-passwords through the URL `https://<docker host ip>:5601`

### Docker Swarm
This will deploy a three node Elasticsearch Cluster with Kibana instance.
1. Ensure Docker Swarm has been initialised and the other nodes have been added if using a cluster.
    1. Run `docker swarm init` to initialise the Swarm
    2. Copy the output from `docker swarm join-token manager` run it on the other nodes to be joined to the Swarm.

2. Clone this repo to one of the Swarm Master nodes using `git clone https://github.com/jamesgarside/elastic-stack-arm.git`.
3. Change directory into the cloned repo `cd elastic-stack-arm`
4. Edit the deployment constraint hostnames for each service within `resources/docker-compose-swarm.default.yml` file to the hostnames of your cluster nodes. This ensures one Elasticsearch node runs on each Swarm node as well as ensures that persistent data is accessed by the correct Docker Service if the Service were to be restarted.
    - Example: `- "node.hostname==docker-1"`
5. Edit the Environment Vars within the `.env` file to your desired settings.
   - At current only the 'STACK_NAME' is used. This is the name your Elastic Stack with assume.
6. At this point it is advised to pull the Elasticsearch docker image down to each of the hosts you wish to deploy to. This ensures the script doesn’t hang while waiting for the image to download. This can be done by running the command `sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:7.10.0` on each of the hosts specified within the constraints on step 5. 

7. Run the build stack script `./build-stack.sh`
    This script does a few things:
    1. Sets Environment Variables from the within the .env file.
    2. Generates crypt for the Elastic Stack nodes
    3. Stores the crypt in Docker Secrets (Makes it available to all docker nodes as well as protects it)
    4. Deploys the Elastic Stack to the Swam using the initial config
    5. Generate the Elasticsearch passwords using Elasticsearches password init binary. These get stored in a file call `stack-passwords.txt`
    6. Creates a new `docker-compose.yml` with updated values unique to the deployment.
        - Sets the kibana_system password which was generated in step 5. 
        - Sets the cluster name based on the value in `.env`.
        - Sets a unique string for the Kibana SavedObjects encryption string. 
    7. Re-deploys/Updates the stack with the new docker-compose. 
8. Once the script has finished the stack is complete. Give Kibana a few minutes to start up as it requires the Elasticsearch cluster to have started before it can initialise itself.
9. Kibana can be accessed by the IP of any of the Docker Swarm hosts down to Docker Swarms mesh networking. `https://<docker host ip>:5601`.
    - Example: `https://192.168.0.10:5601`
11. To log in use the Elastic user generated by the `build-stack.sh` script which is stored in the `stack-passwords.txt` file. This file should be moved to somewhere safe.
    - Username: elastic
    - Password: *found in stack-passwords.txt*

#### Extra configuration
1. If desired the `docker-compose.yml` can be modified so that Kibana can be accessed over port 443 rather than 5601. This is done by changing the `ports:` config from `5601:5601` to `443:5601` and then running `sudo docker stack deploy -c docker-compose.yml`.

### Docker-Compose
To be completed. An early version of the Docker-Compose file is located within the `resources` folder. Crypt will need to be manually generated by using the `gen-certs.sh` script located in the gen_certs folder.

## Further Config
Further configuration can be carried out by using either Environment variables specified within the Service section of the Docker-Compose file or by bind mounting a Config file within the container. 

