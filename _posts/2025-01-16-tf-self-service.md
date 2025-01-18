---
title: "Structuring a Self-Service Platform with Terraform"
last_modified_at: 
categories:
  - Blog
tags:
  - Terraform
  - IaC
  - IDP
---

I recently read an excellent blog post on [Building Self-Service Platforms with Terraform at Scale ](https://sunil-tailor.medium.com/building-self-service-platforms-with-terraform-at-scale-396c208a808d) and found it really well written, thoughtful, useful, and engaging. This is a topic I've spent a lot of time recently thinking about, as it comes up often in my role as a Resident Solutions Architect at [HashiCorp](https://hashicorp.com). 

The author does an excellent job of framing the discussion around self-service Internal Developer Platforms, describing the need for an IDP and how it’s used. The key insight he has is that creating an ___abstraction layer___ (in this case a [YAML](https://yaml.org/) data model) between Terraform and an application team making an infrastructure change request can have a number of benefits. 

Many of the customers I work with have found that it’s not practical to ask application teams to interact with an infrastructure change management system via Terraform. A few highly skilled teams might be able to accommodate this request, but at most large enterprises, teams like that are the exception rather than the rule.

![Fig. 1 - Confused by Terraform](/assets/images/TF%20Confused.png)
<div style="text-align: center; font-size: 14px;">Fig. 1 - Confused by Terraform</div>
<br/>

If a developer is not already familiar with Terraform, asking them to propose a pull request using Terraform code adds a lot of complexity, and will at the very least result in the platform team needing to support the developer. This defeats the purpose of a _self-service_ platform. YAML isn’t exactly plain English, but it’s a lot easier to work with for less-technical teams. At the end of the day, it’s structured data, which means it can be documented, templated, and validated, allowing the platform engineering team to help ensure the application team’s success.

![Fig. 2 - YAML Success](/assets/images/YAML%20Yes.png)
<div style="text-align: center; font-size: 14px;">Fig. 2 - YAML Success</div>
<br/>

Additionally, the article talks about the benefits of having a formal data model schema, in that a validation pipeline can easily understand the intent of what the requestor is trying to do. A pipeline can validate that the request makes sense from a technical perspective and complies with company policies. 

Underemphasized in the article, however, is that automation systems that sit in front of the request process can easily work with structured data in YAML (or another machine-friendly language like JSON), whereas coding an automated process to work with Terraform HCL is cumbersome. Automation becomes especially important when there’s a process enforced in order to even open a pull request, involving one or more workflows. For example, an enterprise might want to force all workflows to start in ServiceNow for issue tracking and support purposes. Some enterprises just want to centralize their self-service operations on an existing platform.

![Fig. 3 - Enterprise Automation for Approval and Logging](/assets/images/ServiceNow.png)
<div style="text-align: center; font-size: 14px;">Fig. 3 - Enterprise Automation for Approval and Logging</div>
<br/>

## The Problem

There’s a subtle issue with the author’s suggestion that the `request -> validate -> approve -> process` workflow be broken across two different repositories, but it’s an issue that I feel has large implications. The problem is that the author has conflated the __means by which a team makes a request__ and the __making of the request itself__.

![Fig. 4 - Two Terraform States](/assets/images/Two%20States.png)
<div style="text-align: center; font-size: 14px;">Fig. 4 - Two Terraform States</div>
<br/>

A file represented in the data abstraction layer _is not in itself a request_. It’s an alternate, simplified representation of the infrastructure model. The process of submitting a request to change infrastructure, regardless of whether it’s represented by Terraform or YAML, should be made through a pull request (hence the name). In a system where requests are stored in their own Terraform state, a pull request happens before a request is made. It should be the other way around.

![Fig. 5 - The Cart Before the Horse](/assets/images/Two%20States%20Process.png)
<div style="text-align: center; font-size: 14px;">Fig. 5 - The Cart Before the Horse</div>
<br/>

Let’s say for the sake of argument that we want the HEAD of the main branch of the requests repository to represent a request to change infrastructure. There’s already an inconsistency in the model, because it’s the delta between what’s in the Terraform state of the requests repository and the Terraform state of the processor repository that represents the actual set of infrastructure change requests to be applied. 

## What Could Go Wrong?

What happens if the plan or apply in the processor repository fails in some unrecoverable way? It’s true that the change request got onto the main branch by passing a validation pipeline, but there’s always the possibility that something got missed, or there’s some conflict in the infrastructure environment, that causes the requested changes to be rejected on Terraform plan or apply. 

What happens now? The request is in a state where it’s been approved and validated but now it needs to be rejected and reversed. This means that the YAML in the requests repository has to change. Without this step, the processor repository’s automation will forever be stuck in a loop, trying and failing to apply requested changes. In order to recover, the platform team and the application team need to work together to debug why the requests state doesn’t translate into a valid infrastructure state, and come up with a new change request that will fix the issue.

![Fig. 6 - Error Loop with Manual Recovery](/assets/images/Two%20State%20Errors.png)
<div style="text-align: center; font-size: 14px;">Fig. 6 - Error Loop with Manual Recovery</div>
<br/>

## Where’s My Request?

The problem with this model is that the requests repository doesn’t actually capture requests to change the infrastructure. It actually captures _approved changes to the infrastructure_. The clue that this is the case is that the “requests” are actually saved in Terraform state. Terraform state is not the appropriate place to model a step in a change request process. That function is already capably filled by the pull request workflow. A developer can make a request, wait for feedback and validation, and the request can be approved or rejected by a third party. Terraform state tracks Terraform’s understanding of how infrastructure looks in real life, and is used to apply just the required changes when infrastructure-as-code changes.

With the two-repository model of requestor / processor, there’s the risk of drift between the Terraform state of the processor and the Terraform state of the requestor, but no ability for the processor to provide feedback of why or how to remedy the situation. 

## A Better Way

The author is on the right track in that a data abstraction layer should be used to codify and enforce design and policy decisions, as well as act as an interface for automation. However a solution that separates the requesting and processing code into two separate repositories adds unnecessary complexity and fragility into the system. 

For separation of concerns, it’s a good idea to have the code that defines the infrastructure YAML and the code that interprets it as Terraform live in two separate repositories.

Let’s put this in terms of object oriented programming. Requestor and processor repositories operate by ___aggregation___, where the processor repository is aware of the requestor repository but not the other way around. Both processor and requestor can exist independently, and they are coupled together to complete the workflow. This loose coupling poses a problem because there is in fact mutual dependency between the two repositories:

The processor repository refers to the state of the requests repository
The requests repository assumes that all of its requests will be successfully processed but has no way of verifying this assumption.

![Fig. 7 - Aggregation](/assets/images/Aggregation.png)
<div style="text-align: center; font-size: 14px;">Fig. 7 - Aggregation</div>
<br/>

The relationship between the processor and the requestor repositories would be better represented with ___composition___. This means that the code that turns YAML into Terraform should be maintained by the platform team in its own repository, and is used as an external module by the application team that maintains their own infrastructure repository. This frees up the platform team to release new versions of the base module via their own development lifecycle, and the consuming teams can take new versions of it when they’re ready. 

![Fig. 8 - Composition](/assets/images/Composition.png)
<div style="text-align: center; font-size: 14px;">Fig. 8 - Composition</div>
<br/>

The requestor repository then becomes a _Composite_ Terraform module while the processor repository becomes a _Component_ Terraform module. This relationship is formalized by the “parent” module (the one that contains the YAML files) calling the “child” module (the module that turns YAML into Terraform). The system is composite because the base module can’t exist on its own. Its only purpose is to be included in an infrastructure repository maintained by an application team.

In the composite architecture, a requestor repository becomes known as an “environment” repository, in that it captures the IaC definition of a particular environment’s infrastructure resources. In this case, the environment would be something like “Vault Dev.”

![Fig. 9 - One Terraform State](/assets/images/One%20State.png)
<div style="text-align: center; font-size: 14px;">Fig. 9 - One Terraform State</div>
<br/>

The process to request an infrastructure change to a particular environment is:
- Check out an environment repository
- Open a PR to propose a change to a YAML file (e.g. add a KV engine to Vault)
- The PR build validates the YAML according to the data model and company policy
- If validation passes, the build goes on to perform a Terraform Plan to see the effect that the change would have on the infrastructure.
- If needed, the platform operations team comments on the PR and requests changes 
- The platform operations team approves the PR and merges the PR branch to main. 
- The main branch build runs Terraform apply and changes the infrastructure based on the Terraform resources generated by the base module from the YAML files.

![Fig. 10 - GitOps](/assets/images/One%20State%20Process.png)
<div style="text-align: center; font-size: 14px;">Fig. 10 - GitOps</div>
<br/>

Now, what if something goes wrong with TF apply? The build will fail, and the platform ops team is alerted to fix the situation. Without getting the application team involved, the platform ops team can get back to a known good state by rolling back to the commit before the merged PR. This process can even happen automatically. This recovery process keeps the YAML description lined up with the Terraform state, and allows the application team time to understand the error and debug their code while the build is not broken.

![Fig. 11 - Automated Recovery](/assets/images/One%20State%20Errors.png)
<div style="text-align: center; font-size: 14px;">Fig. 11 - Automated Recovery</div>
<br/>

## Conclusion

In short, using YAML as a data abstraction layer is a great idea which brings automation, auditability, compliance, and clarity to self-service internal developer platforms. However, modeling requested changes to infrastructure in a repository separate from the infrastructure itself introduces unnecessary complexity and fragility to the system. By architecting the relationship between components using composition instead of aggregation, we can reap the benefits of a data abstraction layer on top of Terraform while leveraging the built-in concept of pull requests via GitOps to model the requesting of changes to the infrastructure.

