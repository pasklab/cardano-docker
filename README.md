# cardano-docker
Docker files for setting up Cardano Node environment on Raspberry Pi (arm64/aarch64)

#### Reference

Many steps used in this repository are from resources bellow:

Thanks to CoinCashew for providing a great guide.

[CoinCashew Guide: How to build a Haskell Testnet Cardano Stakepool](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node)

Thanks to everyone behind CNODE from **cardano-community repository**.

[https://github.com/cardano-community/guild-operators](https://github.com/cardano-community/guild-operators)

### Building from source 

Since we need our binary to work on Aarch64 architecture, you'll need to build the node from the source files.
I've wrote a Dockerfile that simplify the process.

First, you need to build all required images:
  
1. The Cardano sources image:
        
        docker build \
            -t cardano_env:latest \
            ./Dockerfiles/build_env

    ** Tip: _Add `--no-cache` to rebuild from scratch_ **
        
2. Set the version variable (Set the right release VERSION_NUMBER, ie: `1.14.0`)

        VERSION_NUMBER=<VERSION_NUMBER>

3. The node image:

        docker build \
            --build-arg RELEASE=${VERSION_NUMBER} \
            -t cardano_node:${VERSION_NUMBER} Dockerfiles/node
        
4. The cli image:

        docker build \
            --build-arg RELEASE=${VERSION_NUMBER} \
            -t cardano_cli:${VERSION_NUMBER} Dockerfiles/cli
        
5. Tag your images with the **latest** tag:

        docker tag cardano_node:${VERSION_NUMBER} cardano_node:latest
        docker tag cardano_cli:${VERSION_NUMBER} cardano_cli:latest
                                     
### Node configuration

Now you've created yours images, it's time to create your `config` folder. Your container will bind to this folder,
so you can access your configuration from within.

    mkdir config
    cd config
        
If your OS is unix based, you can use the `wget` utility to download all configuration files from the
[official source](https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/index.html).

    wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/shelley_testnet-config.json
    wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/shelley_testnet-shelley-genesis.json
    wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/shelley_testnet-topology.json

#### !!! Important !!!!
Once you have the files, rename them to `genesis.json`, `topology.json` and `config.json` to avoid breaking the script every times they
change the name..! Don't forget to update the reference to the `genesis.json` file in your `config.json`.
        
Now, if you wish to use the `start-relay.sh` script provided in my repository, add a `port.txt` file under your `/config` 
folder. Your file should contain only one line representing the **PORT** used by your node. 
    
    echo 3000 > port.txt
    
#### Activating LiveView

If you want to use the LiveView interface, you can update the `ViewMode` and `TraceBlockFetchDecisions` in your 
`ff-config.json` file by running the following command:

    sed -i.bak -e "s/SimpleView/LiveView/g" -e "s/TraceBlockFetchDecisions\": false/TraceBlockFetchDecisions\": true/g" ff-config.json
    
#### Relay node configuration

Now you need to configure your ff-topology.json file with your Relay and Producer node information.

See: [Configure the block-producer node and the relay nodes](https://www.coincashew.com/coins/overview-ada/guide-how-to-build-a-haskell-stakepool-node#3-1-configure-the-block-producer-node-and-the-relay-nodes)

### Monitoring

Since it's nice to monitor your setup, follow the Grafana/Prometheus installation step bellow:

1. Build the cardano_monitor image:

        docker build \
            -t cardano_monitor:latest \
            ./Dockerfiles/monitor
            
2. Build the grafana image:

        docker build \
            -t grafana:latest \
            ./Dockerfiles/grafana

### Monitoring tools configuration \*\*OPTIONAL\*\*

Now you've created yours monitoring images, it's time to create your `monitoring-config` folder.
Your `cardano_monitor` container will bind to this folder, so you can access your configuration from within.

    mkdir monitoring-config
    cd monitoring-config

You can now copy the following config file in this folder:

- [./Dockerfiles/monitor/files/grafana.ini](Dockerfiles/grafana/files/grafana.ini)

#### Creating the Grafana web server

Since you don't need to run a Grafana web server on every **cardano-node** host, it's container creation isn't included
in the `docker-compose.yml` file. Use the following command to create its container in a location of your choosing.

Go in the same folder where you've created your `monitoring-config` folder and run the following command:

    docker run -dit \
        --network host \
        --mount source=grafana-data,target=/root/grafana/data \
        --mount type=bind,source="$(pwd)"/monitoring-config,target=/root/config \
        --name grafana grafana:latest

** Tips: Just remove the `bind mount` line if you're good with the provided configuration file.

### Creating the containers with Docker Compose

You can copy the docker-compose.yaml where your `config/` folder reside. Then start your containers with the 
following command:

    docker-compose up -d

** Tips: To start a node as producer instead of relay, swap the comment on `CMD` line in the `docker-compose.yaml` file.

### Read further on these topics:

- [How get peers with Topology Updater](Docs/topology.md)
- [Manually creating the containers](Docs/standalone-containers.md)
