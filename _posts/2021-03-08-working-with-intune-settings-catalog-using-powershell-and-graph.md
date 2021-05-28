---
id: 514
title: Working with Intune Settings Catalog using PowerShell and Graph
date: 2021-03-08T22:46:00+10:00
author: Ben
layout: post
guid: http://powers-hell.com/?p=514
permalink: /2021/03/08/working-with-intune-settings-catalog-using-powershell-and-graph/
obfx-header-scripts:
  - ""
obfx-footer-scripts:
  - ""
views:
  - "901"
spay_email:
  - ""
image: /assets/images/2021/03/settingsCatapalooza.gif
categories:
  - Azure
  - Graph
  - Intune
  - Powershell
tags:
  - Azure
  - Intune
  - Powershell
---
Microsoft has recently introduced even more ways to create device configuration profiles.. 

The new profile type, named **Settings Catalog**, allows us to explicitly define and configure a policy that has **only** the settings that they want for that profile, nothing more. Additionally, the existing configuration profiles and ADMX templates have been migrated to the **templates** profile type.

<!--more-->

<img loading="lazy" width="474" height="317" src="/assets/images/2021/03/image.png" />  

I sat down with [Mike Danoski](https://twitter.com/MikeDanoski) for an in-depth chat about this on the <a href="https://intune.training" data-type="URL" data-id="https://intune.training">Intune.Training</a> Channel (video below).<figure class="wp-block-embed-youtube wp-block-embed is-type-video is-provider-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio">

<div class="wp-block-embed__wrapper">
  <span class="embed-youtube" style="text-align:center; display: block;"></span>
</div> 

After spending time with Mike and seeing how settings catalog profiles work from the endpoint portal user interface, I immediately wanted to see what I could do with this new device management framework via graph.

So let's dive in and play!

## **Pulling settings catalog policies from Graph**

First, let's create a policy from the endpoint portal and see what is required to retrieve the policy data.

For this demo, I've created a simple settings catalog with a few settings around bitlocker as shown below. <figure class="wp-block-image size-large">

<img loading="lazy" width="1008" height="1000" src="/assets/images/2021/03/image-1.png" alt="" class="wp-image-517" srcset="/assets/images/2021/03/image-1.png 1008w, /assets/images/2021/03/image-1-300x298.png 300w, /assets/images/2021/03/image-1-150x150.png 150w, /assets/images/2021/03/image-1-768x762.png 768w" sizes="(max-width: 1008px) 100vw, 1008px" />  

The first thing we need to do, as always, is authenticate to graph - At this point I shouldn't need to explain what is happening here. We will use the msal.ps module to make things easier.

<pre class="wp-block-code" title="Configure Authentication for Graph."><code lang="powershell" class="language-powershell">$params = @{
    ClientId = 'd1ddf0e4-d672-4dae-b554-9d5bdfd93547'
    TenantId = 'powers-hell.com'
    DeviceCode = $true
}
$AuthHeader = @{Authorization = (Get-MsalToken @params).CreateAuthorizationHeader()}</code></pre>

Now that we've authenticated to graph, let's use the new graph endpoint **configurationPolicies** to have a look at how this new feature looks in the backend.

<pre class="wp-block-code" title="Get configurationPolicies"><code lang="powershell" class="language-powershell">$baseUri = "https://graph.microsoft.com/beta/deviceManagement"
$restParam = @{
    Method = 'Get'
    Uri = "$baseUri/configurationPolicies"
    Headers = $authHeaders
    ContentType = 'Application/json'
}

$configPolicies = Invoke-RestMethod @restParam
$configPolicies.value</code></pre>

As you can see, the code above is quite simple, and looking at the resultant data shows we get some basic data back showing all available **settings catalog** policies that are in our tenant (in our case just the one).<figure class="wp-block-image size-large">

<img loading="lazy" width="588" height="287" src="/assets/images/2021/03/image-2.png" alt="" class="wp-image-518" srcset="/assets/images/2021/03/image-2.png 588w, /assets/images/2021/03/image-2-300x146.png 300w" sizes="(max-width: 588px) 100vw, 588px" />  

Ok, so we've got the basic metadata of our policy - so let's grab the id from the previous call and dive in further..

<pre class="wp-block-code"><code lang="powershell" class="language-powershell">$policyId = $configPolicies.value[0].id #grabbing the id from the previous code block
$restParam = @{
    Method = 'Get'
    Uri = "$baseUri/configurationPolicies('$policyId')/settings"
    Headers = $authHeaders
    ContentType = 'Application/json'
}

$configPolicySettings = Invoke-RestMethod @restParam
$configPolicySettings.value</code></pre>

The code above returns data on all available settings that we configured in our policy..<figure class="wp-block-image size-large">

<img loading="lazy" width="1024" height="163" src="/assets/images/2021/03/image-3-1024x163.png" alt="" class="wp-image-519" srcset="/assets/images/2021/03/image-3-1024x163.png 1024w, /assets/images/2021/03/image-3-300x48.png 300w, /assets/images/2021/03/image-3-768x122.png 768w, /assets/images/2021/03/image-3.png 1367w" sizes="(max-width: 1024px) 100vw, 1024px" />  

if we drill in to one of the **settingInstance** objects, we should see more info..<figure class="wp-block-image size-large">

<img loading="lazy" width="926" height="240" src="/assets/images/2021/03/image-4.png" alt="" class="wp-image-520" srcset="/assets/images/2021/03/image-4.png 926w, /assets/images/2021/03/image-4-300x78.png 300w, /assets/images/2021/03/image-4-768x199.png 768w" sizes="(max-width: 926px) 100vw, 926px" />  

As we can see, this particular setting is for **allow warning for other disk encryption** - as clearly defined in the **definitionId** value. If we drill into the **choiceSettingValue** item, we will see the applied value and any other child properties within that setting..<figure class="wp-block-image size-large">

<img loading="lazy" width="1024" height="171" src="/assets/images/2021/03/image-5-1024x171.png" alt="" class="wp-image-521" srcset="/assets/images/2021/03/image-5-1024x171.png 1024w, /assets/images/2021/03/image-5-300x50.png 300w, /assets/images/2021/03/image-5-768x128.png 768w, /assets/images/2021/03/image-5.png 1393w" sizes="(max-width: 1024px) 100vw, 1024px" />  

Here we can see the value of **allow warning for other disk encryption** is set to 0 - or false, which correlates to our policy set from the endpoint portal.<figure class="wp-block-image size-large">

<img loading="lazy" width="863" height="215" src="/assets/images/2021/03/image-6.png" alt="" class="wp-image-522" srcset="/assets/images/2021/03/image-6.png 863w, /assets/images/2021/03/image-6-300x75.png 300w, /assets/images/2021/03/image-6-768x191.png 768w" sizes="(max-width: 863px) 100vw, 863px" />  

Here we can see the **child** setting of **allow standard user encryption** with the setting value of 1 - or true.

This example shows how simple it is to capture the basic building blocks of a settings catalog policy. But for those interested to dig deeper, why not check out what happens when you run the same example from above while expanding the **settingDefinitions** property..<figure class="wp-block-image size-full">

<img loading="lazy" width="1920" height="1080" src="/assets/images/2021/03/settingsDefinition.gif" alt="" class="wp-image-523" />  

Cool, huh? literally everything about each and every setting is available to us if we just spend the time to dig into graph a little bit!

## Building a policy from scratch

Now, before we begin, I'm going to put this out there - settings catalog policies are probably not the easiest things to build from scratch..

There is LOTS of metadata that you need to know for each setting before you can build out the policies. However, the concepts shown below can also be leveraged to maintain **reference templates** that can be captured and redeployed to other tenants to allow seamless management of multiple tenants with minimal effort.

Enough stalling, let's see what's required.

### Getting all settings data

So the first question that you may be asking, is, "How do I get the data that I need for the settings that I want to add to my catalog policy?" Luckily, Microsoft has an endpoint in graph that will return all possible settings currently available for the settings catalog.

We can capture all necessary metadata on those available settings by using the **deviceManagement/configurationSettings** endpoint.

<pre class="wp-block-code"><code lang="powershell" class="language-powershell">$restParam = @{
    Method = "Get"
    Uri = "$baseUri/configurationSettings"
    Headers = $authHeaders
    ContentType = 'Application/Json'
}
$settingsData = Invoke-RestMethod @restParam
$settingsData.value</code></pre>

Let's run the above code and see what we get back..

<figure class="wp-block-image size-full">

<img loading="lazy" width="1920" height="1080" src="/assets/images/2021/03/settingsCatapalooza.gif" alt="" class="wp-image-524" />  

Well&#8230; that was a bit much wasn't it! at the time of writing, there is around 2,100 settings available in the settings catalog library with more to come until it is at parity with all existing methods of device configuration (configuration items, ADMX templates, endpoint baselines etc).

Let's filter the settings by a setting **definitionId** that we know (notice that the definitionId isnt a GUID? welcome to the future&#8230;)

<pre class="wp-block-code"><code lang="powershell" class="language-powershell">$settingsData.value | where {$_.id -eq 'device_vendor_msft_bitlocker_allowwarningforotherdiskencryption'}</code></pre><figure class="wp-block-image size-large">

<img loading="lazy" width="1024" height="440" src="/assets/images/2021/03/image-7-1024x440.png" alt="" class="wp-image-525" srcset="/assets/images/2021/03/image-7-1024x440.png 1024w, /assets/images/2021/03/image-7-300x129.png 300w, /assets/images/2021/03/image-7-768x330.png 768w, /assets/images/2021/03/image-7-1536x659.png 1536w, /assets/images/2021/03/image-7.png 1635w" sizes="(max-width: 1024px) 100vw, 1024px" />  

Weird&#8230; doesn't that look the same as the expanded **settingsDefinitions** content from earlier? That's because it is literally the same data! We can dig into this data to find out the available options for each setting, but let's skip that for now and just build our example policy from scratch..

### Posting a settings catalog policy to Intune from Graph

Conceptually we now should understand what's required here. We have some metadata around what the policy is called to which we attach whichever settings we want attributed to our new policy profile. So let's rebuild the original policy in PowerShell!

<pre class="wp-block-code"><code lang="powershell" class="language-powershell">$baseUri = 'https://graph.microsoft.com/beta/deviceManagement/configurationPolicies'

#region build the policy
$newPolicy = [pscustomobject]@{
    name         = "Bitlocker Policy from PowerShell"
    description  = "we built this from PowerShell!"
    platforms    = "windows10"
    technologies = "mdm"
    settings     = @(
        @{
            '@odata.type'   = "#microsoft.graph.deviceManagementConfigurationSetting"
            settingInstance = @{
                '@odata.type'       = "#microsoft.graph.deviceManagementConfigurationChoiceSettingInstance"
                settingDefinitionId = "device_vendor_msft_bitlocker_allowwarningforotherdiskencryption"
                choiceSettingValue  = @{
                    '@odata.type' = "#microsoft.graph.deviceManagementConfigurationChoiceSettingValue"
                    value         = "device_vendor_msft_bitlocker_allowwarningforotherdiskencryption_0"
                    children      = @(
                        @{
                            '@odata.type'       = "#microsoft.graph.deviceManagementConfigurationChoiceSettingInstance"
                            settingDefinitionId = "device_vendor_msft_bitlocker_allowstandarduserencryption"
                            choiceSettingValue  = @{
                                '@odata.type' = "#microsoft.graph.deviceManagementConfigurationChoiceSettingValue"
                                value         = "device_vendor_msft_bitlocker_allowstandarduserencryption_0"
                            }
                        }
                    )
                }
            }
        }
        @{
            '@odata.type'   = "#microsoft.graph.deviceManagementConfigurationSetting"
            settingInstance = @{
                '@odata.type'       = "#microsoft.graph.deviceManagementConfigurationChoiceSettingInstance"
                settingDefinitionId = "device_vendor_msft_bitlocker_requiredeviceencryption"
                choiceSettingValue  = @{
                    '@odata.type' = "#microsoft.graph.deviceManagementConfigurationChoiceSettingValue"
                    value         = "device_vendor_msft_bitlocker_requiredeviceencryption_1"
                }
            }
        }
    )
}
#endregion
#region post the request
$restParams = @{
    Method      = 'Post'
    Uri         = $baseUri
    body        = ($newPolicy | ConvertTo-Json -Depth 20)
    Headers     = $authHeaders
    ContentType = 'Application/Json'
}
Invoke-RestMethod @restParams
#endregion</code></pre>

Once we run this - within seconds we should have a replicated policy in our tenant!<figure class="wp-block-image size-large">

<img loading="lazy" width="1024" height="351" src="/assets/images/2021/03/image-8-1024x351.png" alt="" class="wp-image-526" srcset="/assets/images/2021/03/image-8-1024x351.png 1024w, /assets/images/2021/03/image-8-300x103.png 300w, /assets/images/2021/03/image-8-768x263.png 768w, /assets/images/2021/03/image-8.png 1471w" sizes="(max-width: 1024px) 100vw, 1024px" />  

As mentioned earlier, building these from scratch is tricky - but if you read between the lines, knowing how to capture pre-built policies via graph and using the captured JSON payload to post the same policy to a new tenant (or a few hundred tenants) should make multi-tenant device management less painful.

— Ben