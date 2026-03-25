---
url: https://www.youtube.com/watch?v=cgRm2NpODuI
whisper_cost: $0.0000
parse_cost: $0.0020
summary_cost: $0.0018
total_cost: $0.0038
duration: 19m52s
summary_model: deepseek-chat
---

# 深度笔记：使用联合凭证安全地将代码从 GitHub 部署到 Azure Databricks

## 核心主题与目标受众
本视频的核心主题是**演示如何在不使用个人账户、密码或任何密钥（secrets）的情况下，安全地将 GitHub 仓库中的代码部署到 Azure Databricks**。视频介绍了一种利用 Azure Databricks 中定义的**联合凭证（federated credentials）**来实现这一目标的方法。目标受众是**使用 Azure Databricks 和 GitHub Actions 进行 CI/CD 的数据工程师、DevOps 工程师或平台管理员**，他们希望提升部署管道的安全性并简化密钥管理。

## 视频内容深度解析

### 1. 问题背景与动机：为何要改变现有方法？
视频首先设定了一个常见场景：代码存储在 GitHub 仓库，需要部署到目标 Databricks 工作区。

*   **传统方法的弊端**：过去通常使用在 Entra（原 Azure Active Directory）中定义的服务主体（service principal）并配合密钥（secret）进行认证。虽然可行，但存在两大问题：
    1.  **安全风险**：使用的密码或密钥可能泄露，被恶意利用来破坏环境。
    2.  **运维负担**：这些密钥有有效期，需要定期轮换（rotate）。人们常常忘记更新部署管道中的密钥，导致部署失败。

*   **新方法的目标**：完全摒弃使用密码，同时确保流程安全。**联合凭证**正是解决这些痛点的关键。

### 2. 解决方案概述：联合凭证如何工作？
新方案的核心是创建一个**与 Databricks 工作区关联的服务主体**，并通过一个**联合策略（federation policy）** 将其与 GitHub 绑定。

*   **核心组件**：
    *   **Databricks 服务主体**：在 Azure Databricks 账户控制台（Account Console）中创建的身份，部署操作将在此身份下执行。
    *   **联合策略对象**：链接到上述服务主体。它定义了**允许谁（GitHub）、在什么条件下**使用这个服务主体。
*   **工作原理**：策略将使用权限**精确限定**在特定的 GitHub 组织、仓库，甚至可以细化到特定的 GitHub 环境（如 `dev`, `prod`）。当 GitHub Actions 工作流运行时，GitHub 和 Databricks 之间会进行令牌（token）交换，整个过程**无需显式提供任何密钥或密码**。
*   **安全边界**：这种方法不仅消除了密钥管理，还通过策略实现了精细的访问控制，防止凭证被滥用（例如，从其他仓库或环境调用）。

### 3. 演示步骤详解

#### 第一阶段：初始设置与失败验证
1.  **GitHub 仓库与代码**：演示者有一个简单的 GitHub 仓库，包含一个 Databricks 资产捆绑包（Databricks Asset Bundle）定义，用于部署一个打印“Hello World”的 Databricks 作业（Job）。
2.  **创建基础 GitHub Actions 工作流**：
    *   使用了一个简单的模板工作流，触发条件为推送到 `main` 分支或拉取请求（PR）。
    *   在工作流中添加了部署 Databricks 资产捆绑包的关键步骤：
        ```yaml
        - name: Install Databricks CLI
          run: curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
        - name: Check Databricks Auth
          run: databricks auth whoami
        - name: Validate Bundle
          run: databricks bundle validate
        - name: Deploy Bundle
          run: databricks bundle deploy
        ```
3.  **预期中的失败**：提交此工作流后，部署**如期失败**，错误信息显示“无法配置默认凭据”。这验证了没有有效认证时，任何人都无法随意部署到 Databricks，符合安全预期。

#### 第二阶段：在 Databricks 端配置联合身份
1.  **在 GitHub 中创建环境**：在仓库设置中创建了 `dev` 和 `prod` 两个环境。这为后续的精细权限控制奠定了基础。
2.  **在 Azure Databricks 中创建服务主体**：
    *   切换到 **Databricks 账户控制台**（需要账户管理员权限）。
    *   在“用户管理”下，创建了一个名为 `databricks-managed-sp-dev` 的新服务主体。
