# Azure IoT Edge Troubleshooting

Troubleshooting IoT Edge connectivity and configuration.

<img src=".assets/tshoot.png" width=550 />

## Infrastructure

### 1 - Create the resources

Generate the test-only certificate chain:

```sh
bash scripts/generateCerts.sh
```

Deploy the resources:

```sh
terraform -chdir="infra" init
terraform -chdir="infra" apply -auto-approve
```

<details>
  <summary>(Optional) Upgrade IoT Hub certificate to V2 (DigiCert)</summary>

  ```sh
  az iot hub certificate root-authority set --hub-name "iot-bluefactory" --certificate-authority v2 --yes
  ```
</details>

Make sure the EdgeGateway has completed the installation:

```sh
# Connect to the IoT Edge VM
ssh edgegateway@<public-ip>

# Check if the cloud-init status is "done", otherwise wait with "--wait"
cloud-init status

# Confirm that the IoT Edge runtime has been installed
iotedge --version
```

Restart the VM to activate any Linux kernel updates:

```sh
az vm restart -g "rg-bluefactory" -n "vm-bluefactory-edgegateway"
```

### 2 - Register the IoT Edge device

Run the script to create the IoT Hub device registration:

> ⚠️ IoT Hub supports registering IoT Edge devices only through self-signed method (certificate thumbprint). For a CA-Signed configuration, you must implement device enrollment with DPS. See [this issue](https://github.com/MicrosoftDocs/azure-docs/issues/108363) for details.

```sh
bash scripts/registerEdgeGatewayDevice.sh
```

Upload the required configuration files to the EdgeGateway device:

```
bash scripts/uploadEdgeConfig.sh
```

Connect with SSH to the EdgeGateway and execute the configuration

```sh
sudo bash edgeConfig.sh
```

Verify the results:

```sh
sudo iotedge system status
sudo iotedge system logs
sudo iotedge check
```

### 3 - Deploy the modules

Now that the device is properly registered and connected with IoT Hub, create a deployment:

```sh
az iot edge deployment create --deployment-id "gateway" \
    --hub-name $(jq -r .iothub_name infra/output.json) \
    --content "@edgegateway/deployments/gateway.json" \
    --labels '{"Release":"001"}' \
    --target-condition "deviceId='EdgeGateway'" \
    --priority 10
```

To check the deployment in the EdgeGateway device:

```sh
iotedge list
```

Check and confirm that everything is OK:

```sh
sudo iotedge check
```

All connectivity checks should be:

<img src=".assets/connectivity1.png" width=600 />

## Troubleshooting

### Restrict Public Access

Connect to the Portal and disable public access.

Do not yet apply to the built-in endpoint. Run the check:

```sh
# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

Now apply turn-on the toggle to apply to the built-in endpoint.

<img src=".assets/all_blocked.png" width=600 />

Run the check:

```sh
# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

This test demonstrates that even when blocking all traffic, including the built-in endpoint, the check tool will give it OK ✅ for all checks.

<img src=".assets/connectivity1.png" width=600 />

After restarting the VM or IoT Edge, one error was shown in the configuration:

<img src=".assets/error1.png" width=600 />

### Remove Root Certificate

Let's try removing the Root certificate in this example:

```sh
# Get the "etag"
etag=$(az iot hub certificate list --hub-name iot-bluefactory --query value[0].[etag] -o tsv)

# Delete the certificate
az iot hub certificate delete --hub-name iot-bluefactory --name TerraformRootCA --etag $etag
```

Restart and run the check:

```sh
# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

It was possible to demonstrate that no impact occurred when removing the main certificate:

<img src=".assets/connectivity1.png" width=600 />

This is because the connectivity check will be limited to TCP, the application layer is not verified.

Re-create the certificate:

```sh
# Create the certificate
az iot hub certificate create --hub-name iot-bluefactory --name TerraformRootCA --path ./openssl/certs/azure-iot-test-only.root.ca.cert.pem --verified

# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

### Remove the Device

Let's try and remove the Device Registration from IoT Hub.

```sh
# Delete the device
az iot hub device-identity delete --hub-name iot-bluefactory --device-id EdgeGateway

# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

In this scenario, the check tool will present only one error, while connectivity remains ok:

<img src=".assets/error2.png" width=600 />

Again, because connectivity checks only check TCP, this part is OK.

<img src=".assets/connectivity1.png" width=600 />

To re-register the device:

```sh
# Register the device
bash scripts/registerEdgeGatewayDevice.sh

# Restart
sudo iotedge system restart

# Check
sudo iotedge check
```

Previous deployments have to be applied again:

```sh
# Create the deployment
az iot edge deployment create --deployment-id "gateway-reapply" \
    --hub-name $(jq -r .iothub_name infra/output.json) \
    --content "@edgegateway/deployments/gateway.json" \
    --labels '{"Release":"001"}' \
    --target-condition "deviceId='EdgeGateway'" \
    --priority 10

# List the containers
iotedge list
```

### Firewall ports block

This test will demonstrate the results when outgoing connectivity is blocked in the Firewall.

Delete the IoT Edge outbound allow rule in the NSG:

```sh
# Delete the rule
az network nsg rule delete -g rg-bluefactory --nsg-name nsg-bluefactory-edgegateway -n AllowIoTEdge

# Check
sudo iotedge check
```

It is possible to identify several failures for this test.

> ℹ️ One aspect of this test is that the test takes a lot of time due to timeout for each connection

<img src=".assets/nsg_error1.png" width=600 />

<img src=".assets/nsg_error2.png" width=600 />

<img src=".assets/nsg_error3.png" width=600 />

<img src=".assets/nsg_error4.png" width=600 />

Running the checks with `--verbose` will reveal the timeout details:

<img src=".assets/nsg_error_verbose.png" width=800 />

Running telnet will achieve the same result:

<img src=".assets/telnet_timeout.png" width=500 />

Recreate the rule to proceed with the remaining tests:

```sh
# Create the IoT Edge rule
az network nsg rule create -g rg-bluefactory --nsg-name nsg-bluefactory-edgegateway -n AllowIoTEdge \
    --priority 110 \
    --direction "Outbound" \
    --source-address-prefixes '*' \
    --source-port-ranges '*' \
    --destination-address-prefixes '*' \
    --destination-port-ranges 5671 8883 443 \
    --access "Allow" \
    --protocol "Tcp" \
    --description "Allows IoT Edge connectivity ports."

# Check
sudo iotedge check
```

### Origin restriction

Now let's change the origin parameters to DENY outbound access for the IoT required ports:

```sh
# Create a DENY rule with priority 101
az network nsg rule create -g rg-bluefactory --nsg-name nsg-bluefactory-edgegateway -n DenyIoTEdge \
    --priority 101 \
    --direction "Outbound" \
    --source-address-prefixes '*' \
    --source-port-ranges '*' \
    --destination-address-prefixes '*' \
    --destination-port-ranges 5671 8883 443 \
    --access "Deny" \
    --protocol "Tcp" \
    --description "Deny rule to test connectivity behavior."

# Check
sudo iotedge check
```

Connections will fail with timeout:

<img src=".assets/timeout2.png" width=600 />

Rollback the change, remove the DENY rule:

```sh
# Remove the DENY rule
az network nsg rule delete -g rg-bluefactory --nsg-name nsg-bluefactory-edgegateway -n DenyIoTEdge

# Check
sudo iotedge check
```

## IoT Edge Diagnostics - Identity (aziot-identity-service)

The connectivity check for the identity service runs on top of `aziotctl` binary.

In the code [`mode.rs`](https://github.com/Azure/iotedge/blob/main/edgelet/iotedge/src/check/mod.rs) we can locate the command:

```rust
let aziot_check_out = tokio::process::Command::new(aziot_bin)
    .arg("check-list")
    .arg("--output=json")
    .output()
    .await;
```

The command runs like this:

```sh
aziotctl check-list
```

Here is a TLS example on a MQTT port using `openssl`:

```sh
openssl s_client -connect iot-bluefactory.azure-devices.net:8883 -brief
```

This particular code has it's own repository: https://github.com/Azure/iot-identity-service

## IoT Edge Diagnostics - Connectivity Checks

Here I'll document how `iotedge check` works when doing connectivity checks.

As explained in the [Diagnose IoT Edge devices](https://learn.microsoft.com/en-us/azure/iot-edge/troubleshoot?view=iotedge-1.4) page, the CLI makes use of the container `mcr.microsoft.com/azureiotedge-diagnostics:latest`.

You can verify that by listing the docker images in your IoT Edge device:

```
$ sudo docker image ls
REPOSITORY                                       TAG       IMAGE ID       CREATED        SIZE
mcr.microsoft.com/azureiotedge-diagnostics       1.4.10    c23f7741d317   45 hours ago   91.8MB
mcr.microsoft.com/azureiotedge-agent             1.4       793f1cf53dbe   45 hours ago   179MB
mcr.microsoft.com/azureiotedge-hub               1.4       d3ef8bfe20f8   45 hours ago   181MB
marketplace.azurecr.io/garantiadata/redis-edge   latest    d6f69e2e2942   4 years ago    125MB
```

What happens under the hood is that this block of Rust code will run the Diagnostics program.

Sample from [`container_connect_upstream.rs`](https://github.com/Azure/iotedge/blob/main/edgelet/iotedge/src/check/checks/container_connect_upstream.rs):

```rust
self.diagnostics_image_name = Some(check.diagnostics_image_name.clone());
args.extend([
    &diagnostics_image_name,
    "dotnet",
    "IotedgeDiagnosticsDotnet.dll",
    "upstream",
    "--hostname",
    upstream_hostname,
    "--port",
    &port,
]);
```

The library handling the diagnostic connectivity is written in C#.

If the request is not HTTPS (`port != "443"`), then a `TcpClient` is created:

```csharp
TcpClient client = new TcpClient();
await client.ConnectAsync(hostname, int.Parse(port));
client.GetStream();
```

You can build and run it directly (.NET Core 6.0 is required):

```sh
# Clone and cd into the diagnostics code
cd edge-modules/iotedge-diagnostics-dotnet

# Build the project
dotnet publish

# Run the program
dotnet bin/Debug/net6.0/IotedgeDiagnosticsDotnet.dll upstream --hostname iot-bluefactory.azure-devices.net --port 5671
```

For HTTPS requests or more details, check the class [`Program.cs`](https://github.com/Azure/iotedge/blob/main/edge-modules/iotedge-diagnostics-dotnet/src/Program.cs).

## Utilities

- [Several IoT Edge utilities](https://sandervandevelde.wordpress.com/category/docker/)
- [IoT ELMS](https://github.com/Azure-Samples/iotedge-logging-and-monitoring-solution)
- [IoT Edge runtime host network mode](https://github.com/Azure/iotedge/issues/5935)
- [Azure IoT Identity Service](https://azure.github.io/iot-identity-service/)
