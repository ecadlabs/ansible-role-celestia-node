# celestia_node

A role to configure and deploy a [celestia](https://github.com/celestiaorg/celestia-node) node in a docker container.

> :warning: **This project is still a work in progress**: The role is still being worked on and developed. Please use with caution if you already have a node deployed or existing data that you do not want to lose.

## Role Variables
---

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `node_additional_options` | `[]` | A list of additional options and flags that can be set when initializing or starting a `celestia` node. An example would be `--log.level.module net/identify:ERROR`. |
| `node_container_name` | `celestia-{{ node_type }}-{{ p2p_network }}` | The name of the node docker container |
| `node_container_network_name` | `celestia-{{ p2p_network }}-network` | The name of the docker network that the container will use |
| `node_keyring_backend` | `test` | The keyring backend that will be used for the key that the node is started with. |
| `node_store_path` | `/srv/celestia-{{ node_type }}-{{ p2p_network }}` | The path to the desired celestia node directory on the system |
| `node_config` | `{{ node_store_path }}/config.toml` | The path to the desired celestia node configuration file on the system |
| `node_keyring_accname` | `""` | The account in the keyring for the node to start with |
| `node_type` | `light` | The celestia node type. This can be `full`, `light`, or `bridge` |
| `node_version` | `v0.8.1` | The version of the celestia-node docker image |
| `p2p_network` | `blockspacerace` | The celestia network that the node will operate on |
| `core_ip` | `https://rpc-2.celestia.nodes.guru` | Indicates the node to connect to the given core node. |
| `core_rpc_port` | `26657` | Set a custom RPC port for the core node connection. The --core.ip flag must also be provided. |
| `core_grpc_port` | `9090` | Set a custom gRPC port for the core node connection. The --core.ip flag must also be provided. |
| `node_gateway` | `true` | Whether or not the node should be started with the gateway enabled |
| `node_gateway_address` | `0.0.0.0` | Set a custom gateway listen address |
| `node_gateway_int_port` | `26659` | Set a custom gateway port |
| `node_gateway_ext_port` | `{{ node_gateway_int_port }}` | Set the gateway port exposed by the container |
| `log_level` | `INFO` | DEBUG, INFO, WARN, ERROR, DPANIC, PANIC, FATAL. This can be done for each module by adding the specific module and flag to the `node_additional_options` list |
| `node_rpc_address` | `0.0.0.0` | Set a custom node RPC listen address |
| `node_rpc_int_port` | `26658` | Set a custom node RPC port |
| `node_rpc_ext_port` | `{{ node_rpc_int_port }}` | Set the node RPC port exposed by the container |
| `node_metrics` | `false` | Whether or not the node should push metrics to a collector |
| `node_metrics_endpoint` | `""` | The endpoint for the metrics OTEL collector |
| `node_keys_source_dir` | `""` | If you would like to copy previously generated custom keys to the node. This should be the entire `keys` directory |

## Dependencies
---

* Debian based operating system
* Ansible 2.9 or higher
* A user with `sudo` access on the target host
* Docker Engine on the target host: 
  Install manually or use an ansible role like [geerlingguy.docker](https://galaxy.ansible.com/geerlingguy/docker)
  ```shell
  ansible-galaxy install geerlingguy.docker
  ```
* Python Docker module on the target host:
  Install manually with `python3 -m pip install docker` or use an ansible role like [geerlingguy.pip](https://galaxy.ansible.com/geerlingguy/pip)
  ```shell
  ansible-galaxy install geerlingguy.pip
  ```

## Quickstart
This quickstart guide will configure and run a `light` node on a debian based operating system.
1. Install the role using the `ansible-galaxy` command.
    ```shell
    ansible-galaxy install ecadlabs.celestia_node
    ```
2. Create a simple `inventory.yml` file
    ```yaml
    all:
      hosts:
        localhost:
          ansible_connection: local
    ```
3. Create a simple playbook called `celestia-light-node.yml`. We define three variables in the playbook, `node_type`, `p2p_network`, and `node_store_path`
    ```yaml
    - name: Celestia node
      hosts: localhost
      become: true

      tasks:
        - name: Configure and deploy Celestia light node
          ansible.builtin.include_role:
            name: ecadlabs.celestia_node

      vars:
        node_type: light
        p2p_network: blockspacerace
        node_store_path: /srv/celestia-light-blockspacerace
    ```

4. Run the playbook
    ```shell
    ansible-playbook ./celestia-light-node.yml --inventory ./inventory.yml --diff
    ```
5. Check that the container is running using `docker ps`. The container name should be `celestia-light-blockspacerace` and it should be running and not in a restart loop.
6. Check the container logs for more information and to make sure that the node is syncing.
    ```shell
    docker logs --follow --tail 100 celestia-light-blockspacerace
    ```
7. Test to make sure that you can reach the node gateway.
    ```shell
    curl localhost:26659/head
    ```
7. The playbook will create the required directories and docker volume mounts to persist the node data, keys, etc. Check that the files are available in `/srv/celestia-light-blockspacerace`. These files are owned by the user with ID 10001, you will need to use `sudo` to view the keys directory.

> :warning: There can sometimes be an issue when deploying locally where the python modules are not found. Ensure that the modules are installed for your user and that the correct `ansible_python_interpreter` has been set.
## Example Playbooks
---

In most cases the `init` and `start` containers will use the exact same variables and be started the same way. The main difference for the init container is the exclusion of the `node_config` variable. If the `node_config` variable is included here, the init will fail because the node will look for a config file that does not exist.

```yaml
- name: Celestia node
  hosts: servers
  become: true

  tasks:
    - name: Celestia full node
      ansible.builtin.include_role:
        name: ansible-role-celestia-node
  vars:
    node_keyring_backend: test
    node_keyring_accname: test_account
    node_type: full
    node_version: v0.8.1
    p2p_network: blockspacerace
    core_ip: https://rpc-2.celestia.nodes.guru
    node_gateway_address: 0.0.0.0
    node_gateway: true
    node_metrics: true
    node_metrics_endpoint: otel.celestia.tools:4318
    node_additional_options:
      - "--metrics.tls=false"
    node_keys_source_dir: /home/user/keys
```

In the example playbook we add an additional option `--metrics.tls=false` so that our metrics get pushed to an HTTP port and the node expects an HTTP response.

Because the keys directory is frequently backed up and used by custom keys we can also specify a source directory for these custom keys that will be copied to the target.

## License
---

Apache License 2.0