3.  **创建联合策略（关键步骤）**：
    *   在账户控制台的“凭据和密钥”选项卡中，为刚创建的服务主体添加新策略。
    *   配置策略详情：
        *   **提供者**：选择 `GitHub Actions`。
        *   **GitHub 组织**：填写组织名称（从仓库 URL 获取，例如 `TableonAzure`）。
        *   **GitHub 仓库**：填写具体的仓库名（例如 `federated-identity-databricks`）。
        *   **范围限定**：选择使用“环境”进行进一步限制，并指定仅允许来自 `dev` 环境的请求。
    *   **策略效果**：此策略意味着，只有来自该特定组织、特定仓库，并且在 `dev` 环境上下文中运行的 GitHub Actions，才能获取令牌来扮演这个服务主体。
4.  **将服务主体添加到工作区**：
    *   返回到目标 Databricks 工作区。
    *   在“设置” -> “身份和访问” -> “服务主体”中，添加刚才创建的服务主体，使其在该工作区内可用。

#### 第三阶段：配置 GitHub Actions 以使用联合认证
需要修改 GitHub Actions 工作流文件（`.github/workflows/xxx.yml`），添加联合认证所需的配置。

```yaml
# 1. 添加必要的权限，允许工作流请求 ID 令牌
permissions:
  id-token: write
  contents: read

# 2. 指定运行环境和认证方式
environment: ‘dev‘ # 必须与联合策略中允许的环境名匹配

# 3. 在部署步骤中配置 Databricks CLI 使用联合认证
- name: Check Databricks Auth
  env:
    DATABRICKS_HOST: https://your-workspace.cloud.databricks.com
    DATABRICKS_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }} # 服务主体的应用程序(客户端) ID
  run: |
    databricks auth login --host $DATABRICKS_HOST \
    --client-id $DATABRICKS_CLIENT_ID \
    --azure-use-device-code
```
*   **`AZURE_CLIENT_ID`**：这是服务主体的“应用程序(客户端) ID”，**它不是密钥**，只是一个公开的标识符，可以安全地存储在 GitHub 仓库的 Secrets 中或直接引用（视频中为演示清晰直接写入工作流文件，实践中建议使用 Secrets）。
*   **认证流程**：当工作流以 `dev` 环境运行时，GitHub 的 OIDC（OpenID Connect）提供者会生成一个针对该特定工作流运行的令牌。Databricks CLI 使用此令牌和 `client_id` 向 Azure/Databricks 证明自己的身份，后者根据联合策略进行验证并交换为 Databricks 访问令牌。

#### 第四阶段：验证成功与安全测试
1.  **成功部署**：提交更新后的工作流，GitHub Actions 运行成功。`databricks auth whoami` 命令显示当前登录身份正是之前创建的 `databricks-managed-sp-dev` 服务主体。随后，资产捆绑包验证和部署步骤均成功完成。
2.  **结果验证**：在 Databricks 工作区的“作业”面板中，可以看到由该服务主体创建和运行的“Hello World”作业，执行成功。
3.  **安全策略有效性测试（关键验证）**：
    *   **测试1：更改环境**：将工作流中的 `environment: ‘dev‘` 改为 `environment: ‘prod‘` 并触发运行。**部署失败**，错误为“invalid grant”。这是因为联合策略只允许 `dev` 环境，`prod` 环境的请求被拒绝。
    *   **测试2：隐含的结论**：同样，如果有人将此工作流配置复制到另一个仓库，也会因为联合策略中绑定了特定仓库而失败。
    *   **结论**：这证明了联合策略的**精细访问控制是有效的**，权限被严格限定在预设的边界内。

### 4. 方案总结与适用范围
*   **本方案优势**：
    1.  **无密钥管理**：彻底摆脱了密码和密钥的创建、存储、轮换负担。
    2.  **增强安全性**：基于 OIDC 的令牌交换是短期的、自动管理的。权限被严格限定在特定的 GitHub 上下文（组织、仓库、环境）中，减少了凭证泄露和滥用的风险。
    3.  **简洁优雅**：配置清晰，无需维护长期有效的秘密信息。
