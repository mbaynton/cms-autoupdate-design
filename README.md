#  Drupal/Backdrop Automatic Update Technical Design Planning

The purpose of this document is to form concensus around the technical decisions related to distributing and deploying automatic updates for PHP-based CMS software such as Drupal or Backdrop. There's a large bin of quality ideas on this topic at https://www.drupal.org/node/2367319, but the lack of concensus is probably hampering progress. My hope is that a single document updated via github pull requests will provide a clear picture of the plan and a controlled structure to amend it as better ideas are suggested.

If you just have a quick observation or thought, post an issue here or a comment back at the d.o issue; if you have a revision to suggest, please feel free to submit a pull request. Revisions will be evaluated for merging based on whether they further the objectives outlined in the ***design concerns***. The design concerns are of course themselves up for amending as well...I've tried to gather all the distinct points that were raised in the d.o issue. The document is written in Markdown, if you're looking for an easy way to edit markdown for a PR try dillinger.io.

If you would like to help take a leadership role in shaping this document, ask me about commit access. Although there is currently only one committer this is not an intentional dictatorship.

## Purpose of Automatic Updates
Automatic updates are envisoned as a tool to mitigate the impact of security vulnerabilities in the CMS software, particularly those where sites would otherwise be compromised via automated exploits.

## Design Concerns
The objective of this design is to address the below concerns:

  1. There is no "one size fits all" solution. Many personas are responsible for ongoing maintenance of websites, with varying requirements, preferences, and degrees of technical skill ([Source](https://www.drupal.org/node/2367319#comment-9308063)). Therefore, design features that promote configuration and customization are favorable, especially with regard to how the updated code is installed/deployed.
  2. Any central infrastructure used to carry out the automatic updates will be an obvious target for hackers seeking to gain control of all the updated websites ([Source](https://www.drupal.org/node/2367319#comment-9307997)), or simply to diminish the infrastructure's effectiveness at disseminating an update ([Source](https://www.drupal.org/node/2367319#comment-11379019)). Therefore, design features that mitigate the impact should central infrastructure be compromised are favorable, as are design features that make it difficult to effectuate brute force attacks against the central infrastructure.
  3. For a variety of reasons, it is not desirable for a single individual to be empowered to launch an automatic update event ([Source](https://www.drupal.org/node/2367319#comment-9313573)). Therefore, design features that administratively discourage or cryptographically ensure no single person can complete an automatic update event are favorable.
  4. When patching a disclosed security vulnerability, time is of the essence (Source [1](https://www.drupal.org/PSA-2014-003), [2](https://www.drupal.org/node/2367319#comment-9316051)); even more so in the case of a 0-day vulnerability. Therefore, design features that expedite the distribution and deployment of the automatic update are favorable.
  5. Industry-standard assurance technologies such as HTTPS and code signing are clearly desirable to ensure the authenticity and integrity of code in an automatic update. However, these technologies depend on software libraries external to the CMS and PHP, so assurance is not guaranteed to be possible on all servers running the CMS. The existing automatic update mechanism in [WordPress](https://wordpress.org/) opts to treat such technologies as optional benefits rather than requirements of the automatic update system ([Source](https://www.drupal.org/node/2367319#comment-11384875)). The lack of any serious problems arising in the wild as a result of this design provides empirical evidence that a design is favorable if it is capable of using security libraries when they are available, but is also capable of falling back to less rigorous methods so servers lacking these technologies are still updated.

## Design Proposal
The technical design can be divided into Deployment and Distribution components. Distribution refers to the process of getting the update package to all the servers that host CMS websites. Deployment refers to the details of installing the update to a given site once it is received.

In short, the Deployment will be accomplished via a PHAR archive signed with OpenSSL that is fully self-contained and secure, independent of security features built into the Distribution component. The Distribution will addiitonally utilize its own security features to ensure messages received are authentic and combat abuse arising from false messages received in volume.

### Distribution
In order to receive automatic updates, a site must opt in to receive them and register with a web service:
  - the URL to an endpoint where updates and related messages will be transmitted.
  - A seed for a PRNG generating at least 32-bit random values.

All messages exchanged between the central infrastructure and the site will be digitally signed using the current value of the PRNG as a shared secret. After each exchange, both ends increment their PRNG.

When an automatic update event is launched, the following exchanges occur between central infrastructure and each registered site:
  1. Central infrastructure sends a short (signed with rolling shared secret) initialization message to the site's endpoint.
  2. Site verifies message is authentic. If message exceeds the small expected size of the initialization message or if message signature is not verified, responds 403 and closes connection. If message is authentic, site issues a session token to central infrastructure. Further communications bearing a valid session token will have their signature verified even if the communication is large.
  3. Central infrastructure sends an index file of update packages that are currently on offer, including the update's security risk score. Like all messages, the index file is signed with the rolling key, and the session token is also sent. The updates on offer could pertain to core or contrib, and could be offered as backports for previous versions.
  4. Site determines which of the offered updates it wishes to receive automtically based on which modules are installed, the security risk scores, and whether a backport to the currently installed version is available. If site wishes to receive one or more of the update packages, it responds with the name of the first. If site does not wish to further process any available updates, it responds it is done receiving update packages. In this case the exchange ends.
  5. Central infrastructure sends the indicated update package. Although each update package contains an embedded OpenSSL-based signature ensuring the integrity and authenticity of the package, OpenSSL may not be available everywhere, so the message containing the update package is additionally signed with the rolling shared secret. This is likely weaker than OpenSSL, but renders tampering in transit infeasible.
  6. If site requires additional update packages from the index file, it responds with another name, else it responds it is done receiving update packages. In this case the exchange ends.


### Deployment
Updates will be packaged as executable PHAR archives signed with OpenSSL.
Discussion:
  1. PHAR archives 