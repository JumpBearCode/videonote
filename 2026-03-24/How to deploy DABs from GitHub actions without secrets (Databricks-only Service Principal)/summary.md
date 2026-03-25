---
url: https://www.youtube.com/watch?v=cgRm2NpODuI
whisper_cost: $0.0000
parse_cost: $0.0000
summary_cost: $0.0017
total_cost: $0.0017
duration: 19m52s
summary_model: deepseek-chat
---

好的，作为一名深度内容分析专家，我已将视频转录内容转化为以下详尽的结构化笔记。

# 深度笔记：使用联合凭证安全地将代码从 GitHub 部署到 Azure Databricks

## 核心主题与目标受众
本视频的核心主题是演示如何在不使用个人账户、密码或任何密钥（Secrets）的情况下，通过**联合凭证（Federated Credentials）**，安全地将 GitHub 仓库中的代码自动部署到 Azure Databricks。其目标是解决传统服务主体（Service Principal）认证方式中密钥管理和泄露的风险。目标受众是使用 Azure Databricks 和 GitHub Actions 进行 CI/CD 的数据工程师、DevOps 工程师或平台管理员，他们关注部署流程的安全性和可维护性。

---

## 视频内容深度解析

### 第一部分：问题与动机——为何要改变现有部署方式？

视频首先设定了一个常见场景：代码存储在 GitHub 仓库，需要部署到目标 Databricks 工作区。

*   **传统方法的弊端**：过去通常使用在 Microsoft Entra ID（原 Azure Active Directory）中定义的服务主体，并配合一个**密钥（Secret/Password）** 进行认证。虽然这种方法有效，但存在两大核心问题：
    1.  **安全风险**：密钥可能泄露，被恶意利用以干扰或破坏你的环境。
    2.  **运维负担**：密钥具有**有效期**，必须定期轮换（Rotate）。人们常常忘记更新部署管道中的密钥，导致部署失败。

*   **新方案的目标**：因此，我们需要一种全新的方法，目标是**完全跳过使用密码**，同时确保流程是**安全**的。这就是引入**联合凭证**的动机。

### 第二部分：解决方案概述——联合凭证如何工作？

视频提出了一个基于“信任关系”的解决方案模型。

*   **核心组件**：
    1.  在 **Azure Databricks** 中创建一个新的服务主体。
    2.  为该服务主体创建一个**联合策略对象（Federation Policy）**。

*   **工作原理**：
    *   联合策略将服务主体与特定的 **GitHub 仓库** 和 **GitHub 环境（如 dev, prod）** 绑定。
    *   当 GitHub Actions 工作流运行时，GitHub 和 Databricks 之间会进行**令牌（Token）交换**。
    *   **关键优势**：整个过程无需在代码或配置中明文提供任何密钥或密码。访问权限被严格限定在策略所定义的范围内（特定仓库、特定环境）。

### 第三部分：实战演示——从零开始配置安全部署

演示从一个简单的 GitHub 仓库和空的 Databricks 工作区开始。

#### 步骤 1：初始化 GitHub Actions 工作流（并验证其会失败）
1.  **代码结构**：仓库中包含一个简单的 Databricks 资产捆绑包（Databricks Asset Bundle），定义了一个名为 “hello world” 的作业，该作业仅运行一个打印 “Hello World” 的 Python 脚本。
2.  **创建工作流**：在 GitHub 中创建一个基础工作流文件（`.github/workflows/deploy.yml`），并添加部署 Databricks 资产捆绑包的核心步骤：
    *   `安装 Databricks CLI`
    *   `databricks bundle validate`
    *   `databricks bundle deploy`
3.  **预期失败**：提交此工作流后，运行**果然失败**，错误信息显示“无法配置默认凭据”。这验证了在未配置任何认证的情况下，任何人都无法随意部署到 Databricks，这是安全的基本要求。

#### 步骤 2：在 GitHub 端创建环境（Environments）
*   **目的**：为了在联合策略中实现更精细的权限控制（例如，区分开发和生产）。
*   **操作**：在 GitHub 仓库设置中，创建两个环境：`dev` 和 `prod`。本演示将主要使用 `dev` 环境。

#### 步骤 3：在 Azure Databricks 端配置身份与策略
这是一个关键环节，所有操作都在 Databricks 管理控制台完成。

1.  **创建服务主体**：
    *   进入 **Databricks 账户控制台**（需要账户管理员权限）。
    *   在“用户管理”下，**创建新的服务主体**，命名为 `databricks-managed-sp-dev`。这个身份将成为部署操作的执行者。

