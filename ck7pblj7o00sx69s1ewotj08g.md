## How Our MongoDB Data Was Kidnapped and How You Can Protect Your MongoDB from Data Kidnappers

A few months ago I worked on a project and the application data got kidnapped 2 months after our team deployed the application. You might be wondering what Data Kidnapping means. Well, Data Kidnapping is the process of gaining unauthorised access to data by a malicious user and the encryption or erasure that data for malicious purposes. These malicious purposes could involve demanding for a ransom, selling of the data, political purposes, to crash a competition's business and the list goes on. I guess the extent of evil humans can do is unimaginable. 

### How it Happened

Alright let's get back to the story. I was a solo backend developer on a CRM like application. It was designed as an internal tool and I setup CI/CD pipelines to handle automated deployments to a single Digital Ocean droplet. I intended the environment to serve as a hosted *development* environment for testing purposes. One morning I was woke up to messages asking why the application no longer works. Unknown to me, the product owners had started using the development environment as a production environment which isn't so bad except for the fact that I didn't setup that environment to be production ready. I quickly went through the application logs and realised that no database call went through. Next thing I did was to open up my database client and connect to the cloud instance. I connected and to my surprise the database was empty. I mean only the default MongoDB collections were there. I started to panic then I looked closer and found one strange looking collection. I opened it up and saw a weird record that contained a message asking me to send an email to a specified email address to retrieve my data. At this point, I already knew what was going on. Our data had been kidnapped. I reached out to the product owners to explain what just happened and to my relieve they explained that no serious operation had been done on the application and all they did was just add user and project records meaning we can recreate the missing records. I decided to email this malicious person and he asked for $250 worth BTC for him to return the data. 

![Screenshot 2020-03-12 at 22.12.08.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1584047818372/CrXQ2yhBj.png)

The product owners suggested that we ignore him and just go through the painful process of recreating the lost records.

### Protecting Your Data
After this event, I decided not to let anything like this occur again as it could have had disastrous effects if the stakes were different. So I researched on ways to prevent this and secure MongoDB from malicious people on the internet. Let's look at some steps to take to protect your data.

#### Enable Authentication on your MongoDB instance

By default, MongoDB installs with authentication disabled. This is great for quick development on your local machine but becomes a risk when you install it on a server instance with public access. To secure your databases, enable authentication when running any MongoDB instance. Mongoaudit wrote [an article](https://medium.com/mongoaudit/how-to-enable-authentication-on-mongodb-b9e8a924efac) that helps get this done easily. You should check it out.

#### Run MongoDB on Non-standard ports

MongoDB's standard port is **27017** and this is the first port attackers would attempt to exploit. It's advisable to setup MongoDB on a different port to make the attacker's life a little harder. This can be done by following the steps highlighted [here](https://www.configserverfirewall.com/mongodb/change-mongodb-default-port/).

#### Block all Incoming Traffic to the MongoDB port using a Firewall Rule

We can go one step further and block all incoming traffic to the specified port running the MongoDB instance by specifying firewall rules. This can be done using [ufw](https://www.linux.com/tutorials/introduction-uncomplicated-firewall-ufw/) on linux or [Windows Defender Firewall](https://www.faqforge.com/windows/windows-10/how-to-create-advanced-firewall-rules-in-windows-10-firewall/) on Windows. This can also be done on the network level on cloud service providers. Digital Ocean provides [Cloud Firewalls](https://www.digitalocean.com/docs/networking/firewalls/) that can be attached to droplets, AWS has [Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html) for EC2 instances, GCP has [Firewall rules](https://cloud.google.com/vpc/docs/firewalls) for VM instances, etc. So make sure to only allow traffic from only known IP addresses.

 #### Setup Continuous Data backups 

In events of crisis, backups are life savers and can determine if your business dies or lives to see another day. It's important to ensure that your MongoDB is backed up often. The backup interval is up to you based on your business needs, budget and [Recovery Period Objective](https://whatis.techtarget.com/definition/recovery-point-objective-RPO). Managed MongoDB vendors abstract this process and make things frictionless at a cost. If you're self hosting your database instance, periodic backups/snapshots can be done on the instance volumes or a periodic export of the data can be implemented. The [MongoDB Backup documentation](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/)  does well to cover how to achieve this.

#### Encryption

Encryption is also a tool that helps protect data from malicious internet users. MongoDB data encryption can be done in transit using [TLS/SSL](https://docs.mongodb.com/manual/core/security-transport-encryption/) and/or at rest using any encryption standard supported by MongoDB. You can find more information about that [here](https://docs.mongodb.com/manual/core/security-encryption-at-rest/).

### Conclusion

Security is very important and should be taken seriously when working on software applications. Leaving things to chance might not end well. This article doesn't attempt to cover all security best practices but to highlight some of the important ones. Feel free to explore more ways to protect your data and let me know in the comments below.

#### References
- [Data kidnapping by Management Mania](https://managementmania.com/en/data-kidnapping)
- [How to Enable Authentication on MongoDB](https://medium.com/mongoaudit/how-to-enable-authentication-on-mongodb-b9e8a924efac)
- [Change MongoDB Default Port in Ubuntu/CentOS/Windows](https://www.configserverfirewall.com/mongodb/change-mongodb-default-port/)
- [MongoDB Manual - Security](https://docs.mongodb.com/manual/security/)