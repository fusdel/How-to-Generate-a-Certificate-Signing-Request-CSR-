# How to Generate a Certificate Signing Request (CSR)

A step-by-step guide to creating a private key and a Certificate Signing Request (CSR) for an SSL/TLS certificate using OpenSSL.

## Table of Contents

- [What is a CSR?](#what-is-a-csr)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Method 1: Interactive (Single Domain)](#method-1-interactive-single-domain)
- [Method 2: One-Line Command](#method-2-one-line-command)
- [Method 3: With SAN / Multiple Domains (Recommended)](#method-3-with-san--multiple-domains-recommended)
- [Method 4: ECDSA Key Instead of RSA](#method-4-ecdsa-key-instead-of-rsa)
- [Verify Your CSR](#verify-your-csr)
- [Submit the CSR](#submit-the-csr)
- [Install the Certificate](#install-the-certificate)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

---

## What is a CSR?

A Certificate Signing Request (CSR) is a block of encoded text that you send to a Certificate Authority (CA) when applying for an SSL/TLS certificate. It contains:

- Your **public key**
- Details about your organization and domain (the *subject*)
- A signature made with your **private key**, proving you own the key pair

The private key is generated alongside the CSR and **never leaves your server**.

## Prerequisites

- OpenSSL installed
  - Linux/macOS: usually pre-installed. Check with `openssl version`
  - Windows: install via [Git for Windows](https://git-scm.com/download/win), WSL, or [OpenSSL for Windows](https://slproweb.com/products/Win32OpenSSL.html)
- Access to a terminal
- The domain name(s) you want to secure

## Quick Start

```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout example.com.key \
  -out example.com.csr \
  -subj "/C=US/ST=California/L=San Francisco/O=Example Inc/CN=example.com"
```

This creates:

| File | Purpose |
|------|---------|
| `example.com.key` | Your private key (**keep secret**) |
| `example.com.csr` | Your CSR (send this to the CA) |

---

## Method 1: Interactive (Single Domain)

**Step 1: Generate the private key**

```bash
openssl genrsa -out example.com.key 2048
```

> Use `4096` instead of `2048` for a stronger key (slightly slower).

**Step 2: Generate the CSR**

```bash
openssl req -new -key example.com.key -out example.com.csr
```

**Step 3: Answer the prompts**

| Field | Description | Example |
|-------|-------------|---------|
| Country Name (C) | 2-letter country code | `US` |
| State or Province (ST) | Full state/province name | `California` |
| Locality (L) | City | `San Francisco` |
| Organization (O) | Legal company name | `Example Inc` |
| Organizational Unit (OU) | Department (optional) | `IT` |
| Common Name (CN) | **Fully qualified domain name** | `www.example.com` |
| Email Address | Optional | `admin@example.com` |
| Challenge password | Leave blank | *(blank)* |

> **Important:** The Common Name must match the domain exactly. For a wildcard certificate, use `*.example.com`.

---

## Method 2: One-Line Command

Generate the key and CSR together without prompts:

```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout example.com.key \
  -out example.com.csr \
  -subj "/C=US/ST=California/L=San Francisco/O=Example Inc/OU=IT/CN=example.com"
```

- `-nodes` means the private key is not passphrase-protected. Remove it if you want to encrypt the key.

---

## Method 3: With SAN / Multiple Domains (Recommended)

Modern browsers rely on **Subject Alternative Names (SAN)** rather than the Common Name. Use this method to secure several domains with one certificate.

**Step 1: Create a config file** named `csr.conf`:

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[dn]
C  = US
ST = California
L  = San Francisco
O  = Example Inc
OU = IT
CN = example.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = example.com
DNS.2 = www.example.com
DNS.3 = api.example.com
```

**Step 2: Generate the key and CSR**

```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout example.com.key \
  -out example.com.csr \
  -config csr.conf
```

---

## Method 4: ECDSA Key Instead of RSA

ECDSA keys are smaller and faster, and are widely supported.

```bash
# Generate an EC private key (P-256 curve)
openssl ecparam -genkey -name prime256v1 -out example.com.key

# Generate the CSR
openssl req -new -key example.com.key -out example.com.csr \
  -subj "/C=US/ST=California/L=San Francisco/O=Example Inc/CN=example.com"
```

> Confirm your CA supports ECDSA before using this method.

---

## Verify Your CSR

Check the contents before submitting:

```bash
openssl req -text -noout -verify -in example.com.csr
```

Check that the subject and SANs are correct, and look for `verify OK` in the output.

Confirm the CSR and private key match (the hashes must be identical):

```bash
openssl req -noout -modulus -in example.com.csr | openssl md5
openssl rsa -noout -modulus -in example.com.key | openssl md5
```

---

## Submit the CSR

1. Display the CSR contents:

   ```bash
   cat example.com.csr
   ```

2. Copy everything, including the header and footer lines:

   ```
   -----BEGIN CERTIFICATE REQUEST-----
   MIIC...
   -----END CERTIFICATE REQUEST-----
   ```

3. Paste it into your Certificate Authority's order form (DigiCert, Sectigo, GoDaddy, etc.).
4. Complete domain validation (DNS, email, or HTTP).
5. Download the issued certificate and any intermediate/chain certificates.

---

## Install the Certificate

Once issued, combine the certificate with the chain if required by your server.

**Nginx**

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.fullchain.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
}
```

**Apache**

```apache
<VirtualHost *:443>
    ServerName example.com
    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt
</VirtualHost>
```

Create a full-chain file:

```bash
cat example.com.crt intermediate.crt > example.com.fullchain.crt
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `openssl: command not found` | Install OpenSSL (`sudo apt install openssl` or `brew install openssl`) |
| CA rejects the CSR | Check that the CN/SAN matches your domain and that the key is at least 2048-bit |
| Key and certificate don't match | Compare the modulus hashes (see [Verify Your CSR](#verify-your-csr)) |
| Special characters rejected | Avoid `, / \ ! @ #` in organization names |
| Need to change details | You must generate a new CSR; the subject cannot be edited |

---

## Security Best Practices

- **Never share your private key**, only the CSR goes to the CA.
- Set strict permissions: `chmod 600 example.com.key`
- **Do not commit `.key` files to GitHub.** Add this to your `.gitignore`:

  ```gitignore
  *.key
  *.pem
  *.p12
  *.pfx
  ```
- Back up the private key in a secure location (password manager or encrypted vault).
- Use at least 2048-bit RSA (or ECDSA P-256).
- Generate a new key pair when renewing, rather than reusing the old one.

---

## References

- [OpenSSL `req` documentation](https://www.openssl.org/docs/manmaster/man1/openssl-req.html)
- [RFC 2986: PKCS #10 Certificate Request Syntax](https://datatracker.ietf.org/doc/html/rfc2986)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)

## License

This guide is released under the MIT License. Feel free to use and adapt it.
