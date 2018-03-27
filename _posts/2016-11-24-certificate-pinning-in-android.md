---
title: Certificate Pinning in Android
author: Marcos Placona
---
Certificate pinning is a security mechanism which allows HTTPS websites and applications using HTTPS services to resist impersonation by attackers using mis-issued or otherwise fraudulent certificates.

With certificate pinning it is possible to mitigate or severely reduce the effectiveness of [MiTM](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) attacks enabled by spoofing a back-end server’s SSL certificate.

If you are already using API's or services with [OkHTTP](http://square.github.io/okhttp/) or any library that uses it, such as [Retrofit](https://square.github.io/retrofit/) or [Picasso](http://square.github.io/picasso/) the good news is that you're already half way there.

Let's look at the following example from the [documentation](https://github.com/square/okhttp/wiki/Recipes#synchronous-get):

```java
private final OkHttpClient client = new OkHttpClient();
public void run(String url) throws Exception {
    Request request = new Request.Builder()
            .url(url)
            .build();

    client.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            Log.e(TAG, "onFailure: " + e.getMessage());
        }

        @Override
        public void onResponse(Call call, okhttp3.Response response) throws IOException {
            Log.d(TAG, "onResponse: " + response.body().string());

        }
    });
}
```

And you'd run it like this:

```
run("https://publicobject.com/helloworld.txt");
```

At this point you think: 

> Great! I'm going though HTTPS, so this must be safe right?

Well... No.

With enough knowledge about proxies (N.B. I have barely just enough), one can intercept that request and see every incoming and outgoing packages... As plain text!

![00.png](/images/00.png)

Now if we change our client slightly we can add certificate pinning to it, which will make it way harder for someone to impersonate our certificate and our app will refuse to make requests through that certificate.

```java
private CertificatePinner certificatePinner = new CertificatePinner.Builder()
    .add("publicobject.com", "sha256/0jQVmOH3u5mnMGhGRUCmMKELXOtO9q8i3xfrgq3SfzI")
    .build();

private final OkHttpClient client = new OkHttpClient
    .Builder()
    .certificatePinner(certificatePinner)
    .build();
````

And your code will error and log a message like this.

```sh
E/MainActivity: onFailure: Certificate pinning failure!
                  Peer certificate chain:
                    sha256/afwiKY3RxoMmLkuRW1l7QsPZTJPwDS2pdDROQjXw8ig=: CN=publicobject.com,OU=PositiveSSL,OU=Domain Control Validated
                    sha256/klO23nT2ehFDXCfx3eHTDRESMz3asj1muO+4aIdjiuY=: CN=COMODO RSA Domain Validation Secure Server CA,O=COMODO CA Limited,L=Salford,ST=Greater Manchester,C=GB
                    sha256/grX4Ta9HpZx6tSHkmCrvpApTQGo67CYDnvprLg5yRME=: CN=COMODO RSA Certification Authority,O=COMODO CA Limited,L=Salford,ST=Greater Manchester,C=GB
                  Pinned certificates for publicobject.com:
                    sha256/0jQVmOH3u5mnMGhGRUCmMKELXOtO9q8i3xfrgq3SfzI=
```

And this is **great**! Because it tells us which [SHA256](https://en.wikipedia.org/wiki/Secure_Hash_Algorithm) hash we should be using. So now we can change our code to use the keys described on the error as follows:

```java
private CertificatePinner certificatePinner = new CertificatePinner.Builder()
    .add("publicobject.com", "sha256/afwiKY3RxoMmLkuRW1l7QsPZTJPwDS2pdDROQjXw8ig=")
    .add("publicobject.com", "sha256/klO23nT2ehFDXCfx3eHTDRESMz3asj1muO+4aIdjiuY=")
    .add("publicobject.com", "sha256/grX4Ta9HpZx6tSHkmCrvpApTQGo67CYDnvprLg5yRME=")
    .build();
```

And sure enough, if you run your code again everything should work as expected
but with an added bonus. Our code is now checking that the certificate of the
host matches with the one we're expecting. And if it doesn't, it will refuse to make a request.

A proxy would then return a message saying: `No request was made. Possibly the SSL certificate was rejected.`.

![01.png](/images/01.png)

And now we've pinned our certificate to our code.
