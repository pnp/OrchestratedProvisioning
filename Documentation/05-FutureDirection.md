# Teams Provisioning Sample

## Part 5: Addendum: A Change in Direction

It's only been a couple weeks since I published the 4-part blog series on Teams provisioning, and already I've learned a lot. So here is part 5 of the 4-part series, which will explore early learnings and begin to discuss future directions for the project.

The good news is that the feedback has been extremely positive. A lot of partners and others in the community have indicated an interest in a Teams provisioning solution that's easy for no-code/low-code developers to use. Yet in spite of the positive feedback, it's also become apparent that there's a lot of work ahead to make this viable in the real world, and that not all my original goals will be attainable for a while anyway (specifically, the goal to avoid using user delegated permissions).

### A Reality Check

Really, this is still just a Proof of Concept. There are a number of aspects of the project that remain incomplete:

* API's are still in preview (however they'll probably be released soon)
* The solution is largely untested; I've really only tested it with the sample Teams JSON from the documentation
* There isn't yet a clear way to generate the JSON "templates" on which the solution is based. I had started to discuss adding a command in the [Office 365 CLI](https://pnp.github.io/office365-cli/) to generate these templates; that's another whole project in itself.
* Some things, such as app installation in a Team, aren't supported with "app only" permissions; as a result, one of the apps in the sample content doesn't get installed (and no error is reported back from the API)

The biggest thing I've learned is that provisioning a Team really isn't sufficient for many. Several people expected the solution to provision not only the Team, but other Office 365 content that appears in a Team, such as files in SharePoint and plans in Planner. This is the flip side of the "single pane of glass": Teams stitches together the various services in Office 365 so seamlessly that it's easy to assume that everything on the screen comes from Teams itself.

### PnP to the rescue

Another thing I didn't realize when I wrote this is that the PnP team is nearly ready to release a new feature, Tenant Templates, which will soon be able to provision Teams as well as SharePoint sites, and most likely other Office 365 content over time. I heard a little about this at Ignite last fall, but had missed the [December webcast](https://developer.microsoft.com/en-us/office/blogs/pnp-webcast-introduction-to-pnp-tenant-templates/) which explained that Teams provisioning was coming soon.

If you've ever used the [SharePoint Starter Kit](https://github.com/SharePoint/sp-starter-kit), you know that it provisions the foundation of an Intranet with a single PowerShell command, including a hub with multiple site collections and a number of great customizations. The Starter Kit's powerful provisioning engine extends the [PnP Provisioning Templates](https://docs.microsoft.com/en-us/sharepoint/dev/solution-guidance/pnp-provisioning-framework) which have been around for a while, but which were limited to provisioning a single SharePoint site. This is the genesis of Tenant Templates. The PnP team plans to build this up to a point where complete Office 365 demos can be provisioned just as easily.

So the obvious direction is to use Tenant Templates to provision not only a Team but also the underlying content. This has the advantage that the PnP team is already developing and testing Tenant Templates, and is likely to invest in tools to generate them. Switching to Tenant Templates will allow this project to focus on provisioning logic rather than the templating technology itself.

Adding support for Tenant Templates will unfortunately mean a rewrite of the solution code from v2 Azure Functions in NodeJS to v1 Azure Functions in C#/.NET Classic. This limitation is inherited from the [SharePoint CSOM](https://docs.microsoft.com/en-us/sharepoint/dev/sp-add-ins/sharepoint-net-server-csom-jsom-and-rest-api-index), a major dependency for Tenant Templates, which is currently .NET Classic only (though they're working on a .NET Core implementation).

### Security Challenges

One of the original architectural goals for the project was to create Teams using an application identity so no user identity is involved. Although the app identity needs to have a lot of permission to do its work (so far Group.ReadWrite.All and User.Read.All), management of app secrets in [KeyVault](https://azure.microsoft.com/en-us/services/key-vault/) with [Managed Service Identities](https://azure.microsoft.com/en-us/blog/keep-credentials-out-of-code-introducing-azure-ad-managed-service-identity/) (MSIs) is well understood and effectively prevents someone from using it outside of the application code. [Use of an MSI to access the Graph](https://finarne.wordpress.com/2019/03/17/azure-function-using-a-managed-identity-to-call-sharepoint-online/) rather than just to access an app secret from KeyVault might have been a further step to prevent misuse of such an app identity.

Unlike an app identity, user delegated access requires use of a licensed user account with sufficient permissions to do whatever the service needs. This does cut down on the power of the account somewhat - for example, an app with Group.ReadWrite.All can read all content in all Groups, whereas a user account may not have permission to read existing Groups. However, that same user account can be used log in via a web browser, thus opening the possibility for misuse. Furthermore, to to be useful in a background process, such an identity may also need an exception from security features such as multi-factor authentication and conditional access.

However it's become clear to me that for now user delegated access is the only route that will really work for a couple of reasons:

* The biggest reason is that Tenant Templates don't support app identities. This isn't by choice; it's because some of the underlying API's don't support app identities. And even for calls that support them, there are sometimes caveats (such as the [issues with Team cloning](https://laurakokkarinen.com/cloning-teams-and-configuring-tabs-via-microsoft-graph-cloning-a-team/#bugs-or-working-as-intended) and the inability to install a Teams app - though these may just be Beta limitations). There are parts of the SharePoint API such as managed metadata which don't support app identities; other workloads such as Planner don't support them at all.
* Even where app identities are supported, the results when using them may vary from the results of the same calls with user delegated permissions. That makes it difficult to test a template interactively with confidence that it will work the same way when a service applies it using app permissions.

For these reasons, it seems clear that the project needs to shift to user delegated access, and the tricky business of managing a privileged user account in a background service. Informal discussion has revealed that this is already the situation in some large enterprises which have implemented their own SharePoint provisioning services.

To that end, I'm looking for best practices (or at least well understood tradeoffs) in how to set up and manage such a user account.

### What's Next and other questions

A few things are clear to me at least; I'd welcome your thoughts and comments:

* The solution needs to support Tenant Templates instead of or in addition to Teams Templates
* The solution needs to shift to user delegated permissions
* The solution needs to move out of my personal Github account to a place where the whole community can participate; effective immediately the solution will migrate [to this repo under the PnP Github org](#).

Some questions I'm still pondering - and I'd hugely appreciate your ideas and input:

* Should the solution continue to support the JSON from the Create Team Graph call in addition to Tenant Templates?
* Should the solution continue to support Team cloning in spite of the numerous caveats? (Note that some of these will be addressed by switching to user delegated access.)
* Is the Azure Storage queue-driven access the right design? (I still think it is - and people love that it doesn't require a Premium Flow connector. But while everything else is changing, it's worth bringing it up for discussion.)
* What is the best approach to managing a user identity for delegated access - or what are the approaches and tradeoffs? Is there a way to get the identity via a Flow connection, and is that a good idea? It would encourage use of the Flow creator's account, is that a good thing or does it mean giving too much permission to the Flow creator? And would it be worth moving to a custom connector, which is a Premium feature?

Another reality check is that my time for this project is limited over the next few weeks as I'll be travelling almost constantly and leading workshops until after SharePoint Conference, but that's a temporary situation. During this time, the PnP Provisioning Engine should release Teams creation within Tenant Templates. I also hope to use the time to gather input and determine the right architecture for the next phase of the project. If time allows, I'll start rewriting in C#/.NET Classic in anticipation of the move to the PnP provisioning engine.

Thanks in advance for your thoughts and ideas!