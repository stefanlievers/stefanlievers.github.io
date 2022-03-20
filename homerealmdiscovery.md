# Home Realm Discovery for an Application

## Skipping the "choose account" page when using Single Sign-On in Azure

When an application needs to authenticate a user most of the time Single Sign On (SSO) comes to mind. It's convenient for the user, a trusted identity management platform can be used and there is no need to remember an extra pair of credentials for yet another SaaS application. In this particular blogpost we focus on Azure AD.

## Who is this blogpost for?
If you have the situation where your users get a "choose account" screen to choose which account to use for authentication, but there is only one account present, then this post is for you.

Keep in mind that if there are multiple accounts known in the browser, the selection screen will always show.  

## What will we create?
We will create an Azure AD Policy which will help us for Home Realm Discovery. In other words, we tell Azure: if you have an account at hand that uses the domain we tell you in this policy, don't give us a choice, just use it and move on.

The goal of this action is to give your users one less click to login, which delivers a more seamless login experience. 

## What do we need?

There's two ways I know of to get this done, one is PowerShell, the other is Microsoft Graph. In this post I show the PowerShell way only. Furthermore we need the PowerShell AzureADPreview module, the cmdlets we use are currently in preview.

The last thing we need is an Enterprise Application to which we can link the policy.

You do need an Azure subscription, but I assume you have one since you're dealing with this challenge.

  

## Let's get started

Start PowerShell.

Import the AzureADPreview module

```PowerShell

Import-Module AzureADPreview

```

If you receive errors you may need to install the module first.

If you get the error that certain cmdlets are already imported, use -allowclobber

```PowerShell

Import-Module AzureADPreview -AllowClobber

```

Now that the module is loaded the AzureADPolicy cmdlets are available to us.

  

### List the current policies

Type:

```PowerShell

Get-AzureADPolicy

```

My output looks something like this:

```

Id DisplayName Type IsOrganizationDefault

-- ----------- ---- -------------

<< id hidden for obvious reasons >> B2BManagementPolicy B2BManagementPolicy True

```

> Note: If you already have policies that set Home Realms, please figure out how they work first.

  

### Create the new policy

There are 3 ways to about an auto-acceleration policy, depending on your architecture:

* Single-tenant auto-acceleration

* Multi-tentant auto-acceleration

* Username/Password authentication for specific applications

  

If you have a single-tenant use the first option, but when in doubt go the multi-tenant route.

  

#### Create a single-tent auto-acceleration policy

```PowerShell

New-AzureADPolicy -Definition @("{`"HomeRealmDiscoveryPolicy`":{`"AccelerateToFederatedDomain`":true}}") -DisplayName BasicAutoAccelerationPolicy -Type HomeRealmDiscoveryPolicy

  

# "BasicAutoAccelerationPolicy" is the policy name and can be edited to match name conventions.

```

  

#### Create a multi-tenant auto-acceleration policy

```PowerShell

New-AzureADPolicy -Definition @("{`"HomeRealmDiscoveryPolicy`":{`"AccelerateToFederatedDomain`":true, `"PreferredDomain`":`"stefanlievers.com`"}}") -DisplayName "MultiDomainAutoAccelerationPolicy" -Type HomeRealmDiscoveryPolicy

  

# Change the PrefferedDomain to the domain you wish to use. If you want to use more then one domain, create multiple policies.

# "MultiDomainAutoAccelerationPolicy" is the policy name and can be edited to match name conventions.

```

#### Create a Username/Password auto-acceleration policy

```PowerShell

New-AzureADPolicy -Definition @("{`"HomeRealmDiscoveryPolicy`":{`"AllowCloudPasswordValidation`":true}}") -DisplayName "EnableDirectAuthPolicy" -Type HomeRealmDiscoveryPolicy

  

# "EnableDirectAuthPolicy" is the policy name and can be edited to match name conventions.

```

### Let's see the policy we build

Let's get a list of policies one again

```PowerShell

Get-AzureADPolicy

```

Based on the choice you made above your result may vary, my output looks like this:

```

Id DisplayName Type IsOrganizationDefault

-- ----------- ---- -------------

<< id hidden for obvious reasons >> B2BManagementPolicy B2BManagementPolicy True

xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx MultiDomainAutoAccelerationPolicy HomeRealmDiscoveryPolicy False

```

> Copy the Id of your newly created policy, you need it in a next step.

  

### List the Enterprise Applications we have

Let's list the Enterprise Applications from our Azure tenant

```PowerShell

Get-AzureADServicePrincipal

```

This list might be short or very long, depending on your environment.

From personal experience I noticed that the application you need is not always present in the output. If that's also true in your case just login to the Azure Portal, browse to the Enterprise Application and copy the ObjectID value from the application blade.

  

### Link the application to the policy

Let's link the application to the policy. Remember it like that, because you link the Service Principal to the Policy, not the other way around.

> Note: In this step you need two ID's, the Object ID of the application and the Policy ID from the policy we just created.

```PowerShell

Add-AzureADServicePrincipalPolicy -Id "InsertObjectIDhere (from application)" -RefObjectId "InsertPolicyIDhere"

```
Now your Enterprise Application is linked to the Policy you created. 
It's time to test if your changes have the desired effect.

### Check the application
For this step you need the ObjectId from the Enterprise application again.
```PowerShell
Get-AzureADServicePrincipalPolicy -Id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#Fill in the ObjectId from the Enterprise Application you want to check.
```
### Removing a linked policy from your application
Removing a linked policy from your application can be done easily using the corresponding Cmdlets.
```PowerShell
Remove-AzureADServicePrincipalPolicy
```

### Take notice
- You're using _preview_ cmdlets, things may change, results may be unexpected.
- You create **one** policy
- You can link that policy to zero, one or multiple (n) Enterprise Applications
- Multiple credentials present in the users webbrowser session can still give the "choose account" screen
- Error messages can most of the times be linked to using the wrong Id's, it can become blurry if you don't seperate the Id's from the start. A notepad session is your friend here.
