---
title: IDOR
headline: Insecure Direct Object References
description: Fundamentals of IDOR vulnerabilities and prevention methods.
date: 2022-11-16T18:14:00.415Z
author: nouman
draft_img: /img/drafts/pexels-pixabay-270408.jpg
tags:
  - webapps
mermaid: false
---
Insecure direct object reference (IDOR) vulnerabilities frequently place on the OWASP Top 10. They are  fundamentally a broken access control issue and as such are classifed by OWASP under _Broken Access Control_ vulnerabilities. 

In this blog post, we will explore the premise behind IDOR vulnerabilities and discuss some mitigation strategies.

## What are IDOR Vulnerabilities?

Insecure direct object reference is a type of access control vulnerability that results when a web application uses a unique identifier for direct access to an internal object without additional access control or authentication checks.

Exposed direct object references are not necessarily a security risk by themselves; however when combined with improper or insufficient access control, direct object references enable possibility of enumeration attacks which could lead to circumvention of poor access control measures. 

An IDOR vulnerability arises when the following conditions are satisfied:

1. An application exposes a direct reference to an internal object or operation.
2. A client can modify the object reference through manipulation of the URL via GET and POST parameters (URL Tampering) and/or form parameters (Body Manipulation).
3. The application grants client access to the internal object without an authorization check.

## Basis of an IDOR Vulnerability Attack

Consider the following website with an access control vulnerability:

```
https://www.example.com/account?id=001
```

In this case, the `id` parameter is used as an index in the _id table_ in the database (ie. exposed direct object reference). When passed an appropriate value, the web application queries the internal database and returns relevant information to the client.

Since the `id` parameter is used directly to query the database, an attacker can enumerate valid parameters to gain access to information associated with any other `id`. In our example, passing parameter `id=002` will result in the web application returning relevant information associated with that specific database index. 

As a result of this IDOR vulnerability, an attacker can perform horizontal privilege escalation attacks against the web application thereby accessing information they normally should not have access to. If the attacker is able to enumerate an `id`  associated with elevated privileges then a vertical privilege escalation is also possible.

It is worth mentioning that the most common type of IDOR vulnerabilities are associated with directory or path traversal which can permit an attacker to manipulate file system resources by allowing uploading or downloading of files and gain access to sensitive resources such as `/etc/passwd`.

## Preventing IDOR Vulnerabilities

The best way to protect against possible IDOR vulnerabilities is through implementation of strict authorization and access control to validate and manage user access. Syntactic validation can also be employed to identify suspicious user inputs before the input is processed by the webserver.

Additionally, indirect reference mapping can be employed to avoid exposing direct object references. This allows substitution of direct object references with difficult to guess indirect references. The mapping between indirect references and direct references could then be maintained on a secure server. Alternatively, salted hashes can be used to replace direct object references. For more information, refer to [Insecure Direct Object Reference Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) on OWASP cheat Sheet Series on IDOR.
