# Improving Router security

(starting doc: https://connext.academy/routers/how-to-improve-router-security-by-p2p/)

**include a blurb about why security is important for the router operator and also the network as a whole, eg: the better the collective security...**

**include a blurb about Knowable's strategy, eg: we run our router assuming that our Router operator machine will be compromised"

***
## Unexpected Docker/UFW Interaction -- Securing the Router's API Endpoint

Please be aware that Docker will by default override UFW/iptables rules. If you're using the Docker Router, check that you're not accidentally exposing your Router API endpoint externally.

**Example:**  
After configuring UFW on a server running the Docker Router to block `ROUTER_EXTERNAL_PORT` (defined in the `.env` file -- `8000` by default), we expect the following to fail from an external machine:  

`curl http://x.x.x.x:8000/config`  

Instead, we receive a response indicating that the port is open.  

`{"signerAddress":"0x26Ad85....."}`

One simple solution to edit the `docker-compose.yml` file to bind the exposed port to **localhost**, by changing:

```
...
  ports:
    - $ROUTER_EXTERNAL_PORT:8080
...
```
to:
```
...
  ports:
    - 127.0.0.1:$ROUTER_EXTERNAL_PORT:8080
...
```

Restart Docker-Compose after making the change. The endpoint should now only be available from the machine running the Router -- try again from an external machine to make sure the change was successful.
***
## Admin Token Best Practices

***[To verify: is REST API still implemented in Amarok? Doc page seems to have been removed]***

Each Router has an **Admin Token**, which is a string chosen by the operator and set in its `config.json`.

The Admin Token is used to authenticate requests made to the Router's REST API endpoint and must be kept secret.

If your Router's API endpoint is exposed to the public (see section **Docker Port Mappings and UFW**) and your Admin Token is compromised or vulnerable to brute forcing, you could leave yourself at risk of unauthorized access.

Be sure to use a sufficiently long token (50 characters or more) to avoid the possibility of a brute force attack. You can easily generate a secure token using `pwgen` with the following command:

`pwgen -s 50 1`

#### Optional: Move Admin Token Out of plaintext

The Admin Token is stored in plaintext in the Router's `config.json` file, so for additional security you may wish to follow the steps below to load your **config.json** into `tmpfs` before starting the Router, and unmount it after the Router is started. Keep in mind that using this method will require you to generate and move your configuration into **tmpfs** on each Router restart, and the old contents will be lost -- so don't forget to backup any important information first (e.g. your wallet mnemonic, if using one!)

1. Create **tmpfs**:  
`mount -t tmpfs -o size=100m tmpfs /mnt/tmpfs`

2. Move config file to **tmpfs**:  
`mv config.json /mnt/tmpfs/config.json`

3. Change the volume point in docker-compose (use type [bind](https://github.com/docker/compose/issues/2781#issuecomment-441653347)):  
`- /mnt/tmpfs/config.json:/home/node/router/config.json`

4. Run `docker-compose`

5. Finally, unmount the **tmpfs** dir. After this step all data in **/mnt/tmpfs/** will be lost:  
`umount /mnt/tmpfs`


***
## Setting Recipient and Owner Addresses

In addition to the Router's own signing address, each Router is associated with a `Recipient` and `Owner` address. These are configured in the respective `Connext Handler` contracts of each chain.

- `Recipient`: Whenever liquidity is removed from the Router, the funds will be sent to this address. The rationale is that if an attacker were to compromise your Router, they would at best be able to withdraw your funds to the **Recipient** address (which you hopefully still control).

- `Owner`: Only the **Owner** has the ability to change the **Recipient** and **Owner** addresses.

The **Recipient** and **Owner** addresses can be changed by calling the corresponding write methods on the `Connext Handler` contract (`setRouterRecipient` and `proposeRouterOwner`/`acceptProposedRouterOwner` respectively) from the **Owner** address. This must be done on each chain supported.

It's **HIGHLY** recommended that you set your **Recipient** and **Owner** addresses to something different from your Router's signing address, and to use a hardware wallet. Since the **Recipient** and **Owner** private keys are not accessed by or stored on the Router, simply compromising your Router would not be enough for an attacker to access your funds -- they would need one or both of these keys also.

Please be aware that each Router's **Recipient** and **Owner** addresses can be publicly queried from the **Connext Handler** contract.


***
## Protecting Your Router's Private Key

**A little more research needed: any specific problems (eg: griefing, double spend) either for individual operator or network/users that result from a compromised Router colluding with a user? Or is it just best practices?**

Avoid operating your Router with your private key or mnemonic stored in plaintext. While it's possible to use a mnemonic in `config.json` or stored unencrypted in a `key.yml` file (as in `key.example.yml`), these should be considered for testing purposes only.

Instead, use one of the supported [Web3Signer methods](https://docs.web3signer.consensys.net/en/latest/HowTo/Use-Signing-Keys/). Using an **external KMS** that explicitly whitelists **Web3Signer** will allow you to move your private key out of plaintext and off your Router server entirely.

**Web3Signer** supports a variety of external key vaults and HSMs. Consult the [official docs](https://docs.web3signer.consensys.net/en/latest/Reference/Key-Configuration-Files/) to get started, but if you need help getting up and running with a specific method, please feel free to reach out on our Discord... chances are we can connect you with another operator who's already gone through the setup process!

**Potential contributor opportunity:  
Pick one specific Web3Signer method and write a guide on setting it up step by step for mission points. Ideally we could end up with guides on 3-4 different methods**

***
## Protect Your Router's IP Address

Consider using a setup that doesn't expose your Router IP Address to the public. For example, many cloud providers support features like virtual private networks, private instances and Cloud NATs. An example setup might look like this:  
```
  Router <-----> Cloud NAT <-----> Internet
    |
    |
Key Vault
```

**Further reading:**  
AWS docs: [Private Instances](https://aws.amazon.com/vpc/), [Cloud NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)  
GCP docs: [Private VPC](https://cloud.google.com/data-fusion/docs/how-to/create-private-ip), [Cloud NAT](https://cloud.google.com/nat/docs/overview)

**Potential contributor opportunity:  
Write a guide detailing a mainnet-ready cloud setup, and how/why, for points**
