---
layout: layout.pug
navigationTitle:  Unsealing the Secret Store
title: Unsealing the Secret Store
menuWeight: 3
excerpt:

enterprise: true
---

# About unsealing the Secret Store

The Secret Store can become sealed under the following circumstances.

- [After being manually sealed.](/1.9/security/ent/secrets/seal-store/)
- After a power outage.

A sealed Secret Store cannot be accessed from the GUI. Secret values cannot be retrieved using the [Secrets API](/1.9/security/ent/secrets/secrets-api/). Services that depend on values provisioned to them via environment variables may fail to deploy.

<!--
The procedure for unsealing the Secret Store differs according to the keys used to seal it.

- [Unsealing a Secret Store sealed with default keys](#unseal-def-keys).

- [Unsealing a Secret Store sealed with custom keys](#unseal-cust-keys).
-->

**Prerequisites:**

- [DC/OS CLI installed](/1.9/cli/install/)
- Logged into the DC/OS CLI as a superuser via `dcos auth login`
- If your [security mode](/1.9/security/ent/#security-modes) is `permissive` or `strict`, you must [get the root cert](/1.9/networking/tls-ssl/get-cert/) before issuing the curl commands in this section.  If your [security mode](/1.9/security/ent/#security-modes) is `disabled`, you must delete `--cacert dcos-ca.crt` from the commands before issuing them.

**Note:** In these procedures, we will use two terminal prompt tabs: one to SSH into the master and use GPG; another to execute curl requests and use xxd. The master does not have xxd installed by default at this time. Nor does it have a package manager. If you do not wish to shuttle between terminal prompt tabs, you can run xxd inside a container on the master.

# <a name="unseal-def-keys"></a>Unsealing a Secret Store sealed with default keys

1. From a terminal prompt, check the status of the Secret Store via the following command.

   ```bash
   curl --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" $(dcos config show core.dcos_url)/secrets/v1/seal-status/default
   ```

1. The Secret Store service should return the following response.

   ```json
   {"sealed":true,"threshold":1,"shares":1,"progress":0}
   ```

   If the value of `"sealed"` is `false`, do not complete the rest of this procedure. Your Secret Store is not sealed, so you cannot unseal it.

1. After confirming that your Secret Store is indeed sealed, open a new terminal prompt tab.

1. From the new tab, [SSH into your master](/1.9/administering-clusters/sshcluster/) and launch the ZooKeeper command line interface as follows.

   ```bash
   /opt/mesosphere/packages/exhibitor--*/usr/zookeeper/bin/zkCli.sh
   ```

1. Execute the following ZooKeeper command to gain additional privileges, specifying the user name and password of the ZooKeeper superuser. By default, this is set to `super:secret` but we recommend [changing the default](/1.9/installing/ent/custom/configuration/configuration-parameters/#zk-superuser).

   ```bash
   addauth digest super:secret
   ```

1. Retrieve the default private GPG key using the following command.

   ```bash
   get /dcos/secrets/keys/bootstrap_user.key
   ```

1. Select the first value returned, everything in between the quote marks, and copy it to your clipboard.

1. Type `quit` to exit the ZooKeeper command line.

1. Decode the private GPG key using the following command.

   ```bash
   echo <base64-encoded-gpg-key> | base64 -d
   ```

1. This will return the decoded private GPG key, which should look as follows.

   ```
   -----BEGIN PGP PRIVATE KEY BLOCK-----
   xcZYBFfr8jEBEACoG/RL2hGhwoUYRpWue4nTZYQYna1Hbm0TaPYWjiek/ScXwgIt
   ...
   =Xc0I
   -----END PGP PRIVATE KEY BLOCK-----
   ```

1. Select everything in between and including `-----BEGIN PGP PRIVATE KEY BLOCK` and `END PGP PRIVATE KEY BLOCK-----`. Copy it to your clipboard and paste it into a new file giving it a name such as `gpg-private.key`.

1. Load the decoded GPG key into GPG as follows.

   ```bash
   gpg --allow-secret-key-import --import gpg-private.key
   ```

1. Delete the file.

   ```bash
   rm -rf gpg-private.key
   ```

1. Switch back to the original terminal prompt tab.

1. Use the `init` endpoint of the Secrets API to retrieve the encrypted unseal key as shown in the curl below.

    ```bash
    curl --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" $(dcos config show core.dcos_url)/secrets/v1/init/default
    ```

1. This command should return a JSON object similar to the following.

    ```json
    {"initialized":true,"keys":["c1c..."],"pgp_fingerprints":["524c98..."],"root_token":"147de72..."}
    ```

1. Copy the value of `"keys"` to your clipboard. This is your encrypted unseal key in ASCII format.

1. Transform the encrypted unseal key into binary and save the result into a new file using the following command. Before executing the command, replace `c1c04c...d00` with the value of your encrypted unseal key.

   ```bash
   echo "c1c04c...d00" | xxd -r -p > binary-unseal.key
   ```

1. Use secure copy to transfer the new file to your master, as shown below. Replace `<cluster-IP>` below with the IP address of your cluster. You can locate this value in the top left of the DC/OS dashboard.

   ``` bash
   scp binary-unseal.key core@<cluster-IP>:~
   ```

1. Return to your secure shell terminal prompt tab.

1. Confirm that the `binary-unseal.key` file copied over successfully using the following command.

   ```bash
   ls -la
   ```

1. Use the following command to decrypt the unseal key with GPG.

   ```bash
   gpg -d binary-unseal.key
   ```

1. This should return the decrypted unseal key value. Copy this value to your clipboard.

1. Return to the original terminal prompt tab.

1. Use the following curl command to unseal the store. Before executing this command, replace `c9e...33` with the decrypted unseal key value.

    ```bash
    curl -X PUT --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" -d '{"key":"c9e...33"}' $(dcos config show core.dcos_url)/secrets/v1/unseal/default -H 'Content-Type: application/json'
    ```

1. The Secret Store service should return the following JSON response, indicating success.

   ```json
   {"sealed":false,"threshold":1,"shares":1,"progress":0}
   ```

<!--
# <a name="unseal-cust-keys"></a>Unsealing a Secret Store sealed with custom keys

1. From a terminal prompt, check the status of the Secret Store via the following command.

   ```bash
   curl --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" $(dcos config show core.dcos_url)/secrets/v1/seal-status/default
   ```

1. Check the response to make sure it corresponds to the following.

   ```json
   {"sealed":true,"threshold":1,"shares":1,"progress":0}
   ```

   If the value of `"sealed"` is `false`, do not complete the rest of this procedure. Your Secret Store is not sealed, so you cannot unseal it.

1. Use the `init` endpoint of the Secrets API to retrieve the encrypted unseal key as shown in the curl below.

    ```bash
    curl --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" $(dcos config show core.dcos_url)/secrets/v1/init/default
    ```

1. This command should return a JSON object similar to the following.

    ```json
    {"initialized":true,"keys":["c1c...0700"],"pgp_fingerprints":["9b25...622b"],"root_token":"3fd...a3d"}
    ```

1. Copy the value of `"keys"` to your clipboard. This is your encrypted unseal key in ASCII format.

1. Transform the encrypted unseal key into binary and save the result into a new file using the following command. Before executing the command, replace `c1c...0700` with the value of your encrypted unseal key.

   ```bash
   echo "c1c...0700" | xxd -r -p > binary-unseal.key
   ```

1. Use secure copy to transfer the new file to your master, as shown below. Before executing the command, replace `<master-IP>` with the IP address of the master.

   ``` bash
   scp binary-unseal.key core@<master-IP>:~
   ```

   **Tip:** If you used GPG to generate the custom GPG keypair as described in [Reinitializing the Secret Store with a custom GPG keypair](/1.9/security/secrets/custom-key/) and you have multiple masters, use the IP address of the master that you used to generate the keypair.

1. [SSH into your master](/1.9/administering-clusters/sshcluster/)

1. Confirm that the `binary-unseal.key` file copied over successfully using the following command.

   ```bash
   ls
   ```

1. If your GPG private key is not already loaded into GPG, go ahead and load it now.

   ```bash
   gpg --allow-secret-key-import --import gpg.key
   ```

   **Tip:** If you recently completed the [Reinitializing the Secret Store with your own GPG key](/1.9/security/secrets/custom-key/) procedure, your private key should already be loaded in GPG.

1. Use the following command to decrypt the unseal key with GPG.

   ```bash
   gpg -d binary-unseal.key
   ```

1. GPG should return the decrypted unseal key value. Copy this value to your clipboard.

1. Switch to another terminal prompt tab.

1. Use the following curl command to unseal the store, replacing `bd3e...78c` with the decrypted unseal key value.

    ```bash
    curl -X PUT --cacert dcos-ca.crt -H "Authorization: token=$(dcos config show core.dcos_acs_token)" -d '{"key":"bd3e...78c"}' $(dcos config show core.dcos_url)/secrets/v1/unseal/default -H 'Content-Type: application/json'
    ```

1. It should return the following JSON response, indicating success.

   ```json
   {"sealed":false,"threshold":1,"shares":1,"progress":0}
   ```
-->
