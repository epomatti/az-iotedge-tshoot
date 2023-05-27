# Azure IoT Edge Troubleshooting

### 1 - Generate the certificate chain:

```sh
bash scripts/generateCerts.sh
```

### 2 - Create the infrastructure:

```sh
terraform -chdir="infra" init
terraform -chdir="infra" apply -auto-approve
```

### 3 - (Optional) Migrate the IoT Hub to use DigiCert G2 root:

```sh
az iot hub certificate root-authority set --hub-name "iot-bluefactory" --certificate-authority v2 --yes
```

### 4 - Device check

Make sure the EdgeGateway has completed the installation:

```sh
# Connect to the IoT Edge VM
ssh edgegateway@<public-ip>

# Check if the cloud-init status is "done", otherwise wait with "--wait"
cloud-init status

# Confirm that the IoT Edge runtime has been installed
iotedge --version

# Restart the VM to enable any Linux kernel updates
az vm restart -n "vm-bluefactory-edgegateway" -g "rg-bluefactory"
```

### 5 - Register the IoT Edge device:

> ⚠️ IoT Hub allows only IoT Edge devices with self-signed (thumbprint) method. For CA-Signed, you [must use Device Provisioning Service](https://github.com/MicrosoftDocs/azure-docs/issues/108363).

```sh
bash bash scripts/registerEdgeGatewayDevice.sh
```

### 6 - 