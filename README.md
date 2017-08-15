# Private TLS Cert

This repository contains a [Terraform](https://www.terraform.io/) module that can be used to generate self-signed TLS
certificate. To be more accurate, the module generates the following:

* A Certificate Authority (CA) public key 
* The public and private keys of a TLS certificate signed by the CA

This TLS certificate is meant to be used with **private** services, such as a web service used only within your 
company. For publicly-accessible services, especially services you access through a web browser, you should NOT use 
this module, and instead get certificates from a commercial Certificate Authority, such as 
[Let's Encrypt](https://letsencrypt.org/).

If you're unfamiliar with how TLS certificates work, check out the [Background section](#background).




## Quick start

*Note: This Terraform module uses bash commands in a `local-exec` provisioner to copy the generate certificates to
files, so currently, it will only work on Linux, Unix, and OS X.*

1. `git clone` this repo to your computer and go into the `modules/generate-cert` folder.

1. Open `vars.tf` and fill in the variables that do not have a default.

1. DO NOT configure Terraform remote state storage for this code. You do NOT want to store the state files as they 
   will contain the private keys for the certificates.

1. Run `terraform init`.

1. Run `terraform apply`. The output will show you the paths to the generated files:

    ```
    Outputs:
    
    ca_public_key_file_path = ca.key.pem
    private_key_file_path = my-app.key.pem
    public_key_file_path = my-app.crt.pem
    ```
    
1. Delete your local Terraform state:

    ```
    rm -rf terraform.tfstate*
    ```

   The Terraform state will contain the private keys for the certificates, so it's important to clean it up!

You can now use the TLS certs with your applications! To inspect a certificate, you can use OpenSSL:

```
openssl x509 -inform pem -noout -text -in my-app.crt.pem
```




## Background


### How TLS/SSL Works

The industry-standard way to add encryption for data in motion is to use TLS (the successor to SSL). There are many 
examples online explaining how TLS works, but here are the basics:

- Some entity decides to be a "Certificate Authority" ("CA") meaning it will issue TLS certificates to websites or 
  other services

- An entity becomes a Certificate Authority by creating a public/private key pair and publishing the public portion 
  (typically known as the "CA Cert"). The private key is kept under the tightest possible security since anyone who 
  possesses it could issue TLS certificates as if they were this Certificate Authority!

- In fact, the consequences of a CA's private key being compromised are so disastrous that CA's typically create an 
  "intermediate" CA keypair with their "root" CA key, and only issue TLS certificates with the intermediate key.

- Your client (e.g. a web browser) can decide to trust this newly created Certificate Authority by including its CA 
  Cert (the CA's public key) when making an outbound request to a service that uses the TLS certificate.

- When CAs issue a TLS certificate ("TLS cert") to a service, they again create a public/private keypair, but this time 
  the public key is "signed" by the CA. That public key is what you view when you click on the lock icon in a web 
  browser and what a service "advertises" to any clients such as web browsers to declare who it is. When we say that 
  the CA signed a public key, we mean that, cryptographically, any possessor of the CA Cert can validate that this same 
  CA issued this particular public key.

- The public key is more generally known as the TLS cert.

- The private key created by the CA must be kept secret by the service since the possessor of the private key can 
  "prove" they are whoever the TLS cert (public key) claims to be as part of the TLS protocol.

- How does that "proof" work? Well, your web browser will attempt to validate the TLS cert in two ways:
  - First, it will ensure this public key (TLS cert) is in fact signed by a CA it trusts.
  - Second, using the TLS protocol, your browser will encrypt a message with the public key (TLS cert) that only the
    possessor of the corresponding private key can decrypt. In this manner, your browser will be able to come up with a
    symmetric encryption key it can use to encrypt all traffic for just that one web session.

- Now your client/browser has:
  - declared which CA it will trust
  - verified that the service it's connecting to possesses a certificate issued by a CA it trusts
  - used that service's public key (TLS cert) to establish a secure session


### Commercial or Public Certificate Authorities

For public services like banks, healthcare, and the like, it makes sense to use a "Commercial CA" like Verisign, Thawte,
or Digicert, or better yet a widely trusted but free service like [Let's Encrypt](https://letsencrypt.org/). That's 
because every web browser comes pre-configured with a set of CA's that it trusts. This means the client connecting to 
the bank doesn't have to know anything about CA's at all. Instead, their web browser is configured to trust the CA that 
happened to issue the bank's certificate.

Connecting securely to private services is similar to connecting to your bank's website over TLS, with one primary 
difference: **We want total control over the CA.**

Imagine if we used a commercial CA to issue our private TLS certificate and that commercial or public CA--which we 
don't control--were compromised. Now the attackers of that commercial or public CA could impersonate our private server. 
And indeed, [it](https://www.theguardian.com/technology/2011/sep/05/diginotar-certificate-hack-cyberwar) [has](
https://www.schneier.com/blog/archives/2012/02/verisign_hacked.html) [happened](
http://www.infoworld.com/article/2623707/hacking/the-real-security-issue-behind-the-comodo-hack.html)
multiple times.


### How We'll Generate a TLS Cert for Private Services

One option is to be very selective about choosing a commercial CA, but to what benefit? What we want instead is 
assurance that our private service really was launched by people we trust. Those same people--let's call them our 
"operators"--can become their *own* CA and generate their *own* TLS certificate for the private service.

Sure, no one else in the world will trust this CA, but we don't care because we only need our organization to trust 
this CA.

So here's our strategy for issuing a TLS Cert for a private service:

1. **Create our own CA.**
  - If a client wishes to trust our CA, they need only reference this CA public key.
  - We'll deal with the private key in a moment.

1. **Using our CA, issue a TLS Certificate for our private service.**
  - Create a public/private key pair for the private service, and have the CA sign the public key.
  - This means anyone who trusts the CA will trust that the possessor of the private key that corresponds to this public 
    key is who they claim to be.
  - We will be extremely careful with the TLS private key since anyone who obtains it can impersonate our private 
    service! For this reason, we recommend immediately encrypting the private key with 
    [KMS](https://aws.amazon.com/kms/).

1. **Freely advertise our CA's public key to all internal services.**
  - Any service that wishes to connect securely to our private service will need our CA's public key so it can declare 
    that it trusts this CA, and thereby the TLS cert it issued to the private service.

1. **Throw away the CA private key.**
    - By erasing a CA private key it's impossible for the CA to be compromised, because there's no private key to steal!
    - Future certs can be generated with a new CA.




## License

This code is released under the MIT License. See [LICENSE.txt](/LICENSE.txt).