*   **重要限制**：
    *   本视频演示创建的是 **Azure Databricks 管理的服务主体**。因此，通过此方式认证的 GitHub Actions **只能访问和操作 Databricks 资源**，无法用于部署 Azure 上的其他服务（如 Data Factory, Logic App 等）。
*   **扩展方案**：如果需要从 GitHub Actions 部署包括 Databricks 在内的多种 Azure 资源，视频提到可以参考另一个关于使用 **Entra（Azure AD）管理的服务主体配合联合凭证**的方案，该方案提供了更广泛的 Azure 资源访问权限。

## 核心要点总结
本视频的核心收获是，通过利用 **Azure Databricks 中的联合凭证（Federated Credentials）**，可以构建一个**既高度安全又易于维护的 CI/CD 管道**，用于从 GitHub 到 Databricks 的代码部署。这种方法的核心价值在于**用基于身份的、上下文绑定的临时令牌替代了传统的静态密钥**，从而消除了密钥泄露风险和繁琐的轮换操作。同时，通过将权限精细地限定在特定的 GitHub 组织、仓库和环境中，它极大地缩小了攻击面，确保了即使工作流配置被部分复制，也无法在未经授权的上下文中获得访问权限。对于专注于 Databricks 生态系统的团队，这是一个应优先考虑的安全部署实践。

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

So basically my goal is as follows. I want to grab code from the GitHub repository and deploy it to Databricks. And you might say, all right, but we've been doing this already using a service principal defined in Entra with some secret. And it worked, right? So why should we bother with a different approach?

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

So what I will do in this episode will be to create a new service principal inside Azure Databricks, now in Entra, in Databricks, and then I will use it during the deployment phase.

But now, how to make GitHub to be able to use this service principal? Well, that's where we'll create a federation policy object tied with a service principal.

that will define it it will allow GitHub to use this particular service

principal but only from my GitHub repository and only when using specific

GitHub environment like dev and prod and in this scenario tokens will be

exchanged between GitHub and Databricks and it will work without providing any

secrets or any passwords explicitly. So let me show you that in action. So I have

a GitHub repository, very simple one, this federated identity Databricks. It

It contains a very simple code that I cloned to my local machine.

And basically what I have here is a job definition,

Databricks job, this hello world one,

that contains only a single Python step

that runs this hello.py script

that in turn prints hello world statement.

And I configured a very simple Databricks asset bundle

use to deploy this job to a target environment.

A very simple scenario.

Now, in my GitHub repository,

I don't have any actions defined yet

to deploy my stuff to Databricks.

So let's start with doing this.

And to avoid creating everything from scratch,

let me use this very simple template.

So let me find it.

It's called simple workflow. In our case it is fine. We'll adjust it in a second. Let's give it a name and let's verify what it does. So basically it triggers on a push to main branch, pull request to main and it can be run manually. That's fine.

Then it runs on some Linux machine it gets the repo and it prints some hello world statements which we don't need actually So let me remove this oh or let me remove all of this and that's it and let's commit those changes and please know that I'm doing this directly on the main branch

you shouldn't do this in action in real life but for demo purposes this is fine

and because I made a commit it already resulted in my GitHub action being triggered and actually

it went fine it is green it didn't do anything yet with my Databricks code but at least we can see that

the trigger works so let's jump to VS Code let me pull those changes and let's update the workflow

definition. And now we know that to deploy Databricks asset bundles we have to first

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

and to deploy stuff to my Databricks environment.

So we expect this to fail.

And it did, it failed.

With error that default authentication,

I cannot configure default credentials, blah, blah, blah.

Basically, it means that we are not authenticated.

So let's fix this.

So following our design, we would like to make it secure.

So first of all, in my GitHub environment,

in my repository, I will create two environments,

dev and prod.

And that's basically what you do in your projects

because you might use different variables,

different secrets per environment,

or you might have a different protection rules

per environment.

And we'll use those environments

in our federation policy in a second.

