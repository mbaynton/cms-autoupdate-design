#  Drupal/Backdrop Automatic Update Technical Design Planning

The purpose of this document is to form consensus around the technical decisions related to distributing and deploying automatic updates for PHP-based CMS software such as Drupal or Backdrop. There's a large bin of quality ideas on this topic at https://www.drupal.org/node/2367319, but the lack of consensus is probably hampering progress. My hope is that a single document updated via github pull requests will provide a clear picture of the plan and a controlled structure to amend it as better ideas are suggested.

If you just have a quick observation or thought, post an issue here or a comment back at the d.o issue; if you have a revision to suggest, please feel free to submit a pull request. If you would like to help take a leadership role in shaping this document, ask me about commit access. Although there is currently only one committer this is not an intentional dictatorship. Revisions will be evaluated for merging based on whether they further the objectives outlined in the ***design concerns***. The design concerns are of course themselves up for amending as well...I've tried to gather all the distinct points that were raised in the d.o issue. 

The document is written in Markdown, if you're looking for an easy way to edit markdown for a PR try dillinger.io.

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
  - the URL to an endpoint where updates and related messages will be sent.
  - A randomly generated shared secret key.

For the endpoint to ensure messages being recieved are from the trusted central infrastructure, the Hawk HTTP authentication scheme will be used. Under Hawk, all messages exchanged between the central infrastructure and the site are be sent along with the message's sha-256 hash and an HMAC result computed over the hash using the shared secret key established during registration. Hawk also incorporates features to combat replay attacks.

When an automatic update event is launched, the following exchanges occur between central infrastructure and each registered site:
  1. Central infrastructure sends a notification of new updates to the site's endpoint. This notification consists of an index file of update packages that are currently on offer, including the update's security risk score. The updates on offer could pertain to core or contrib, and could be offered as backports for previous versions, so there could potentially be several update packages. This index file makes it possilbe to offer many update packages simultaneously, without requiring sites to send central infrastructure detailed information about their installed modules/module versions or owner's thresholds for installing an update automatically.
  4. Site determines which of the offered updates it wishes to receive automtically based on which modules are installed, the security risk scores, and whether a backport to the currently installed version is available. If site wishes to receive one or more of the update packages, it responds with the name of the first. If site does not wish to further process any available updates, it responds it is done receiving update packages. In this case the exchange ends.
  5. Central infrastructure sends the indicated update package. Although each update package contains an embedded OpenSSL-based signature ensuring the integrity and authenticity of the package, OpenSSL may not be available everywhere, so the entirety of the message containing the update package is still sent with an HMAC.
  6. If site requires additional update packages from the index file, it responds with another name, else it responds it is done receiving update packages. In this case the exchange ends.

### Deployment
Updates will be packaged as self-contained executable phar files signed with OpenSSL. Although they will be pushed by the distribution mechanism and could subsequently be executed by an automatic phar runner (see below), they are not tightly coupled to any specific distribution method and could be used as an alternative and safe way to manually apply an update as well, should site owners choose not to allow automatic updates.

Using Phar archives as the basis for packaging updates has these advantages:
  - It is a full-featured archive format guaranteed to be readable by any server needing an update.
  - The rich tooling for Phar examination and manipulation built into the interpreter and SPL would make deployment customization as easy as possible for advanced users.
  - For servers where PHP is linked with OpenSSL, the Phar format offers assurance that an update package is authentic and free of tampering through digital signatures. Based on popular linux distribution package dependencies for php, the vast majority of php installations probably do have OpenSSL support.
  - The ability to package executable code - like an installer - with the patch itself offers the possibility to accomodate future needs within existing infrastructure. (`composer install`, anyone?)

Update packages will run in three distinct phases. Normally the phases will all be executed in order, but the package will also support manual execution, and advanced users may choose to run the phases separately. In the first phase, the update is verified to be applicable to the site it is attempting to be applied to by comparing version numbers. In the second phase, updates to the code tree on the filesystem occur (file adds, modifications, and/or deletes.) In the third phase, a post-update script is run to perform tasks such as invoking database schema updates.

Although the phar packages will, strictly speaking, be directly executable at the command line, this should not be a recommended/documented primary way to run them. The main reason for this is that it is impractical for users to verify that the particular phar they are about to execute contains an OpenSSL signature at all. When it does not, the phar will be executed by the interpreter without question, so anyone in a position to tamper with the package could easily defeat the digital signature protection by simply removing it from the archive. (The extension ".phars" should totally be a thing, but that's another topic altogether.) Secondarily, in order to execute a properly signed phar from the command line, one needs to first copy the public key to a file in the same directory as the archive, named identical to the archive with .pubkey append, and that's just clunky.

Instead, the recommended ways to open and execute phar update packages will be by calling a small bit of previously installed trusted code, passing it the phar file. This code could be invoked through drush or other command-line tools, as well as directly by the CMS. It will:
  1. Check the php installation for OpenSSL support. If not present, 0 out the digitally signed feature flag from the update package to be run and set interpreter runtime value to phar.require_hash=0 long enough to open this package without any signature. (This seems the simplest way; we could alternatively skip the runtime config change by computing one of the hash-only signatures for the package and overwriting the OpenSSL signature with it.)
  2. If OpenSSL support is present in this installation, verify that the update package to run is signed with OpenSSL. Abort if not.
  3. Copy the public key from a known location within the CMS installation to the working directory where the phar is stored, and name it matching the .phar file's name per the interpreter's requirements.
  4. Set the working directory to the root of the CMS installation to be updated.
  5. Execute the phar.

This model makes it as simple as possible to kick off the update code in a variety of ways with necessary OS permissions.

### Automatic phar runner
A separate module, the Trusted Installer, will manage execution of phar archives such that they are installed onto a site. Splitting this process into a separate module offers the potential to offer future support for one-click installation of contrib modules by site owners, without having to have their sites writable by the webserver and without having to deal with sftp credentials.

Fundamentally, the Trusted Installer will be a public key repository, an API to run a phar on a particular site, and two implementations of the phar runner. One will target setups where the webserver is allowed to update the CMS code directly (WordPress' automatic updates  shows us this is prevalant among shared hosts), and another will target Unix-like OS's where the webserver does not have sufficient permission to modify the CMS source tree.

The in-webserver runner should be trivial.

The runner for the case where the webserver does not have permission to modify the CMS source will have these components:
  1. A small amount of code within the CMS that responds to update package received events by connecting to a local socket and passing the update package through it, perhaps plus some metadata about the site it is coming from.
  2. A daemon process that receives data on the local socket, calls the phar loader wrapper described above to verify the signature, and executes the phar.

Under this model, the server administrator would need to do a one-time setup of the daemon, configuring it to run under a user allowed to edit the CMS source tree(s). One such daemon should be sufficient to update multiple CMS installations if desired by running as a priviliged process that forks and makes setuid/setgid calls.

The key observation from a security standpoint is that the process with permission to adjust the site's source tree first performs its own verification that the changes to be made are from a trusted source.
