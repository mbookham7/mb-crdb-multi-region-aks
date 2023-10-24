# Creating Wildcard Certificate for the DBColsole

Install `certbot` using brew.

```
brew install certbot
```

Using `certbot` we send a request to LetEncrypt to generate a certificate. We have asked to use DNS in validating ownership of the requested domain name.

```
sudo certbot certonly --manual --preferred-challenges dns -d "mikebookham.co.uk,*.mikebookham.co.uk"
```

As a result of this you will be asked to create two TXT records. This need to be done via the provider that hosts the domain.

```
Please deploy a DNS TXT record under the name:

_acme-challenge.mikebookham.co.uk.

with the following value:

K0NhOSajk0LgNUWp2enjQ1Ucvx1UC-YbKYtvccQ8Xi8

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.mikebookham.co.uk.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/mikebookham.co.uk/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/mikebookham.co.uk/privkey.pem
This certificate expires on 2023-02-14.
These files will be updated when the certificate renews.

NEXT STEPS:
- This certificate will not be renewed automatically. Autorenewal of --manual certificates requires the use of an authentication hook script (--manual-auth-hook) but one was not provided. To renew this certificate, repeat this same certbot command before the certificate's expiry date.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

To update the cert you will need to update the secret containing the files for the certificate.

To update the secret you will delete it and recreate it with the new files.
```
kubectl delete secret cockroachdb.client.root --context $clus1 --namespace $loc1
kubectl delete secret cockroachdb.client.root --context $clus2 --namespace $loc2
kubectl delete secret cockroachdb.client.root --context $clus3 --namespace $loc3
```

Recreate the secret in each region.
```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```
```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus2 \
--namespace $loc2
```
```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus3 \
--namespace $loc3
```

Rolling restart
```
kubectl rollout restart sts/cockroachdb --context $clus1 \
--namespace $loc1

kubectl rollout restart sts/cockroachdb --context $clus2 \
--namespace $loc2

kubectl rollout restart sts/cockroachdb --context $clus3 \
--namespace $loc3
```

