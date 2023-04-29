---
layout: post
title:  When Trust Fails - Unauthenticated RCE via API calls from CLI Binary
categories: [Research,API Security]
---

In one of my recent 3rd party security assessments the objective was to perform a security review an application that is hosted internally in the clients network. The application is a file system data platform that specializes in high-performance, scalable storage solutions for enterprise applications. This application is used by organizations in a variety of industries, including finance, healthcare, and research, to store and analyze large amounts of data.


## The Data Platform 

The web application used for managing this data platform provides a graphical user interface (GUI) for administrators to monitor and manage the file system. This could include features such as creating and managing file shares, setting up storage policies, and configuring performance and security settings. The web application also provide reporting and analytics tools to help administrators track usage, identify trends, and troubleshoot issues. 

The CLI binary used for managing the data platform provide a command-line interface for administrators to perform similar tasks as the web application, but through text-based commands. This could include creating and managing file shares, setting up storage policies, and configuring performance and security settings. 

## Accessing the Application

The application is hosted on an isolated network and can only be accessed through a bastion jump host after connecting to the client's network via VPN. For  web app and api testing, I've set up an SSH socks proxy connection to the bastion host and configured Burpsuite to forward traffic through the proxy. This allowed traffic from the local machine to be forwarded through a secure and encrypted channel to the bastion host. The bastion host functioned as an intermediary between the local machine and the data platform, enabling the secure relay of intercepted API calls through the bastion host to ultimately reach the data platform.

## Overview of the Security Vulnerability

The CLI binary subcommands require authentication in order to execute commands to the data platform. Users can authenticate to the data platform via CLI by running the `login` subcommand followed by username and password. Once successfully authenticated, users can run any commands as long as they have the appropriate privileges. When users execute commands using the CLI binary, the binary creates an HTTPS request over its declared port to the data platform. For example, the following subcommand `user whoami` submits HTTPS request to `/api/v1/[redacted]` with the user’s JWT token attached to the authorization header.

The security concern in this case is located within the `local` subcommand of the CLI binary. This administrative subcommand controls the data platform on the local server and is intended for users with root-level privileges on the data platform server. Users without root-level privileges are not authorized to execute any operation using the `local` subcommand.

When root-level users execute the `local` subcommand, the CLI binary submits an HTTPS request to a special `/[RootOnlyRedacted]` endpoint without any form of authentication which us different from the other subcommand that sent request to `/api/v1/[redacted]` endpoint. The data platform assumes that  users submitting requests to the  `/[RootOnlyRedacted]` endpoint are root-level users because the authorization check is assumed to be performed on the data platform server host.

So how do we exploit this if the `locale` subcommand is only available for root level user and its only available via CLI? Well, I've already jumped the gun and mentioned that the `/[RootOnlyRedacted]` endpoint accepts request without any form of authentication, this means anyone could impersonate a root-level user and make API calls to that endpoint via curl to execute administrative commands to the data platform. So what's so difficult about that? 

## Exploitation

The primary challenge in exploiting the lack of authorization associated with the `/[RootOnlyRedacted]` endpoint was intercepting API calls made by the CLI binary. This task proved to be difficult due to two main factors: the restrictions on installing tools on the remote host server of the data platform and the inherent complexity of the environment. These combined obstacles made discovering the vulnerable endpoint a particularly challenging process.

To intercept the API calls from the CLI, I've opted to install the data platform onto a container that I controlled. Since I had root-level privileges on the container, I could set up forwarding of all HTTP/HTTPS traffic to my proxy hosted on my local machine which allowed me to analyze each request made from the CLI binary. The discovery of the `/[RootOnlyRedacted]` endpoint was made when I was reviewing the inner-working of each API request made by the CLI binary.

The diagram below illustrates the process. I created an Ubuntu container, downloaded the data platform, and set up port forwarding to redirect traffic from port 17000, generated by the CLI, to port 8080 on my host. Burp Suite was configured on my host machine to redirect traffic from port 8080 to an SSH SOCKS proxy listener, which ultimately forwarded the traffic to the bastion host. This setup allowed me to execute CLI commands from my Ubuntu container, review and manipulate traffic via my Burp proxy, and forward the traffic to the data platform.

![](/images/diagram_whentrustfails.png)

## Conclusion 
In conclusion, the vulnerability and exploitation process described in this blog post may appear straightforward; however, the fact that the data platform server was hosted internally and the lack of privileges to install necessary tools on the server added a significant layer of complexity to the challenge.

The most important takeaway from this experience is the crucial role of research in uncovering and understanding such vulnerabilities. By delving deep into the API calls made by CLI binary and its interactions with the data platform, I was able to identify a security flaw that could have had severe consequences if left unchecked. 

Respecting the client's request, I opted to detail the vulnerability in this write-up rather than submitting it for a CVE, ensuring that valuable insights are still shared with the community.