2.  **创建联合凭据策略**：
    *   切换到“凭据和密钥”选项卡。
    *   为刚创建的服务主体**新建一个策略**，并进行详细配置：
        *   **提供程序**：选择 `GitHub Actions`。
        *   **GitHub 组织**：填写你的组织名称（从仓库 URL 获取）。
        *   **GitHub 仓库**：填写具体的仓库名称（从仓库 URL 获取）。
        *   **作用域限制**：选择使用 **GitHub 环境** 进行限制。这里指定仅允许从 `dev` 环境使用此服务主体。
    *   **策略效果**：此策略意味着，**只有**来自指定组织、指定仓库，并且在 `dev` 环境下运行的 GitHub Actions，才能获取令牌来扮演这个服务主体。

3.  **将服务主体添加到工作区**：
    *   返回到你的 **Databricks 工作区**（非管理控制台）。
    *   在“设置” -> “身份和访问” -> “服务主体”中，**添加**刚才在账户层面创建的服务主体 (`databricks-managed-sp-dev`)。
    *   **目的**：这使得该服务主体在你具体的开发或生产工作区中可见并可用。

#### 步骤 4：修改 GitHub Actions 工作流以使用联合认证
需要对工作流文件进行两处关键修改，以启用联合身份验证。

1.  **添加权限**：在工作流定义中，添加 `permissions:` 块，包含 `id-token: write`。这是允许工作流从 GitHub 的 OIDC（OpenID Connect）提供商请求 JSON Web 令牌（JWT）所必需的。
2.  **配置环境与认证**：
    *   **指定环境**：使用 `environment: dev`，以匹配我们在联合策略中设置的限制。
    *   **配置 Databricks CLI 认证**：在部署步骤中，通过环境变量设置认证方式：
        *   `DATABRICKS_AUTH_TYPE：github-oidc` （告知 CLI 使用 GitHub OIDC 流程）
        *   `DATABRICKS_HOST：https://<你的-workspace-url>` （目标 Databricks 工作区地址）
        *   `DATABRICKS_CLIENT_ID：<服务主体的应用程序(客户端) ID>` （从 Databricks 账户控制台的服务主体详情中获取，**这不是密钥，而是公开标识符**）

#### 步骤 5：验证部署成功与策略有效性
1.  **成功部署**：提交更新后的工作流。此次运行成功通过认证步骤，`databricks bundle deploy` 成功执行。在日志中可以看到，当前登录身份显示为我们创建的服务主体 `databricks-managed-sp-dev`。在 Databricks 工作区的“作业”页面中，可以看到由该服务主体创建的 “hello world” 作业，运行后成功输出 “Hello World”。
2.  **策略约束验证（安全测试）**：
    *   **测试1：切换环境**：将工作流中的 `environment: dev` 改为 `environment: prod` 并运行。**部署失败**，错误提示“无效的授权”。这证明了联合策略成功限制了该服务主体只能在 `dev` 环境下使用。
    *   **隐含测试**：视频指出，如果有人将此工作流配置复制到其他仓库，同样会失败，因为策略限定了仓库名称。这确保了权限不会被滥用或意外扩散。

### 第四部分：方案总结与适用范围

*   **本方案的优势**：
    1.  **无密钥管理**：彻底消除了密码的存储、轮换和泄露风险。
    2.  **精细的权限控制**：通过绑定仓库和环境，实现了最小权限原则，访问范围被严格限定。
    3.  **简洁安全**：整个配置清晰、优雅且安全。
*   **本方案的局限性**：
    *   此方案中创建的服务主体是 **Databricks 托管**的。因此，GitHub Actions **只能访问和操作 Databricks 资源**，无法用于部署其他 Azure 服务（如 Data Factory, Logic App 等）。
*   **扩展方案提示**：如果需要在一个工作流中同时部署 Databricks 和其他 Azure 服务，视频推荐了另一种“企业级”方案，即使用 **Entra ID 托管的服务主体** 并配置联合凭证，这需要在 Entra ID 中进行策略配置。

---

## 核心要点总结

本视频的核心 **Takeaway** 是：通过利用 **Azure Databricks 中的联合凭证**，可以构建一个既安全又易于维护的从 GitHub 到 Databricks 的 CI/CD 管道。该方法的核心价值在于用基于**信任关系（OIDC 令牌交换）和策略绑定（限定仓库与环境）** 的认证模型，取代了传统的、存在泄露和运维负担的静态密钥认证。它不仅解决了密钥管理的痛点，还通过精细的访问控制极大地提升了整体部署流程的安全性。对于纯 Databricks 部署场景，这是一个首选的最佳实践。

---

## Transcript

Hey there. So let's say you implemented your Databricks code,

you store it in a GitHub repository,

and now you would like to deploy it to

Azure Databricks in a secure way without using

your personal accounts or any passwords or secrets.

In this episode, I will show you how to do this

