---
title: Istio Security
date: 2020-03-10 16:25:45
tags:
---

> 本文基于 Istio 1.4,翻译自[Istio官网](https://istio.io/docs/concepts/security/)，完全手工翻译

将单体应用程序分割成多个原子服务能带来很多好处，包括更高的灵活性，伸缩性和重用性。但是微服务也有特殊的安全需求：

* 为了防御中间人攻击，需要流量加密
* 为了提供灵活的服务访问控制，需要相互的TLS和细粒度的访问策略。
* 为了确定谁在什么时候做了什么，需要审计工具。

Istio Security 为解决这些问题提供了综合的安全方案，下图就如何使用 Istio Security 来加密应用给了一个概览，不管在哪里运行。Istio Security 缓解了应用内外的威胁，保护了数据，端点，通信和平台。

![Istio Security 概览](./overview.svg)

Istio 安全特性提供了强大的身份识别、强大的策略、透明的TLS加密以及身份验证、授权和审计(AAA)工具来保护您的服务和数据。Istio 安全的目标是：

* 默认安全性:不需要对应用程序代码和基础设施进行任何更改
* 深度防御:与现有安全系统集成，提供多层防御
* 零信任网络:在不信任的网络上构建安全解决方案

## 高级架构

Istio 安全涉及了多个组件：

* 用于密钥和证书管理的认证中心(CA)
* 分发到代理的 API 配置服务：
  
    * 认证策略
    * 授权策略
    * 安全命名信息

* 边车和边界代理作为策略实施点(PEPs)来保护客户机和服务器之间的通信。
* 一组用于管理遥测和审计的 Envoy 代理扩展

控制面板处理来自 API 服务器的配置，并在数据面板中配置  PEPs 。PEPs 最终使用 Envoy 实现的，下图显示了体系结构。

![安全架构](./arch-sec.svg)

## Istio 身份

身份在任何安全基础设置中都是一个基础概念，在工作负载到工作负载的通信开始时，双方必须交换有身份信息的凭据，以实现相互身份验证。在客户端，将根据安全命名信息检查服务器的身份，以确定它是否是工作负载的合法运行对象。在服务器端，服务器可以根据授权策略确定客户端可以访问哪些信息，审计谁在什么时候访问了什么，根据客户端使用的工作负载向客户端收费，并拒绝任何未能支付账单的客户端访问工作负载。

Istio 身份模型使用一流的 `service identity` 来确定请求源的身份。此模型为服务标识提供了极大的灵活性和粒度，以表示人类用户，单个工作负荷或一组工作负荷。在没有服务标识的平台上，Istio 可以使用可以对工作负载实例进行分组的其他标识，比如服务名称。

下面的列表显示了可以在不同平台上使用的服务标识示例：

* Kubernetes：Kubernetes 服务帐户
* GKE/GCE：GCP 服务帐户
* AWS：AWS IAM 用户/角色帐户
* 本地（非Kubernetes）：用户帐户，自定义服务帐户，服务名称，Istio 服务帐户或 GCP 服务帐户。 定制服务帐户是指现有服务帐户，就像客户的身份目录管理的身份一样。

## 公钥基础设施（Public Key Infrastructure ，PKI）

Istio PKI 安全地使用 X.509 证书为每个工作负载提供强身份。为了大规模地自动执行密钥和证书轮换，PKI 在每个 Envoy 代理旁边运行一个 Istio 代理，以进行证书和密钥预配。下图展示了身份标识的供应流程：

![ID 供应流程](./id-prov.svg)

Istio 通过使用以下流程的秘密发现服务(SDS)提供身份：

1. CA 提供了一个 gRPC 服务来接收证书签名请求(CSRs)。
2. Envoy 通过 Envoy 秘密发现服务(SDS) API 发送证书和密钥请求。
3. 在接收到SDS请求后，Istio 代理创建私钥和 CSR，然后将CSR 及其凭据发送给 Istio CA 进行签名。
4. CA 验证 CSR 中携带的凭据，并签署 CSR 以生成证书。
5. Istio 代理通过 Envoy SDS API 将从 Istio CA 接收到的证书和私钥发送给 Envoy。
6. 为了证书和密钥的旋转，上面的CSR过程会周期性地重复。

## 认证

Istio 提供了两种类型的认证：

* 对等身份验证：用于服务到服务的身份验证，以验证建立连接的客户端。 Istio 提供双向 TLS 作为用于传输授权的解决方案，可以在不更改服务代码的情况下启用它。这种情况下：
  
    * 为每个服务提供一个表示其角色的强标识，以支持跨集群和云的互操作性。
    * 确保服务到服务通信。
    * 提供密钥管理系统，以自动化密钥和证书的生成、分发和轮换。


* 请求身份验证：用于最终用户身份验证，以验证附加到请求的凭据。 Istio 使用JSON Web 令牌（JWT）验证启用请求级身份验证，并使用自定义身份验证提供程序或任何 OpenID Connect提供程序提供简化的开发人员体验，例如：
    
    * ORY Hydra
    * Keycloak
    * Auth0
    * Firebase Auth
    * Google Auth

在所有情况下，Istio 都会通过自定义的 Kubernetes API 将身份验证策略存储在 `Istio config store` 中。Istiod使它们与每个代理保持最新，并在适当的地方加上键。另外，Istio支持在许可模式下进行身份验证，以帮助您了解策略更改如何在实施之前影响您的安全状态。

### 双向 TLS 认证

Istio 通过 Envoy 代理实现的客户端和服务器端 PEP 建立服务到服务的通信隧道。当一个负载使用双向 TLS 认证来向另一个负载发送消息时，请求会被如下处理：

1. Istio 将出站流量从客户端重新路由到客户端的本地边车 Envoy 。
2. 客户端 Envoy 与服务器端 Envoy 开始双向 TLS 握手。 在握手期间，客户端 Envoy 还会进行安全命名检查，以验证服务器证书中提供的服务帐户是否有权运行目标服务。
3. 客户端 Envoy 和服务器端 Envoy 建立了双向 TLS 连接，Istio 将流量从客户端 Envoy 转发到服务器端 Envoy。
4. 经过授权后，服务器端特使通过本地TCP连接将流量转发给服务器服务。

#### 许可模式

Istio 双向 TLS有一个许可模式，它允许一个服务同时接受明文流量和双向 TLS 流量。该特性极大地改进了双向 TLS 的调试体验。

与非 Istio 服务器通信的许多非 Istio 客户端给想要在启用双向 TLS 的情况下将该服务器迁移到Istio的运营商带来了问题。一般来说，运营商不可能同时在所有的客户端安装 Istio 边车，甚至有些客户端运营商都没有权限。即使安装了 Istio 边车，运营商也无法在不终止已有通信的情况下启用双向 TLS。

在许可模式开启的情况下，服务器同时接受明文流量和双向 TLS 流量。该模式为入职流程提供了更大的灵活性。服务器安装的Istio 边车可立即进行双向 TLS 流量，而不会破坏现有的纯文本流量。因此，运营商可以逐步安装和配置客户端的 Istio 边车，以发送相互的 TLS 流量。一旦客户端配置完毕，运营商可以将服务器设置为双向 TLS 模式。


### 安全命名

服务的身份在证书中编码，但是服务的名称是通过发现服务或者 DNS 获取。安全命名信息将服务器标识映射到服务名称。一个身份 `A` 到服务 `B` 的映射表示“ A 被允许运行 B 服务”。控制面板监听 `apiserver`，生成安全命名映射，并将它们安全地分发给 PEP。下面的示例解释了安全命名在身份验证中的重要性。

假设运行 `datastore ` 服务的合法服务器仅使用 `infra-team` 身份。 恶意用户拥有 `test-team` 身份的证书和密钥。恶意用户打算模拟服务以偷窥从客户端发送的数据。恶意用户使用证书和密钥为 `infra-team` 身份部署一个伪造的服务器。假设恶意用户成功劫持（通过DNS欺骗，BGP /路由劫持，ARP欺骗等）发送到 `datastore` 的流量并将其重定向到伪造的服务器。

当客户端调用 `datastore` 服务时，它从服务端证书中提取 `test-team` 身份标识，然后通过安全命名信息检查 `test-team` 是否有权限运行 `datastore` 服务。客户端发现 `test-team` 无权运行 `datastore` ，认证就会失败。

安全命名能够防止 HTTPS 流量的一般网络劫持。它还可以保护TCP 流量免受一般网络劫持。然而，安全命名并不能防止 DNS 欺骗，因为在这种情况下，攻击者会劫持 DNS 并修改目标的 IP 地址。这是因为 TCP 流量不包含主机名信息，我们只能依赖 IP 地址进行路由。事实上，这种 DNS 劫持甚至可以在客户端特使接收到流量之前发生。

### 认证架构

可以使用对等和请求身份验证策略在 Istio 网格中为接收请求的工作负载指定身份验证需求。网格使用 `.yaml` 文件指定策略。部署之后，策略将保存在 Istio 配置存储中。Istio 控制器监视配置存储。

在任何策略更改后，新策略都会转换为适当的配置，告诉 PEP 如何执行所需的身份验证机制。控制面板可以获取公钥并将其附加到配置中以进行 JWT 验证。另外，Istiod 提供了到 Istio 系统管理的密钥和证书的路径，并将它们安装到应用程序 pod 中以实现双向 TLS。

Istio 异步地将配置发送到目标端点。一旦代理接收到配置，新的身份验证要求立即在该 pod 上生效。

发送请求的客户机服务负责遵循必要的身份验证机制。对于对等身份验证，应用程序负责获取 JWT 凭据并将其附加到请求。对于双向 TLS, Istio 自动将两个 PEP 之间的所有流量升级为双向 TLS 。如果身份验证策略禁用了双向 TLS 模式，Istio 将继续在 PEP 之间使用纯文本 。要覆盖此行为，请使用目标规则明确禁用双向 TLS 模式。

![认证架构](./authn.svg)

Istio 将身份验证和身份验证中的其他声明（如果适用）输出到下一层：授权。

### 认证策略

认证策略应用于服务接收到的请求。要在双向 TLS 中指定客户端身份验证规则，需要在 `DestinationRule` 中指定 `TLSSettings` 。

和其他的 Istio 配置一样，你可以在 `.yaml` 文件中指定认证策略。使用 `kubectl` 部署策略。以下示例指定了标签为 `app:reviews` 的工作负载的传输策略必须使用双向 TLS 。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-peer-policy"
  namespace: "foo"
spec:
  selector:
    matchLabels:
      app: reviews
  mtls:
    mode: STRICT
```

#### 策略存储 

Istio 将网格范围的策略存储在根命名空间中。这些策略只有一个空的选择器，所以适用于所有的工作负载。有命名空间的策略会存储在对应的命名空间内，这类策略只影响同一个命名空间的工作负载。如果配置了 `selector` 属性，认证策略就会只影响符合配置条件的工作负载。

对等和请求身份验证策略按种类分别存储，分别为 `PeerAuthentication` 和 `RequestAuthentication` 。

#### 选择器属性

对等和请求身份验证都使用 `selector` 属性来指定策略应用目标负载的标签，以下示例展示了一个应用于 `app:product-page` 标签的的策略选择器：

```yaml
selector:
     matchLabels:
       app:product-page
```
如果没有为选择器字段提供值，则 Istio 会将策略与策略存储范围内的所有工作负载进行匹配。因此，选择器字段可帮助您指定策略的范围：

* 网格范围的策略：为根名称空间指定的策略，不带或带有空的选择器字段。
* 命名空间范围的策略：为非根命名空间指定的策略，不带或带有空的选择器字段。
* 特定于工作负载的策略：在常规名称空间中定义的策略，带有非空选择器字段。

对等方和请求身份验证策略对 `selector` 字段遵循相同的层次结构原则，但是 Istio 以稍微不同的方式组合并应用它们。

每个命名空间只能有一个网格范围的对等身份验证策略，并且只能有一个命名空间范围的对等身份验证策略。如果同一网格或名称空间配置多个网格范围或名称空间范围的对等身份验证策略时，Istio 会忽略较新的策略。当多个特定于工作负载的对等身份验证策略匹配时，Istio 将选择最旧的策略。

Istio按照以下顺序为每个工作负载应用最窄的匹配策略：

1. 指定工作负载
2. 命名空间范围
3. 网格范围

Istio 可以将所有匹配的请求身份验证策略组合起来，就像它们来自单个请求身份验证策略一样。因此，可以在网格或名称空间中具有多个网格范围或名称空间范围的策略。但是，避免使用多个网格范围或命名空间范围的请求身份验证策略仍然是一个好习惯。

#### 对等认证

对等身份验证策略指定 Istio 对特定工作负载实施的双向 TLS 模式。支持以下模式：

* 许可：工作负载接受双向 TLS 和纯文本流量。此模式在迁移过程中最有用，此时没有边车的工作负载无法使用双向 TLS 。使用边车注入迁移工作负载后，应将模式切换为严格模式。
* 严格：工作负载仅接受双向 TLS 通信。
* 禁用：双向 TLS 被禁用。从安全角度来看，除非您提供自己的安全解决方案，否则不应使用此模式。

当模式未设置时，父范围的模式将被继承。 默认情况下，未设置模式的网格范围对等身份验证策略使用许可 `PERMISSIVE` 模式。

以下对等验证策略要求所有 `foo` 命名空间的工作负载使用双向 TLS ：

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-policy"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
```
使用特定于工作负载的对等身份验证策略，可以为不同的端口指定不同的双向 TLS 模式。对于端口范围的双向 TLS 配置，只能应用于工作负载占用的端口。以下示例为 `app:example-app` 工作负载在 80 端口禁用双向 TLS ,然后在其他端口为对等认证启用双向 TLS:

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "example-workload-policy"
  namespace: "foo"
spec:
  selector:
     matchLabels:
       app: example-app
  portLevelMtls:
    80:
      mode: DISABLE
```

上面的对等身份验证策略仅起作用是因为下面的服务配置将来自 `example-app` 工作负载的请求绑定到 `example-service` 的 80 端口：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: foo
spec:
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: example-app
```

#### 请求认证

请求身份验证策略指定验证 JSON Web 令牌（JWT）所需的值。这些值包括：

* 令牌在请求中的位置
* 发行者或请求
* 公用 JSON Web 密钥集（JWKS）

Istio 检查提供的令牌，如果违反了请求身份验证策略中的规则，就会拒绝无效令牌的请求。当请求不携带令牌时，默认接受它们。要拒绝没有令牌的请求，请提供授权规则，该规则指定对特定操作（例如，路径或操作）的限制。

如果每个 JWT 使用唯一的位置，请求身份验证策略可以指定多个 JWT 。当多个策略匹配一个工作负载时，Istio 将组合所有规则，就像将它们指定为单个策略一样。这种行为对于编程工作负载从不同的提供者接收JWT非常有用。但是，不支持具有多个有效 JWT 的请求，因为此类请求的输出主体是未定义的。

#### 原则

使用对等身份验证策略和双向 TLS 时，Istio 将身份标识从对等身份验证提取到 `source.principal` 中。同样，当使用请求身份验证策略时，Istio 会将 JWT 中的身份标识分配到 `request.auth.principal` 。使用这些原则设置授权策略并作为遥测输出。

### 更新认证政策

可以随时更新认证策略， Istio 几乎是实时地推送新策略到工作负载。但是 Istio 无法保证所有工作负载都同时收到新政策。 以下建议有助于避免在更新身份验证策略时造成干扰：

* 将模式从禁止（DISABLE）更改为严格（STRICT）时，请使用许可（PERMISSIVE）模式使用缓和对等身份验证策略，反之亦然。 当所有工作负载成功切换到所需模式时，您可以将策略应用于最终模式。 您可以使用 Istio 遥测技术来验证工作负载已成功切换。
* 将对等身份验证策略从一个 JWT 迁移到另一个 JWT 时，将新 JWT 的规则添加到该策略中，而不删除旧规则。 然后，工作负载接受两种类型的JWT，并且当所有流量都切换到新的JWT时，您可以删除旧规则。 但是每个 JWT 必须使用不同的位置。

## 授权

Istio 的授权功能为网格中的工作负载提供了网格，命名空间和工作负载范围的访问控制。这种控制级别具有以下优点：

* 工作负载到工作负载和最终用户到工作负载授权。
* 一个简单的 API ：它包含单个 `AuthorizationPolicy` CRD，它易于使用和维护。
* 灵活的语义：运营商可以在 Istio 属性上定义自定义条件，并使用 DENY 和 ALLOW 操作。
* 高性能：Istio 授权在 Envoy 上本地执行。
* 高兼容性:支持 gRPC、HTTP、HTTPS 和 HTTP2 本机，以及任何普通的 TCP 协议。

### 授权架构

每个 Envoy 代理都运行一个授权引擎，该引擎在运行时对请求进行授权。当请求到达代理时，授权引擎会根据当前的授权策略评估请求上下文，并返回授权结果 ALLOW 或 DENY。 运营商使用 `.yaml`文件指定 Istio 授权策略。

![授权体系](./authz.svg)

### 隐式支持

您无需明确启用 Istio 的授权功能。只需将授权策略应用于工作负载即可实施访问控制。对于没有应用授权策略的工作负载， Istio 不会默认执行允许所有请求的访问控制。

授权策略支持 ALLOW 和 DENY 动作。拒绝策略优先于允许策略。如果将任何允许策略应用于工作负载，则默认情况下将拒绝对该工作负载的访问，除非策略中的规则明确允许。当您将多个授权策略应用于相同的工作负载时，Istio 会附加地应用它们。

### 授权策略

要配置授权策略，请创建一个 `AuthorizationPolicy` 自定义资源。授权策略包括选择器，动作和规则列表：

* `selector` 字段指定了规则应用的目标
* `action` 字段指定了是允许还是拒绝请求
* `rules` 指定何时触发动作
    * `rules` 中的 `from` 字段指定请求的源
    * `rules` 中的 `to` 字段指定请求的操作
    * `when` 字段指定应用规则所需的条件

下例显示了一个授权策略，该策略允许两个源（ `cluster.local/ns/default/sa/sleep` 服务帐户和 `dev` 命名空间）使用合法的 JWT 访问 `foo` 命名空间中标签为 `app: httpbin` 和 `version: v1` 的工作负载。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/default/sa/sleep"]
   - source:
       namespaces: ["dev"]
   to:
   - operation:
       methods: ["GET"]
   when:
   - key: request.auth.claims[iss]
     values: ["https://accounts.google.com"]
```
以下示例显示了一个授权策略，如果源不是 `foo` 名称空间，该策略将拒绝请求：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin-deny
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: DENY
 rules:
 - from:
   - source:
       notNamespaces: ["foo"]
```

拒绝策略优先于允许策略。如果匹配允许策略的请求也与拒绝策略匹配，则可以拒绝这些请求。Istio 首先评估拒绝政策，以确保允许政策不会绕过拒绝政策。

#### 策略目标

可以通过 `metadata/namespace` 字段和可选的 `selector` 字段来指定策略的范围或目标。策略应用于 `metadata/namespace` 字段中配置的命名空间。 如果将其值设置为根命名空间，则该策略将应用于网格中的所有命名空间。根命名空间的值是可配置的，默认值是 `istio-system` 。如果设置为任何其他命名空间，则该策略仅适用于指定的命名空间。

可以使用选择器字段来进一步限制策略以应用于特定工作负载。选择器使用标签选择目标工作负载。选择器包含一系列的键值对，其中的 `key` 是标签的名字。如果未设置，则授权策略将应用于与授权策略相同命名空间中的所有工作负载。

例如，`allow-read` 策略允许通过 `GET` 和 `HEAD` 访问 `default` 空间中的有 `app：products` 标签的工作负载：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-read
  namespace: default
spec:
  selector:
    matchLabels:
      app: products
  action: ALLOW
  rules:
  - to:
    - operation:
         methods: ["GET", "HEAD"]
```
#### 值匹配

授权策略中的大多数字段都支持以下所有匹配模式：

* 完全匹配：完全匹配的字符串。
* 前缀匹配：以“`*`”结尾的字符串。例如，“test.abc.*”与“test.abc.com”，“ test.abc.com.cn”，“test.abc.org”等匹配。
* 后缀匹配：以“`*`”开头的字符串。例如，“*.abc.com”匹配“eng.abc.com”，“test.eng.abc.com”等。
* 状态匹配：`*` 用于指定任何内容，但不能为空。要指定必须存在一个字段，请使用`fieldname: ["*"]`格式。这与不指定字段（表示匹配任何内容，包括空）不同。

有一些例外。例如，以下字段仅支持完全匹配：

* `when` 部分下的 `key` 字段
* `source` 部分下的 `ipBlocks` 字段
* `to` 部分下的 `ports` 字段

以下示例策略允许访问带有`/test/*`前缀或`*/info`后缀的路径。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: tester
  namespace: default
spec:
  selector:
    matchLabels:
      app: products
  action: ALLOW
  rules:
  - to:
    - operation:
        paths: ["/test/*", "*/info"]
```

#### 排除匹配

为了匹配诸如 `when` 字段中的 `notValues` , `source` 字段中的 `notIpBlocks` ，`to` 字段中的 `notPorts` 之类的否定条件，Istio支持排除匹配。以下示例要求一个有效的请求主体，如果请求路径不是 `/healthz` ，则它是从 JWT 身份验证派生的。因此，该策略从 JWT 身份验证中排除对 `/healthz` 路径的请求：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: disable-jwt-for-healthz
  namespace: default
spec:
  selector:
    matchLabels:
      app: products
  action: ALLOW
  rules:
  - to:
    - operation:
        notPaths: ["/healthz"]
    from:
    - source:
        requestPrincipals: ["*"]
```

下面的示例拒绝不带请求主体的请求到 `/admin` 路径的请求：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: enable-jwt-for-admin
  namespace: default
spec:
  selector:
    matchLabels:
      app: products
  action: DENY
  rules:
  - to:
    - operation:
        paths: ["/admin"]
    from:
    - source:
        notRequestPrincipals: ["*"]
```

#### 允许所有和默认拒绝所有授权策略

以下示例显示了一个简单的允许所有授权策略，该策略允许完全访问 `default` 命名空间中的所有工作负载。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-all
  namespace: default
spec:
  action: ALLOW
  rules:
  - {}
```
以下示例显示了一项策略，该策略不允许对 `admin` 命名空间中的任何工作负载进行任何访问。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: admin
spec:
  {}
```

#### 定制条件

您还可以使用 `when` 部分指定其他条件。例如，下面的 `AuthorizationPolicy` 定义包括了一个 `request.headers [version]` 只能为 “v1” 或 “v2” 的条件。 在这种情况下，键是 `request.headers[version]`，这是 Istio 属性 `request.headers` 中的一项，该属性是一个 map 。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/default/sa/sleep"]
   to:
   - operation:
       methods: ["GET"]
   when:
   - key: request.headers[version]
     values: ["v1", "v2"]
```

#### 认证和未认证的身份

如果你想让一个工作负载公开可访问，你需要将 `source` 属性配为空，这允许来自所有（经过身份验证和未经身份验证的）用户和工作负载的请求，例如：

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - to:
   - operation:
       methods: ["GET", "POST"]
```
为了只允许经过身份验证的用户，可以将 `principals ` 设置为“`*`”:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["*"]
   to:
   - operation:
       methods: ["GET", "POST"]
```
### 在纯 TCP 协议上使用 Istio 授权

Istio 授权使用任何简单的 TCP 协议（例如MongoDB）支持工作负载。在这种情况下，可以按照与 HTTP 工作负载相同的方式配置授权策略。不同之处在于某些字段和条件仅适用于 HTTP 工作负载。这些字段包括：

* 授权策略对象的 `source` 部分中的 `request_principals` 字段
* 授权策略对象的操作部分中的 `hosts`，`methods` `paths` 字段
  
如果将任何仅 HTTP 字段用于 TCP 工作负载，则 Istio 将忽略授权策略中的 HTTP 专用字段。

假设在 27017 端口上运行 `MongoDB` 服务，以下示例将授权策略配置为仅允许 Istio 网格中的 `bookinfo-ratings-v2` 服务访问 `MongoDB` 工作负载。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: AuthorizationPolicy
metadata:
  name: mongodb-policy
  namespace: default
spec:
 selector:
   matchLabels:
     app: mongodb
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/default/sa/bookinfo-ratings-v2"]
   to:
   - operation:
       ports: ["27017"]
```
### 依赖于双向 TLS

Istio使用双向TLS将某些信息从客户端安全地传递到服务器。 在使用授权策略中的以下任何字段之前，必须启用双向 TLS：

*  `source` 部分下的 `principals` 字段
*  `source` 部分下的 `namespaces` 字段
*  `source.principal` 自定义条件
*  `source.namespace` 自定义条件
*  `connection.sni` 自定义条件

如果在授权条件中不使用以上字段，双向 TLS 就不是必须的。