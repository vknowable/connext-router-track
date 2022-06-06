# Connext Router track security advises

Here are some few security tips if you are planning to run a router on the connext

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

- `Recipient`: Whenever liquidity is removed from the Router, the funds will be sent to this address. The rationale is that if an attacker were to compromise your Router, they would at best be able to withdraw your funds to the **Recipient** address (which you hopefully still control). We advise that the recipent address would a hardware wallet.

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


Whenever you are joining a crypto project we advie that you should use a brand new wallet each time. 
You can generate a private key using the following command:
`openssl rand -hex 32 > private_key.json`


***
## Protect Your Router's IP Address


Consider using a setup that doesn't expose your Router IP Address to the public. For example, many cloud providers support features like virtual private networks, private instances and Cloud NATs. An example setup might look like this:  
```
  Router <-----> Cloud NAT <-----> Internet
    |
    |
Key Vault
```

To connect to your router host machine you can use a bastion server(jump server) to access it. A great tool for directly access your host machine is [teleport](https://goteleport.com/)

To have a deep dive understanding how private network works:
![ALT text](schema/nat.png)

As you can see in the digram above you can't access directly your router, only through the bastion server and then you can connect to the router host. Router is accesing the internet through NAT using it as gateway. 

Using this configuration is less possible for a attack to compromose your router host.


**Further reading:**  
AWS docs: [Private Instances](https://aws.amazon.com/vpc/), [Cloud NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)  
GCP docs: [Private VPC](https://cloud.google.com/data-fusion/docs/how-to/create-private-ip), [Cloud NAT](https://cloud.google.com/nat/docs/overview)


If you are connecting straigh from internet to your VM using ssh you should consider  the following options:
  - Use a private key to connect to the VM instead of the password
  - Change default ssh port from port 22 to any other port(for example 9922), they are a lot of bots scraping on the internet for port 22
  - Disable ssh root login
  - Restrict ssh access using iptables(ufw), be carefull you can locked out outside of your VM. You should only allow your host public ip address to connect to the router VM




