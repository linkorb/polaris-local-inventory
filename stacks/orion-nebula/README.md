This stack, orion-nebula, is a production-quality stack in development.

So far, the application runs, but is missing configuration that would tell it where to find external services that it needs.  The app is configured to use the database service in the stack named "database-alpha".  For convenience, it uses the MariaDB root username and password.

 Once you have run the playbook and orion-nebula stack is deployed to the multipass VMs, you will want to create the db and schema.  These commands instruct swarm-manager to create instances of the Nebula container, configured in the same way as the "app" service, to execute `bin/console doctrine` commands:

 ```
 $  multipass exec swarm-manager -- sudo docker service create --name nebulous --detach --replicas 1 --mode replicated-job --restart-condition none --with-registry-auth --config source=dotenv_2025011406,target=/app/.env.local --network traefik ghcr.io/linkorb/nebula:latest bin/console --env=prod doctrine:database:create && multipass exec swarm-manager -- sudo docker service logs -ft nebulous

 Ctrl-C

 $  multipass exec swarm-manager -- sudo docker service rm nebulous

 $  multipass exec swarm-manager -- sudo docker service create --name nebulous --detach --replicas 1 --mode replicated-job --restart-condition none --with-registry-auth --config source=dotenv_2025011406,target=/app/.env.local --network traefik ghcr.io/linkorb/nebula:latest bin/console --env=prod doctrine:schema:create && multipass exec swarm-manager -- sudo docker service logs -ft nebulous

 Ctrl-C

 $ multipass exec swarm-manager -- sudo docker service rm nebulous
 ```
