---
title: "Self-hosted Bluesky PDS on Kubernetes"
date: "2024-12-01"
tags: ["kubernetes", "pds", "bluesky", "atproto", "self-host"]
showTags: true
draft: false
---

## Intro

A few months ago, I created a simple Helm chart to deploy a Bluesky PDS on Kubernetes. I wanted to extend that a bit and show how it can be deployed.

## What's a PDS?

A PDS or Personal Data Server is a component of the AT Protocol network. It's where your user data resides on the AT Protocol.
Due to the open and federated nature of the AT Protocol, it's possible to self-host a PDS on your own server and use Bluesky with it.

## Requirements

Before deploying our PDS, we will need the following :
  * Domain name
  * SMTP Relay service. (e.g. [resend](https://resend.com/) works well.)
    * Not mandatory to use the PDS, but you'll need it to verify user accounts.
  * Kubernetes cluster with :
    * [Ingress NGINX controller](https://github.com/kubernetes/ingress-nginx) (Should™️ also work with a different Ingress Controller)
    * [cert-manager](https://cert-manager.io/) with [HTTP01](https://cert-manager.io/docs/configuration/acme/http01/) and [DNS01](https://cert-manager.io/docs/configuration/acme/dns01/) challenge solvers configured
    * Long-term storage
  * [Bluesky PDS Helm chart](https://github.com/Nerkho/helm-charts/tree/main/charts/bluesky-pds)

Note that I purely focus on the PDS deployment. The elements above will not be covered in details. For this experiment, I have deployed Ingress NGINX with a default configuration.
The DNS01 challenge solver with cert-manager might be tricky depending on your registrar.

## Let's deploy our PDS

If you take a look at the [default values](https://github.com/Nerkho/helm-charts/blob/main/charts/bluesky-pds/values.yaml), you will notice there are sensitive values that need to be passed to the chart. As we are good kids, we will pre-create a secret with those values and reference it with the `existingSecret`.

```yaml
...
pds:
  config:
    hostname: "pds.lab.nerkho.ch"
    secrets:
      # -- Output of `openssl rand --hex 16`
      jwtSecret: ""
      # -- Output of `openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32`
      adminPassword: ""
      # -- Output of `openssl rand --hex 16`
      plcRotationKey: ""
      # -- Example: `smtps://user:password@smtp.example.com:465/`
      emailSmtpUrl: ""
      # -- Set the value for existingSecret to use a pre-created secret
      #existingSecret : ""
...
```

We generate each value as indicated (_save those somewhere safe_) :

```bash
openssl rand --hex 16
<jwtSecret>

openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32
<plcRotationKey>

openssl rand --hex 16
<adminPassword>
```

And we create the secret :

```bash
kubectl create secret generic --namespace app-bluesky bluesky-secret \
    --from-literal=adminPassword=<adminPassword> \
    --from-literal=jwtSecret=<jwtSecret> \
    --from-literal=plcRotationKey=<plcRotationKey>
    --from-literal=emailSmtpUrl=smtps://resend:<your api key here>@smtp.resend.com:465/
```

Now, we can reference that secrets in our value file.

```yaml
...
pds:
  config:
    hostname: "pds.lab.nerkho.ch"
    secrets:
      existingSecret : "bluesky-secret"
...
```

Additionally, we need to set the `pdsEmailFromAddress` value. This is the email address that will be used by the PDS for verification and such.

```yaml
pds:
  config:
    hostname: "pds.lab.nerkho.ch"
    secrets:
      existingSecret: bluesky-secret
    pdsEmailFromAddress: "pds@nerkho.ch"
```

For the storage, we probably want to define our `storageClass`. I'm using [Longhorn](https://longhorn.io/), so I'll set it here.

```yaml
...
pds:
  config:
    hostname: "pds.lab.nerkho.ch"
    secrets:
      existingSecret : "bluesky-secret"
    dataStorage:
        storageClass: "longhorn"
...
```

We aim to add 2 hosts. One for the PDS and a wildcard for each user. This is required as on the AT Protocol, each user is kind of its own website with its own subdomain.

We also need to add the wildcard domain to our TLS hosts. As mentioned earlier, this will require a [DNS01](https://cert-manager.io/docs/configuration/acme/dns01/) challenge solvers to be configured on cert-manager.

If this is not possible in your scenario, you could just list each subdomain for each user of your PDS instead of using a wildcard.

```yaml
    ingress:
      enabled: true
      className: "nginx"
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        kubernetes.io/tls-acme: "true"
      hosts:
        - host: pds.lab.nerkho.ch
          paths:
            - path: /
              pathType: Prefix
        - host: "*.pds.lab.nerkho.ch"
          paths:
            - path: /
              pathType: Prefix
      tls:
        - secretName: bluesky-pds-tls
          hosts:
            - pds.lab.nerkho.ch
            - "*.pds.lab.nerkho.ch"
```

With this, we can deploy the PDS. I, personally, use Flux CD on my cluster, but straight `helm install ...` will work fine as well. If everything goes as planned, you should be able to reach your PDS with `curl` and see something like this :

```bash
curl https://pds.lab.nerkho.ch/
This is an AT Protocol Personal Data Server (PDS): https://github.com/bluesky-social/atproto

Most API routes are under /xrpc/⏎
```

We did not cover it, but I also recommend you to configure the `podSecurityContext` and `securityContext`. Commenting out the default value should work just fine.

## Create a user and start using Bluesky

We can now create our first user. The first step is to create an invite code.

Note: The [pdsadmin scripts](https://github.com/bluesky-social/pds/tree/main/pdsadmin) expect that you are on the same machine as your PDS. I chose to run the commands myself. Another option is to use [this pdsadmin go binary](https://github.com/lhaig/pdsadmin) from [@lhaig.haigmail.com](https://bsky.app/profile/lhaig.haigmail.com)

```bash
export PDS_ADMIN_PASSWORD=<adminPassword>
export PDS_HOSTNAME=pds.lab.nerkho.ch

curl --silent --show-error --request POST --header "Content-Type: application/json" "$@" \
    --user "admin:${PDS_ADMIN_PASSWORD}" \
    --data '{"useCount": 1}' \
    "https://${PDS_HOSTNAME}/xrpc/com.atproto.server.createInviteCode" | jq --raw-output '.code'

<invite-code>
```

With the invite code, we can now create the user.
We need to create an input JSON file with the user data.

```json
{
  "email": "<username@example.com>",
  "handle": "<username>.pds.lab.nerkho.ch",
  "password": "<user-password>",
  "inviteCode": "<invite-code>"
}
```

With this, we can create the user. If this works fine, you will get JSON output indicating the user has been successfully created.

```bash
curl --silent --show-error --request POST --header "Content-Type: application/json" \
    -d @input.json \
    "https://${PDS_HOSTNAME}/xrpc/com.atproto.server.createAccount" | jq
```

At this point, we can now login on https://bsky.app with our new user. We need to specify the URL of our PDS in the first field and login with the defined above.

![bluesky-login](/img/bluesky-pds-login.png)

Et voilà! You are now using Bluesky with a self-hosted PDS. If you'd prefer to use a different domain than your PDS for your handle, you can do so and [set a custom domain as your handle](https://blueskyweb.zendesk.com/hc/en-us/articles/19001802873101-How-to-Set-your-Domain-as-your-Handle).

### Troubleshooting invalid handle warning

If your account shows a warning with `invalid handle` you can debug that with the [Bluesky Handle Debug Page](https://bsky-debug.app/handle).  This should help to find where the issue lies.

![bluesky-login](/img/bluesky-pds-handle-debug.png)

You don't need both method DNS and HTTP. Just one suffice.

If it's green there, often times, just updating the handle in the Account settings help.

## To go further

Currently, I only use my PDS for experimenting. If you plan to go productive with it, you will probably want to look into monitoring.

For this, I recommend taking a look at [this video](https://www.youtube.com/watch?v=7-VJvf39xVE) from [@justingarrison.com](https://bsky.app/profile/justingarrison.com) where he shows how to run a PDS on an RPI and also touches about monitoring.
