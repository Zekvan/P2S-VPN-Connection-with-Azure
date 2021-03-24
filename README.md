# Design and Implementation of Point to Site (P2S) VPN gateway connection in Linux

A Point-to-Site (P2S) VPN gateway connection lets you create a secure connection to your virtual network from an individual client computer. A P2S connection is established by starting it from the client computer. This solution is useful for telecommuters who want to connect to Azure VNets from a remote location, such as from home or a conference. P2S VPN is also a useful solution to use instead of S2S VPN when you have only a few clients that need to connect to a VNet. 

## Installation
### Create a virtual network and a virtual network gateway

#####  1. Create a VNet using the following values:

* Resource group: TestRG1
* Name: VNet1
* Region: (EU) East UE
* IPv4 address space: 10.1.0.0/16
* Subnet name: FrontEnd
* Subnet address space: 10.1.0.0/24

#####  2. Create a virtual network gateway using the following values:

* Name: VNet1GW
* Region: East US
* Gateway type: VPN
* VPN type: Route-based
* SKU: VpnGw1 ( ###### Cheaper)
* Generation: Generation1
* Virtual network: VNet1
* Gateway subnet address range: 10.1.255.0/27
* Public IP address: Create new
* Public IP address name: VNet1GWpip
* Enable active-active mode: Disabled
* Configure BGP: Disabled

### Generate certificates

Install strongSwan

```bash
sudo apt install strongswan
sudo apt install strongswan-pki
sudo apt install libstrongswan-extra-plugins
```

Generate the CA certificate

```bash
ipsec pki --gen --outform pem > caKey.pem
ipsec pki --self --in caKey.pem --dn "CN=VPN CA" --ca --outform pem > caCert.pem
```

Print the CA certificate in base64 format. This is the format that is supported by Azure. You upload this certificate to Azure as part of the P2S configuration steps.


```bash
openssl x509 -in caCert.pem -outform der | base64 -w0 ; echo
```

Generate the user certificate

```bash
export PASSWORD="password"
export USERNAME="client"

ipsec pki --gen --outform pem > "${USERNAME}Key.pem"
ipsec pki --pub --in "${USERNAME}Key.pem" | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "CN=${USERNAME}" --san "${USERNAME}" --flag clientAuth --outform pem > "${USERNAME}Cert.pem"
```



Generate a p12 bundle containing the user certificate. This bundle will be used in the next steps when working with the client configuration files.

```bash
openssl pkcs12 -in "${USERNAME}Cert.pem" -inkey "${USERNAME}Key.pem" -certfile caCert.pem -export -out "${USERNAME}.p12" -password "pass:${PASSWORD}"
```

### Configure OpenVPN clients for Azure VPN Gateway

Open a new Terminal session. Enter the following command to install needed components:

```bash
sudo apt-get install openvpn
sudo apt-get -y install network-manager-openvpn
sudo service network-manager restart
```

Download the VPN profile for the gateway. This can be done from the Point-to-site configuration tab in the Azure portal	

Open ${USERNAME}Cert.pem in a text editor. To get the thumbprint of the client (child) certificate, select the text including and between "_-----BEGIN CERTIFICATE-----_" and "_-----END CERTIFICATE-----_" for the child certificate and copy it.

Open the vpnconfig.ovpn file and find the section shown below. Replace everything between "_<cert>_" and "_</cert>_".

```bash
# P2S client certificate
# please fill this field with a PEM formatted cert
<cert>
$CLIENTCERTIFICATE
</cert>
```

Open the ${USERNAME}Key.pem in a text editor. To get the private key, select the text including and between "-----BEGIN PRIVATE KEY-----" and "-----END PRIVATE KEY-----" and copy it.

Open the vpnconfig.ovpn file in a text editor and find this section. Paste the private key replacing everything between and "_<key>_" and "_</key>_".



```bash
# P2S client root certificate private key
# please fill this field with a PEM formatted key
<key>
$PRIVATEKEY
</key>
```

Click to the network icon on the right-top-corner of screen .

Click + to add a new VPN connection.

Under Add VPN, pick Import from file…

Browse to the profile file and double-click or pick Open.

Click Add on the Add VPN window.

Finally, You can connect by turning the VPN ON on the Network Settings page, or under the network icon.