So let me define those environments

like dev and prod.

I will not configure them in any way

because that's not necessary at this point.

We just need them to exist.

So we have dev and prod, and I will be using dev.

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

And let's create a new policy for this service principal We got to select provider that will use this policy And in our case it is GitHub Actions

because we want to use this service principal

from GitHub Actions.

Then we've got to provide a name of our GitHub organization.

In my case, it is Table on Azure.

You can see this in a URL to your repo.

So let me copy this.

Then we've got to provide a repository that will be allowed to use our service principal.

And in my case, it is federated identity Databricks.

Again, from URL.

So let me paste this.

Then we've got a choice that allows us to narrow down scenarios in which this policy and the service principal can be used.

I want to use environments. That's why we defined them previously.

and basically I want to define that this service principal will be allowed to be used only from

this repository on from this organization and only when deploying stuff to GitHub dev environment

that's what we are configuring here and the rest of the stuff is fine and let me create the policy

and that's it

the next thing to do is to basically create this service principal

in my Databricks workspace

so let's jump back to my workspace

and here in the settings

in identity and access, service principals

I don't have my service principal available in this workspace yet

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

Secondly, we defined this GitHub OpenID Connect

as authentication type.

It tells GitHub, hey, use this federated identity stuff.

I indicated where is my Databricks host located

and we've got to provide client ID of the service principal

that will be used for the login purposes and we can get this from here that's the application id

so let me copy this let me paste it here and that should be fine so let me

push this to the repo add that OpenID Connect stuff and hopefully everything will work

so let's jump back to our GitHub repository to actions and let's see if this time deployment

will pass and you can see that we already passed the step that previously failed right

and actually let's see who is logged in and oh please we can see that display name

is set to our service principal that we defined in Databricks So GitHub action uses this service principal credentials

to connect to Databricks.

So everything looks fine, right?

Then we were able to deploy,

to validate our Databricks asset bundle

and deploy it, right?

Everything went fine.

our our action faces green checkbox so let's verify this let's jump to other Databricks workspace

let's see jobs and pipelines and yes we've got this hello world pipeline that was deployed and

created by using this Databricks managed service principal and let me run it to verify that

everything works and inside we can see that that yes um the creator and the runner's identity is the

um principal that we created in Databricks and when it comes to tasks it has this very simple

single um python script task right so everything looks fine and about execution

it was executed and it printed hello world statement so everything works fine

one thing to check all right it works and please know that I didn't specify any secret

anywhere right no password this thing that might look like some secret stuff it is just an

identifier of my service principal so it is not a secret.

You can safely push this to the repository.

And let's say

what will happen if we try to use the same

service principal but on a production environment.

And let's see

if it will work. Changed it to prod.

This is to verify if our federation policy indeed restricts the usage of the service principal to the environment only.

And let's verify this in actions. Hopefully it will fail. And yes, it failed.

it failed because invalid grant is defined right basically and the stuff

that we configured in GitHub action it doesn't match the stuff we configured in

our Federation policy which simply means that it is not possible to overuse those

permissions right just to copy this workflow definition and use it in another repository

it will fail as well right because the federation policy limits the users of the service principal

to this repository only the same if we change the environment to prod right so basically

by using this federated credentials what we achieved so first of all no secrets no passwords

nothing is stored there is nothing we have to rotate periodically it simply works it is clean

it is elegant it is secure secondly those permissions are narrowed down to be used only

from my repository and only when deploying to dev environment so it is not possible just to grab

the code copy it and use it in another place just hoping that I would get access to

to my environment no it's empty words please be aware that in this scenario

we used a service principal defined in Azure Databricks so our GitHub workflow

will be able to access only Databricks.

It will not be able to go outside,

like to deploy anything to Azure Data Factory,

some logic app, or whatever you want, right?

This scenario, it focuses only on doing stuff

inside Databricks.

If you would like to use a more enterprise version

in which you still use federated stuff,

but you can deploy more things than Databricks,

then check out another video inside,

which I talk about how to do this

using Entra Managed Service Principal.

And basically that's it for today.

Thanks for watching and see you next time.

Take care.
