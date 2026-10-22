# Corridor Write-up

## Enumeration

I started by identifying the target IP address:

```text
10.130.187.137
```

I then accessed the web server:

```text
http://10.130.187.137
```

The website loaded successfully, but there was no useful information displayed on the page.

## Source Code Analysis

Since the page did not contain anything interesting, I inspected the HTML source code.

In the source code, I found links to multiple pages using a hash as part of the URL. The URL format was:

```text
http://10.130.187.137/<hash>
```

I analyzed the hashes and identified them as **MD5 hashes**. After decoding them, I noticed that they corresponded to numbers from **1 to 13**.

This suggested that the application was using the MD5 hash of a number as the page identifier.

## Finding the Flag

Based on the discovered pattern, I tested the MD5 hash of `0`.

The MD5 hash of `0` is:

```text
cfcd208495d565ef66e7dff9f98764da
```

I then accessed:

```text
http://10.130.187.137/cfcd208495d565ef66e7dff9f98764da
```

This page contained the flag:

```text
flag{2477ef02448ad9156661ac40a6b8862e}
```

## Conclusion

The flag was obtained by analyzing the website's source code, identifying the MD5-based URL pattern, and testing the missing value `0`.

**Flag:**

```text
flag{2477ef02448ad9156661ac40a6b8862e}
```