securely using federated credentials defined in Azure Databricks.

So let's get started.

Alright, so let me show you the scenario. So basically we have a GitHub repository with our code and I have a target Databricks workspace to which I would like to deploy stuff.

So basically my goal is as follows. I want to grab code from the GitHub repository and deploy it to Databricks. And you might say, all right, but we've been doing this already using a service principle defined in Entra with some secret. And it worked, right? So why should we bother with a different approach?

And the reason is, yes, this approach works, but it has two big issues.

So first of all, a secret, a password is used.

That might leak and someone might use it in a malicious way to mess around with your environment.

Which is a risk, right?

The second thing is that those secrets, they have expiration period.

And they have to be rotated periodically.

and then update it in your deployment pipeline.

And very often people forget to do so,

which results in your deployment to start failing.

And that's why we want to use a different method.

We want to skip using those passwords entirely.

And at the same time, we want it to be secure.

And that's where federated credentials will be very useful.

So what I will do in this episode will be to create a new service principle inside Azure Databricks, now in Entra, in Databricks, and then I will use it during the deployment phase.

But now, how to make GitHub to be able to use this service principle? Well, that's where we'll create a federation policy object tied with a service principle.

that will define it it will allow github to use this particular service

principle but only from my github repository and only when using specific

github environment like dev and prod and in this scenario tokens will be

exchange between github and Databricks and it will work without providing any

secrets or any passwords explicitly. So let me show you that in action. So I have

a github repository, very simple one, this federated identity Databricks. It

It contains a very simple code that I cloned to my local machine.

And basically what I have here is a job definition,

Databricks job, this hello world one,

that contains only a single Python step

that runs this hello py script

that in turn prints hello world statement.

And I configured a very simple Databricks as in bundle

use to deploy this job to a target environment.

A very simple scenario.

Now, in my GitHub repository,

I don't have any actions defined yet

to deploy my stuff to Databricks.

So let's start with doing this.

And to avoid creating everything from scratch,

let me use this very simple template.

So let me find it.

It's called simple workflow. In our case it is fine. We'll adjust it in a second. Let's give it a name and let's verify what it does. So basically it triggers on a post to main branch, pull request to main and it can be run manually. That's fine.

Then it runs on some Linux machine it gets the repo and it prints some hello world statements which we don need actually So let me remove this oh or let me remove all of this and that it and let commit those changes and please know that i doing this directly on the main branch

you shouldn't do this in action in real life but for demo proposals this is fine

and because I made a commit it already resulted in my github action being triggered and actually

it went fine it is green it didn't do anything yet with my database code but at least we can see that

the trigger works so let's jump to vs code let me pull those changes and let's update the workflow

definition. And now we know that to deploy Databricks as in bundles we have to first

of all install Databricks CLI and then run this Databricks bundle validate and

Databricks bundle deploy commands. So let's add them to our workflow definition.

So let me just copy them

and paste it.

Alright, so here it is.

Install Databricks CLI,

check who is logged into Databricks to Databricks CLI

and then validate and deploy my Databricks asset bundle.

So let me commit this to the repository

like added Databricks steps

and let's push it.

that should trigger the pipeline, deployment pipeline,

that is running.

And actually at this moment, it should fail, right?

Because we didn't specify any authentication yet.

So it would be very bad if anyone could just deploy

and define a deployment workflow to connect

and to deploy stuff to my database environment.

So we expect this to fail.

And it did, it failed.

With error that default authentication,

I cannot configure default credentials, blah, blah, blah.

Basically, it means that we are not authenticated.

So let's fix this.

So following our design, we would like to make it secure.

So first of all, in my GitHub environment,

in my repository, I will create two environments,

dev and prot.

And that's basically what you do in your projects

because you might use different variables,

different secrets per environment,

or you might have a different protection rules

per environment.

And we'll use those environments

in our federation policy in a second.

So let me define those environments

like dev and prot.

I will not configure them in any way

because that's not necessary at this point.

We just need them to exist.

So we have dev and prot, and I will be using dev1.

The next step is to create our service principal

in Azure Databricks, not in Entra, in Azure Databricks.

This scenario focuses on doing everything inside Databricks.

And by the way, that's my Databricks workspace.

And we can see that at this moment,

there are no jobs deployed yet.

I mean, I have some leftovers from previous demos,

but we don't have this hello world job.

So to add a new service principal,

I will switch to Databricks account console.

this view that is available only to Databricks account admins, so it requires high privileges.

Then in user management, I will just create a new service principal and let's call it

Databricks managed service principal dev as follows.

And yes, please create this.

Alright, so we have a service principal, an identity in context of which our deployment will be executed.

But we don't have this federation policy yet.

