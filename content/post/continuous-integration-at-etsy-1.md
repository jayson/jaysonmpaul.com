+++
date = "2013-09-23T20:26:25-04:00"
draft = false
title = "LXC – Running 14,000 tests per day and beyond! (Part 1)"

+++

Continuous Integration at Etsy
------------------------------

As Etsy continues to grow and hire more developers, we have faced the continuous integration scaling challenge of how to execute multiple concurrent test suites without slowing the pace of our deploy queue. With a deployment rate of up to 65 deploys / day and a total of 30 test suites (unit tests, integration tests, functional tests, smokers...) that run for every deploy, this means running the test suites 1950 times a day.

Our core philosophy is to work from the master branch as much as possible; while developers can run the test suites on their own virtual machines, it breaks the principle of [testing in a clone of a production environment](http://en.wikipedia.org/wiki/Continuous_integration#Test_in_a_clone_of_the_production_environment). Since everyone has root access and can install anything on their VMs, we can't ensure a sane, standard test environment on developer VMs. Yet, they must ensure their code will not break the tests before they commit, or they will block the deploy queue. To this end, we developed a small library called "[try](http://codeascraft.etsy.com/2011/10/11/did-you-try-it-before-you-committed/)" which sends a diff of the developer's work to a Jenkins server that in turns applies it against the latest copy of the master branch and runs all the test suites (checkout [TryLib on github](https://github.com/etsy/TryLib)). Unfortunately, providing this feedback loop to every developer comes at a cost. With up to 10 developers "trying" their changes simultaneously, we now need to run the test suites an additional 13700 times a day (we average 515 try runs a day, with different test suite configurations).

Last but not least, the test feedback loop [ought to be short](http://en.wikipedia.org/wiki/Continuous_integration#Keep_the_build_fast) to provide any value. Our current "SLA" for running all test suites is 5 minutes. Following the "[divide and concur](http://codeascraft.com/2011/04/20/divide-and-concur/)" strategy and with the help of the [master project Jenkins plugin](https://github.com/etsy/jenkins-master-project), we are able to run the test suites in parallel. Our initial setup had to maintain multiple executors per box, which caused connection issues, and required preventing multiple executors from attempting to access the same Jenkins workspace at once.

Workload Considerations
-----------------------

While parallelization was initially a major victory, it soon became problematic.  Our [workload](http://codeascraft.com/2011/04/20/divide-and-concur/) consists of two different classes of tests: one class constrained by CPU ("lightweight executors", which run our unit tests), and the other bound to disk I/O ("heavyweight executors", which run our integration tests).  Prior to virtualization, we would find that if multiple heavyweight jobs ran on a single host, the host would slow to crawl, as its disk would be hammered.

We also had issues with workspace contention when executing multiple jobs on a single host, and this need for filesystem isolation, in combination with resource contention and issues within Jenkins concerning too many parallel executors on a single host, led us toward the decision that virtualization was the best path forward.  We were, however, concerned with the logistics of managing potentially hundreds of VMs using our [standard KVM/QEMU/libvirt environment](http://codeascraft.etsy.com/2012/03/13/making-it-virtually-easy-to-deploy-on-day-one/).

Virtualization with LXC
-----------------------

Enter LXC, a technology wonderfully suited to virtualizing homogenized workloads.  Knowing that the majority of our executors are constrained on CPU, we realized that using a virtualization technology for which we would need to be conscious about resource allocation did not make sense for our needs.  Instead of concerning ourselves with processor pinning, RAM provisioning, disk imaging, and managing multiple running kernels, LXC allows us to shove a bunch of jailed processes in LVM containers and call it a day.

While the workings of LXC are outside the scope of this post, here is a little bit of information about our specific configuration.  Our physical hosts are 2U Supermicro chassis containing four blades each.  Each blade has three attached SSDs for a total of 12 disks per chassis.  In our case, disk I/O is a key constraint; if we run two heavyweight jobs accessing a single disk simultaneously, the run time doubles for each job.  In order to maintain our 5 minute run time SLA, we ensure that only one heavyweight executor runs on each disk at any given time.

Previously, we had many executors running on each physical host, but with LXC, we have decided that each container (LXC's terminology for a virtualized instance) shall host a single executor.  Maintaining a 1:1 container:executor mapping alleviates the connection issues we saw prior to virtualization within Jenkins while trying to maintain a large number of executors per host, and we never have to worry about multiple executors attempting to access the same Jenkins workspace at once.  It also allows for easy provisioning using the Virtual Madness interface discussed below.

Continue to [Part 2](https://jaysonmpaul.com/post/continuous-integration-at-etsy-2)...

Co-written by Nassim Kammah, Patrick McDonnell and Jayson Paul. Be sure to stay tuned for Part 2 of this blog post where Jayson will be discussing automating this process of automating LXC management.

*You can follow Nassim on Twitter at *[@kepioo](https://twitter.com/kepioo)\
*You can follow Patrick on Twitter at [@mcdonnps\
](https://twitter.com/mcdonnps)You can follow Jayson on Twitter at [@jaysonmpaul](https://twitter.com/jaysonmpaul)*
