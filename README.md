# Getting Download Hash
+++
title = "Getting Download Hash"
date = "FIXME"
tags = ["golang"]
categories = [ "golang" ]
url = "FIXME"
author = "mikit"
+++

One of the exercises I give my students is to download a big file in parallel.
An extra part is to verify the download from a known MD5 signature, this extra part turns out the be interesting, let's have a look.

## Getting Download Information

Let's make an HTTP HEAD request to get information about the downloaded file

```
01 $ curl -i -I https://storage.googleapis.com/gcp-public-data-landsat/LC08/01/044/034/LC08_L1GT_044034_20130330_20170310_01_T2/LC08_L1GT_044034_20130330_20170310_01_T2_B2.TIF
02 
03 HTTP/2 200 
04 x-guploader-uploadid: ADPycdtU_fX5RkUlYHRaU4ajofN7LOXIdjzNJUzKWyKKIOtIhxyhhyY-0JZ1avs5T1ohCZD7_0jPurQ2ByB3YlCm2D1D0A
05 expires: Thu, 02 Mar 2023 11:02:52 GMT
06 date: Thu, 02 Mar 2023 10:02:52 GMT
07 cache-control: public, max-age=3600
08 last-modified: Fri, 08 Sep 2017 09:10:25 GMT
09 etag: "eec1fa5ce8077d7030e194eb5989c937"
10 x-goog-generation: 1504861825662906
11 x-goog-metageneration: 1
12 x-goog-stored-content-encoding: identity
13 x-goog-stored-content-length: 69962928
14 content-type: application/octet-stream
15 x-goog-hash: crc32c=ir4fqg==
16 x-goog-hash: md5=7sH6XOgHfXAw4ZTrWYnJNw==
17 x-goog-storage-class: STANDARD
18 accept-ranges: bytes
19 content-length: 69962928
20 server: UploadServer
21 alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
```

On line 01 we use `curl` to make a HEAD request (`-I`), the `-i` switch tells curl to include the response headers.
The file is part of a public dataset stored on Google Cloud Storage (GCS) and as we can see on line 19, the size is 69962928 bytes (or about 66.7MB).
On lines 15 and 16 we see the `x-goog-hash` header that contains hash information.
The md5 hash, `md5=7sH6XOgHfXAw4ZTrWYnJNw==` looks strange.
The `md5=` is a prefix telling us this is an MD5 signature. The data after, `=7sH6XOgHfXAw4ZTrWYnJNw==` is the signature encoded in [base64 encoding](https://en.wikipedia.org/wiki/Base64).
Note that we have two `x-goog-hash` HTTP headers, another one for the [CRC check](https://en.wikipedia.org/wiki/Cyclic_redundancy_check).

### Actual File Sum

Let's check the file MD5 signature as reported by the `md5sum` utility.

```
01 $ curl https://storage.googleapis.com/gcp-public-data-landsat/LC08/01/044/034/LC08_L1GT_044034_20130330_20170310_01_T2/LC08_L1GT_044034_20130330_20170310_01_T2_B2.TIF | md5sum
02 eec1fa5ce8077d7030e194eb5989c937  -
```

On line 01 we use `curl` to download the file and pipe the output to the `md5sum` utility.
On line 02 we can see the signature: `eec1fa5ce8077d7030e194eb5989c937`.

Now that we have the data we need, let's start writing Go code to get a signature for a file hosted on GCS.

### Making a `HEAD` request

```go
12 func urlSig(ctx context.Context, url string) (string, error) {
13     req, err := http.NewRequestWithContext(ctx, http.MethodHead, url, nil)
14     if err != nil {
15         return "", err
16     }
17 
18     resp, err := http.DefaultClient.Do(req)
19     if err != nil {
20         return "", err
21     }
22     if resp.StatusCode != http.StatusOK {
23         return "", fmt.Errorf("%q: bad status - %s", url, resp.Status)
24     }
```

On line 12 we define the function `urlSig`, it get a context and a URL.
It's always a good idea to use context for timeouts when dealing with the network.
On line 13 we create a new `HEAD` request with the context and the URL, the last `nil` parameter means an empty request body.
On line 18 we use the default HTTP client to make the call and on line 22 we check that the response status code is `OK`.

### Finding the Signature

```go
26     // Find MD5 hash in HTTP headers.
27     const (
28         header = "x-goog-hash"
29         prefix = "md5="
30     )
31     b64hash := ""
32     values := resp.Header.Values(header)
33     for _, v := range values {
34         if strings.HasPrefix(v, prefix) {
35             b64hash = v[len(prefix):]
36             break
37         }
38     }
39 
40     if b64hash == "" {
41         return "", fmt.Errorf("can't find md5 hash %s: %v", header, values)
42     }
```

On lines 28 and 29 we define the header and the MD5 signature prefix.
On 32 we get a all the `x-goog-hash` headers from the HTTP response and on lines 33-38 we look for the MD5 signature.
On line 35 we remove the `md5=` prefix from the header value to get only the signature.
Finally on lines 40-42 we return an error if we can't find the MD5 signature.

### Decoding and Returning the Hash

```go
44     hash, err := base64.StdEncoding.DecodeString(b64hash)
45     if err != nil {
46         return "", err
47     }
48 
49     // Convert hash to "eec1fa5ce8077d7030e194eb5989c937" format.
50     return fmt.Sprintf("%x", hash), nil
51 }
```

On line 44 we use the standard encoder from the `encoding/base64` to decode the signature into a `[]byte`.
Finally, on line 50, we convert the signature from a `[]byte` to a string which has 2 hex values per byte.

### Testing the Code

```go
53 func main() {
54     url := "https://storage.googleapis.com/gcp-public-data-landsat/LC08/01/044/034/LC08_L1GT_044034_20130330_20170310_01_T2/LC08_L1GT_044034_20130330_20170310_01_T2_B2.TIF"
55     ctx, cancel := context.WithTimeout(context.Background(), time.Second)
56     defer cancel()
57 
58     fmt.Println(urlSig(ctx, url))
59 }
```

Here we write a `main` that calls `urlSig` with the URL we looked at before.

Now, let run it:

```
01 $ go run dlhash.go 
02 eec1fa5ce8077d7030e194eb5989c937 <nil>
```

On line 01 we run the code and on line 02 we see that the signature matches the one we got using the `md5sum` utility.


### Conclusion

Even small tasks such as verifying download integrity can lead to many interesting places.
We covered the following topics to make this happen:
- Making an HTTP head request
- HTTP headers
- Base64 encoding
- MD5 signatures
- Converting `[]byte` to a string

I hope you learned something useful from this blog post, feel free to reach me at miki@ardanlabs.com for comments and suggestions.
And of course, you are more than welcome to join one of out [world class trainings](https://www.ardanlabs.com/training/).