This thing that allows GitHub to use this credential.

And to make it work, we've got to create it, right?

So let's jump into credentials and secrets tab.

And let create a new policy for this service principal We got to select provider that will use this policy And in our case it is GitHub Actions

because we want to use this service principle

from GitHub Actions.

Then we've got to provide a name of our GitHub organization.

In my case, it is Table on Azure.

You can see this in a URL to your repo.

So let me copy this.

Then we've got to provide a repository that will be allowed to use our service principal.

And in my case, it is federated identity Databricks.

Again, from URL.

So let me paste this.

Then we've got a choice that allows us to narrow down scenarios in which this policy and the service principle can be used.

I want to use environments. That's why we defined them previously.

and basically I want to define that this service principle will be allowed to be used only from

this repository on from this organization and only when deploying stuff to github dev environment

that's what we are configuring here and the rest of the stuff is fine and let me create the policy

and that's it

the next thing to do is to basically create this service principle

in my Databricks workspace

so let's jump back to my workspace

and here in the settings

in identity and access, service principles

I don't have my service principle available in this workspace yet

so let me add it

that's the name of my service principal

and yes please add it. So what I did right now

was to make this Databricks managed

dev service principal available

in my target Databricks workspace

right and that's it. So we have

everything configured on the Databricks

site. The next thing is that we have got to modify our workflow. We've got to instruct

GitHub that hey please use this federated identity, those federated credentials to authenticate.

And basically we've got to make two changes. So first of all we've got to add some permissions

to the workflow.

to be able to exchange some tokens with Databricks.

That's required.

So that's one thing.

And secondly, we've got to indicate some variables

that will be used by the workflow as follows.

So first of all, please use dev GitHub environment,

this configuration, why?

Because we know that our Federation policy

we've just defined, it applies only

when using this dev environment.

Secondly, we defined this GitHub open ID connect

as authentication type.

It tells GitHub, hey, use this federated identity stuff.

I indicated where is my Databricks host located

and we've got to provide client ID of the service principle

that will be used for the login proposals and we can get this from here that's the application id

so let me copy this let me paste it here and that should be fine so let me

push this to the repo add that open id connect stuff and hopefully everything will work

so let's jump back to our github repository to actions and let's see if this time deployment

will pass and you can see that we already passed the step that previously failed right

and actually let's see who is locked and oh please we can see that display name

is set to our service principle that we defined in Databricks So GitHub action uses this service principle credentials

to connect to Databricks.

So everything looks fine, right?

Then we were able to deploy,

to validate our Databricks as a bundle

and deploy it, right?

Everything went fine.

our our action faces green checkbox so let's verify this let's jump to other database workspace

let's see jobs and pipelines and yes we've got this hello world pipeline that was deployed and

created by using this databricks managed service principal and let me run it to verify that

everything works and inside we can see that that yes um the creator and the runner's identity is the

um principle that we created in databricks and when it comes to tasks it has this very simple

single um python script task right so everything looks fine and about execution

it was executed and it printed hello world statement so everything works fine

one thing to check all right it works and please know that i didn't specify any secret

anywhere right no password this thing that might look like some secret stuff it is just an

identifier of my service principal so it is not a secret.

You can safely push this to the repository.

And let's say

what will happen if we try to use the same

service principal but on a production environment.

And let's see

if it will work. Changed mf2-prot.

This is to verify if our federation policy indeed restricts the usage of the service principal to the dev environment only.

And let's verify this in actions. Hopefully it will fail. And yes, it failed.

it failed because invalid grant is defined right basically and the stuff

that we configured in GitHub action it doesn't match the stuff we configured in

our Federation policy which simply means that it is not possible to overuse those

permissions right just to copy this workflow definition and use it in a other repository

it will fail as well right because the federation policy limits the users of the service principle

to this repository only the same if we change the environment to prod right so basically

by using this federated credentials what we achieved so first of all no secrets no passwords

nothing is stored there is nothing we have to rotate periodically it simply works it is clean

it is elegant it is secure secondly those permissions are narrowed down to be used only

from my repository and only when deploying to dev environment so it is not possible just to grab

the code copy it and use it in another place just hoping that i would get access to

to my environment no it's empty words please be aware that in this scenario

we used a service principle defined in azure databricks so our github workflow

will be able to access only Databricks.

It will not be able to go outside,

like to deploy anything to Azure Data Factory,

some logic app, or whatever you want, right?

This scenario, it focuses only on doing stuff

inside Databricks.

If you would like to use a more enterprise version

in which you still use federated stuff,

but you can deploy more things than .databricks,

then check out another video inside,

which I talk about how to do this

using Entra Managed Service Principle.

And basically that's it for today.

Thanks for watching and see you next time.

Take care.
