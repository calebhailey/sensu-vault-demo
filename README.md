
## Prerequisites 

- Download & install Docker & `docker-compose` 
- Download & install the Sensu CLI (`sensuctl`) 
  _NOTE: we'll run the Sensu Go Server in a Docker container, but you'll need
  the `sensuctl` CLI to interact with Sensu._
- Download & install the Vault CLI (`vault`)
  _NOTE: we'll run the Vault server in a Docker container, but you'll need the
  `vault` CLI to interact with the Vault server._

## Demo 

1. Startup the demo environment 

   ```
   $ sudo docker-compose up -d -p webinar
   Creating network "webinar_default" with the default driver
   Creating volume "webinar_sensu-backend-data" with local driver
   Pulling sensu-backend (sensu/sensu:5.20.1)...
   5.20.1: Pulling from sensu/sensu
   486039affc0a: Pull complete
   d328af7475d0: Pull complete
   cb728683b14d: Pull complete
   6c0bd049f0a3: Pull complete
   183ed819b4f9: Pull complete
   2279a0c4f92f: Pull complete
   4c06b482ee4b: Pull complete
   Digest: sha256:f13625c8834dd293c8ee13b7f295266c4ec5e81bef58e2da2a6c156feb212c68
   Status: Downloaded newer image for sensu/sensu:5.20.1
   Creating webinar_vault_1         ... done
   Creating webinar_sensu-backend_1 ... done
   ```

2. Configure `sensuctl` to connect to the local backend

   ```   
   $ sensuctl configure -n --url http://127.0.0.1:8080 \
     --username sensu \
     --password sensu \
     --namespace default
   $ sensuctl cluster health
   === Etcd Cluster ID: 3b0efc7b379f89be
         ID           Name     Error   Healthy  
    ────────────────── ───────── ─────── ───────── 
     8927110dc66458af   default           true
   ```

3. Configure the `vault` CLI to connect to the Vault server

   ```
   export $(cat .env | grep VAULT_DEV_ROOT_TOKEN_ID)
   echo "${VAULT_DEV_ROOT_TOKEN_ID}" > $HOME/.vault-token
   export VAULT_ADDR=http://127.0.0.1:8200
   vault status
   vault audit enable file file_path=stdout
   
   Key             Value
   ---             -----
   Seal Type       shamir
   Initialized     true
   Sealed          false
   Total Shares    1
   Threshold       1
   Version         1.4.1
   Cluster Name    vault-cluster-468b591e
   Cluster ID      2d9aa556-a384-0b54-2523-346ae6e6d818
   HA Enabled      false
   ```

4. Create some Vault secrets!

   ```
   $ vault kv put secret/sensu/slack @slack-secrets.json 
   Key              Value
   ---              -----
   created_time     2020-05-18T22:47:17.765882188Z
   deletion_time    n/a
   destroyed        false
   version          1
   $ vault kv get secret/sensu/slack 
   ====== Metadata ======
   Key              Value
   ---              -----
   created_time     2020-05-18T22:47:17.765882188Z
   deletion_time    n/a
   destroyed        false
   version          1
   
   ======= Data =======
   Key            Value
   ---            -----
   channel        #demo
   ttl            30s
   webhook_url    https://hooks.slack.com/services/xxxxxxxxxx/xxxxxxxxxx/xxxxxxxxxxxxxxxxxxxx
   ```

4. Configure the Sensu Vault Provider & Slack handler 

   ```
   $ ll
   $ sensuctl create -f vault.yaml
   $ sensuctl create -f slack.yaml 
   $ sensuctl handler list 
   $ sensuctl secret list 
   ```

5. Create a event that should alert in Slack 

   ```
   $ sensuctl user create webinar --password $(uuid)
   $ sensuctl user add-group webinar cluster-admins
   $ sensuctl api-key grant webinar 
   $ export SENSU_API_KEY=
   $ curl -i -XPOST -H "Authorization: Key $SENSU_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"entity":{"metadata":{"name":"i-424242"}},"check":{"metadata":{"name":"my-app"},"status":2,"output":"ERROR: failed to connect to database.","handlers":["slack"]}}' \
     http://127.0.0.1:8080/api/core/v2/namespaces/default/events
   ```


## Troubleshooting 

- Reset the demo environment 

  ```
  $ sudo docker-compose down 
  $ sudo docker rmi sensu/sensu:5.20.1
  $ sudo docker volume rm webinar_sensu-backend-data
  ```

- Verify that the sensu-backend can reach the Vault API

  ```
  $ docker exec -it sensu-backend /bin/ash 
  # apk add curl 
  # curl -i http://vault:8200/v1/sys/health 
  ``` 

- Verify your VaultProvider config 

  ```
  $ sensuctl dump secrets/v1.Provider 
  ```
