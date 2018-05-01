## Motivation

During the course of operating services and infrastructure, we often find ourselves handling small but sensitive datums such as API keys, username/passwords, security tokens and the like. For example, configuring a database connection from an upstream service to a backend database usually involves three pieces of sensitive data: the connection string, the username and password. 

There exist myriad mechanisms to communicate these small data items from the control plane to the underlying VM or container: instance metadata, K8S configmaps, environment variables and so on.

However these mechanisms do not offer any guarantees around integrity/confidentiality of data. It is therefore up to the operator. 

## Scenario

Building upon the example cited above, this post will explore one possible mechanism to achieve our goal, using only native GCP services. We will present the following approach:

- Construction of a plain-text secrets file
- Creation of KMS keyring/keys
- Encryption of plain-text secrets file using KMS key
- Storage of encrypted secrets in GCS
- Decryption of secrets in shell script and Python

Note: Alternative approaches could be to use GCE instance metadata (assuming the consumer is running in context of GCE), and K8S secrets if running in containerized context. The GCS approach described here works across the range of GCP compute services (including App Engine).

## Prerequisites

The examples that follows makes use of the gcloud SDK. Install this by following the guidance here.

## Setup
### Initialize gcloud configuration

It is a good practice when working with GCP via the command line, especially when using multiple identities, to configure unique configuration profiles for each project you are working on. This allows rapid switching between user identities and projects.

To create a new configuration, simply run  `gcloud init`, select 'Create a new config' and follow the prompts. It is a good idea to have the project ID handy.

View all configured profiles:

```bash
gcloud config configurations list
```

Activate a particular profile:

```bash
gcloud config configurations activate <name>
```

I recommend creating shell aliases for these since you will be using them a lot. For example, for bash/zsh:

```bash
alias gccl='gcloud config configurations list'
alias gcca='gcloud config configurations activate'
```
### Configure environment

```bash
export KEYRING=kms-secrets-demo-keyring
export KEY=kms-secrets-demo-key
export SVCACCT=<NAME>@developer.gserviceaccount.com
export SECRETS_FILE=<FILENAME>.txt
export SECRETS_BUCKET=gs://<BUCKET>
```

### Create keyring

```bash
gcloud kms keyrings create $(KEYRING) --location=global
```

### Create key
```bash
gcloud kms keys create $(KEY) --keyring=$(KEYRING) \
                              --location=global \
                              --purpose=encryption
```

### Create IAM policy binding

```bash
gcloud kms keyrings add-iam-policy-binding $(KEYRING) \
						--location=global \
						--member serviceAccount:$(SVCACCT) \
						--role roles/cloudkms.cryptoKeyDecrypter
```

## Usage
### Create secrets file

Using your editor of choice, build a temporary file containing the secrets. The format of this file is fleixible depending on how you will eventually decrypting these secrets. For example, if you will be decrypting using the gcloud SDK from within a shell script then a good format could be:

```
user.name:name
user.passwd:password
```

### Encrypt secrets file

```bash
gcloud kms encrypt --ciphertext-file=$(SECRETS_FILE).txt.enc \
                   --plaintext-file=$(SECRETS_FILE).txt \
                   --keyring=$(KEYRING) \
                   --key=$(KEY) \
                   --location=global
```

### Delete plaintext

This step is important - to avoid mistakenly distributing the secrets file in plaintext, it is highly encouraged to immediately and permanently delete the plaintext file.

### Upload secrets file to GCS

```bash
gsutil cp $(SECRETS_FILE).txt.enc $(SECRETS_BUCKET)
```

### Decryption

Now that we have an encypted secrets file residing on GCS, we can configure our startup script to stream the secrets and decrypt. We will make use of a couple of handy features to acheive this goal without the use of temporary files:

- The `gsutil cat` command which streams a GCS object to STDOUT
- The ability of `gcloud kms decrypt` to accept input from STDIN and produce output to STDOUT

In order to avoid repeated download and decryption steps for each secret, we will first stream and decrypt the secrets to an environment variable (TODO: max size?)
 
```bash
SECRETS=$(gsutil cat ${SECRETS_FILE} gcloud kms decrypt \
               --keyring=${KEYRING} \
               --key=${KEY} \
               --location=global \
               --ciphertext-file - \
               --plaintext-file -)
```

Now that we have the secrets, it is simply a case of filtering for the property key and extracting the correct field value:

```bash
USERNAME=$(echo $SECRETS \
         | grep ^user.name \
         | cut -d: -f2)
```

We now have the decrypted secret as an environment variable, with no intermediate files to cleanup. This step should be repeated for each unique property required. 
