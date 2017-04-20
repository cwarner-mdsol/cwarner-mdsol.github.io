### i'm on pto from 4/24/2017 - 05/02/2017

There isn't much to say most things are operational or working and shouldn't have any problems continuing to function for the next week or so.

### Things could possibly stop being functional however

ldap, kvcluster and tower would be the only major concerns but anything going immediately wrong here is highly unlikely. In the cases where there is abject failure for LDAP one can search sumologic with the following regex:  

``` (ldap) AND _sourceName="/mnt/medistrano/shared/log/ldap_integration.log ```  

And it should provide relevant information as to what is going on as it relates to Medistrano.

The other case is where LDAP starts spuriously denying connects which would likely be open files limit, meaning in this case more than 512,000 files have been opened on all consumer nodes. While it's highly unlikely, anything is possible, this has been set via limits.conf because the hard ulimit is set to 1024 using the 14.04.5 version of Ubuntu. This was never the case in older versions of the image as it would use sys.file-max value even with hard limits. sys.file-max is determined by the kernel, set based on memory of the node. Nevertheless we've increased node size and set the limits relatively high and haven't seen a problem there since.

There was a reported issue of ldap.imedidata.net timing out that I haven't ever seen, ever. In the case where this was brought up by other people it turned out that they were trying to connect to a non-existent node, which even in that case it would have made it to the node in question before timing out. I haven't heard of this happening from anyone else. In the case that you can't connect to ldap.imedidata.net or it times out please use the tools to verify your route and if you can confirm timeout, please, please, capture tcpdump of the connection story. There are many tutorials on using tcpdump so I won't bother to link to anything specific as a simple google search should suffice. RTFM would be heartless but it's obviously also very useful.

That aside in the case there is an actual problem with denied connections, the first thing to do is login to the consumer nodes and run the below which should give you an idea of how many files are open:

``` lsof | grep openldap | wc -l ``` 

Check the amount of open files, it should be between 10,000-15,000 openfiles give or take. Anything higher shouldn't matter but that's about normal for the last week or two and well under the 512,000 limit. These connections should eventually scale down slightly when nscd is renabled and we have proper caching. There are various tickets on the backlog to enable swap and some other things but for all intents and purposes none of it should hamstring any ldap operation.

In the case where the nodes are at their limit of 512,000 something is terribly wrong and you should restart the slapd service which will give you sometime to diagnose the situation and in the interim also raise the limits even higher to sys.file-max. If after that you are hitting that limit on all nodes, this is an actual emergency and it's safe to call me.

### ldap
Ldap is down, what do I do? Do not panic, all six nodes would have to go down for it to be a real issue. If that has happened, there are much bigger problems. That aside you can spin up a new LDAP cluster and migrate the data from the latest backup in S3. [Instructions on how to spin up ldap with latest backup](https://github.com/mdsol/openldap-playbook/blob/master/docs/ldap_administration_manual.md). A vote of confidence is that this has never happened before, even during the rush on the cluster during initial introduction, the nodes themselves never actually went down (they slowed to a crawl but kept processing requests, albeit, extremely slowly).

### kvcluster
If the key value cluster goes down for some reason, all nodes would have to fail, there are much bigger problems if this happens. If we lose 2 nodes the cluster is still fully operational. [Cluster Health](http://kvcluster.imedidata.net:8500/ui/#/infra-us-east-1/services/consul) - [Help! Need to spin up a new cluster] (https://github.com/mdsol/kvcluster-playbook/blob/54915e49c34206936aecbd1b7ff21dd82d9f9522/docs/starting%20the%20cluster.md). A vote of confidence is that this has actually happened where we lost one of the nodes, the cluster kept working without anyone noticing for almost an entire week. Once identified we pulled the node, no data was loss and nothing actually happened. It looked as if the node was just unreachable for a period of time and was deemed inactive and removed from the rest of the cluster. kvcluster.imedidata.net is actually very important as consul-template hits it for every node that we have up now. Please treat the data specifically the ldap keys etc as production related critical infrastructure.

### tower
Tower could possibly go down if the node goes bad as the clustered version isn't operational yet. Don't shutdown or terminate the bad node. In this case you'll have to spin up a new node in red. It should have the same iam-role and settings as the bad node so just copy from there and download the tower-latest-setup install program, the backups for tower are in the operations-red bucket in tower.imedidata.net and just pull the latest. One you have the backup tarball, simply put in the directory and mv it to tower-backup-latest.tar.gz and run ./setup.sh -r. From there point the route53 A record to the fresh node and give it 60 seconds. Do not terminate the bad node, just stop it so I can diagnose when I get back, or even better figure out what the problem is and fix it before I get back. Do not make any modifications to the configuration unless you absolutely must. Tower is in "beta" and not heavily used just yet so no big deal, a handful of people would be inconvenienced at best. The other thing is tower can't connect to X nodes or whatever. I haven't gotten around to making sure the cloud-app group has the tower ip (really need the tower sec group which has a list of the ips) in any event you can just add the current ip for now until this is ready. The only account available right now is green.

### Do not make changes unless someone can prove their problem
Please, do not make any infrastructure related changes to any of the above unless someone can prove their problem with data, a trace, coredump, screenshot, something. We have to stop this idea that things just magically happen and jumping on the truck everytime someone yells fire. Things happen and there are valid reasons for that but unless it's affecting everyone 9 times out of 10 it's related to that specific user. Please operate on that notion until proven otherwise. If they can't show, prove or communicate the problem, leave it until I get back. Especially if it's not affecting any other user.
