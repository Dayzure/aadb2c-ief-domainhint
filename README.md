# AAD B2C Identity Experience Framework - pass domain_hint parameter
## Background
Let's say you are building a web experience which should be able to authenticate social users (i.e. facebook, or local accounts).
In addition to that you also would like to white label this solution and offer it as a branded SaaS offering to business partners 
and/or business clients. 
And your business partners should be able to login with their corporate accounts. More over, in a white labled solution you 
should probide a direct link to the login experience your business partner already has. And avoid any home realm discovery screens.

## Home realm discovery screen
If you wonder what is home realm discovery screen - this is the screen on which you ask your web experience users to choose which i
dentity provider to use for authentication. In a white label scenario, you really want to avoid that, and directly sign-in the users
with their IdP, as you already know which that is.

## Azure Active Directory Authorisation request query string parameters
Unfortunatelly it is a less known feature of the Azure AD, that you can provide two addition parameters for the Authorisation request -
the domain_hint and the login_hint. These parameters are [well described here](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-protocols-oauth-code) (search for "domain_hint"). 

## The challenge
Well, as we said - wa want everything to start from a B2C tenant. And you can still use the domain_hint in a B2C tenant. 
But the meaning of that parameter in the context of B2C translates to the [Technical Profile defined in your custom policy](https://docs.microsoft.com/en-us/azure/active-directory-b2c/active-directory-b2c-reference-trustframeworks-defined-ief-custom).
In order to understand the challenge, I would refer first to the well described option to include [Azure AD (a "normal" or "corporate" one)](https://docs.microsoft.com/en-us/azure/active-directory-b2c/active-directory-b2c-setup-aad-custom) 
[as Identity Provider in a B2C tenant](https://docs.microsoft.com/en-us/azure/active-directory-b2c/active-directory-b2c-setup-aad-custom).

Here is the catch - when you provide domain_hint to your authorisation request for B2C, in order the B2C to automatically redirect 
the user to the corporate AD, that domain_hint is valid only for the first request (to B2C) and never gets to the corporate Azure AD.

## The solution
The solution is relatively simple, and once you understand how the Identity Experience Framework works, it will be just a "but of course,
it is as simple as that"!

Again, please read and understand (and at best create a working sample for yourself) how you federate to Azure AD from B2C tenant.
In this sample, I am only showing what you should do additionally to provide a domain_hint to the "real" Azure AD.

There are three things which we should do.

### Add a new claim in your claims schema
The claims schema is on top of your TrustFrameworkExtensions.xml file. You would add a new claim like that:
```xml
<BuildingBlocks>
    <ClaimsSchema>
        <ClaimType Id="domain_hint">
            <DisplayName>Domain Hint for AAD</DisplayName>
            <DataType>string</DataType>
            <UserHelpText/>
        </ClaimType>
    </ClaimsSchema>
</BuildingBlocks>
```  
This claim will later need in the technical profile

### Add a new claims transformation to populate this claim value
We can populate this claim value from the initial authorisation request to the B2C. We can get additional query string parameters 
like that:
```xml
<BuildingBlocks>
	...    
	<ClaimsTransformations>
	  <ClaimsTransformation Id="CreateDomainHintClaim" TransformationMethod="CreateStringClaim">
		<InputParameters>
		  <InputParameter Id="value" DataType="string" Value="{OAUTH-KV:dh}" />
		</InputParameters>
		<OutputClaims>
		  <OutputClaim ClaimTypeReferenceId="domain_hint" TransformationClaimType="createdClaim" />
		</OutputClaims>
	  </ClaimsTransformation>
	</ClaimsTransformations>
</BuildingBlocks>
```  
We use the special {OAUTH-KV:dh} value to indicate that we would like to read a query string value where the key woud be **dh**

### Use the new claim in the techical profile
Finally, we use the newly created claim transformation and claim in the technical profile as an **input claim** like that:

```xml
<InputClaimsTransformations>
    <InputClaimsTransformation ReferenceId="CreateDomainHintClaim" />
</InputClaimsTransformations>
<InputClaims>
    <InputClaim ClaimTypeReferenceId="domain_hint" />
</InputClaims>
```

A more complete view on the technical profile is here:
```xml
<ClaimsProvider>
    <Domain>Corporate-Domain-Name</Domain>
    <DisplayName>Login using CORPORATE-DOMAIN</DisplayName>
    <TechnicalProfiles>
        <TechnicalProfile Id="CorporateDomainProfile">
            <DisplayName>CorporateDomain Employee</DisplayName>
            <Description>Login with your CorporateDomain account</Description>
            <Protocol Name="OpenIdConnect"/>
            <OutputTokenFormat>JWT</OutputTokenFormat>
            <Metadata>
                <Item Key="METADATA">https://login.windows.net/CorporateDomain.onmicrosoft.com/.well-known/openid-configuration</Item>
                <Item Key="ProviderName">https://sts.windows.net/TENANT-ID-FOR-CORPORATE-DOMAIN/</Item>
                <Item Key="client_id">THE-CLIENT-ID</Item>
                <Item Key="IdTokenAudience">AGAIN-THE-CLIENT-ID</Item>
                <Item Key="response_types">id_token</Item>
                <Item Key="UsePolicyInRedirectUri">false</Item>
            </Metadata>
            <InputClaimsTransformations>
	            <InputClaimsTransformation ReferenceId="CreateDomainHintClaim" />
            </InputClaimsTransformations>
            <InputClaims>
              <InputClaim ClaimTypeReferenceId="domain_hint" />
            </InputClaims>
        </TechnicalProfile>
    </TechnicalProfiles>
</ClaimsProvider>
```

A complete [TrustFrameworkExtensions.xml](./TrustFrameworkExtensions.xml) file is located [here in this repository](./TrustFrameworkExtensions.xml). Be aware - here I am only showing you what you need
to change, not how you can add Azure AD as Identity Provider in a B2C tenant.

### The authorisation request
Finally your authorisation request to B2C should include both **domain_hint** and **dh** query string parameters.
The **domain_hint** should have the value you put under `<Domain>` in your technical profile. This will instruct B2C that the request
is actually for the corporate Azure AD. And the second (**dh**) will translate into a query string **domain_hint** value for the 
corporate Azure AD: `&domain_hint=Corporate-Domain-Name&dh=azure-ad-domain-name`
At the end, your authorisation request will look something like that:
```
  https://login.microsoftonline.com/b2c-tenant.onmicrosoft.com/oauth2/v2.0/authorize?p=sign-up-in-policy&client_id=valid-client-id&nonce=defaultNonce&redirect_uri=https%3A%2F%2Flocalhost%2F&scope=openid&response_type=id_token&prompt=login&domain_hint=Corporate-Domain-Name&dh=azure-ad-domain-name
```
## Important
When you register an Application in your **corporate** Azure AD - to be the trust endpoint for the B2C requests, make sure you
flag it as being [multi tenant](https://docs.microsoft.com/en-us/azure/active-directory/application-dev-setup-multi-tenant-app).
Thus you will not have to create a trust relationship with all your partners - but just with your app.
Again, please read follow all the links under [multi tenant apps](https://docs.microsoft.com/en-us/azure/active-directory/application-dev-setup-multi-tenant-app) to fully understand
what multi tenant apps are and how the authentication works!
