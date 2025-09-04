# 安全性

在微服务中的安全性可能感觉像是一片雷区，在某些方面确实如此。本章主要旨在探讨一些你可以做的事情来提高你的 Go 代码的安全性；然而，我认为讨论一些更广泛的问题也很重要。对于像防火墙配置这样的主题，最好留给专业书籍来深入讨论；然而，我们将介绍一些你面临的概念和问题，以便你为更深入的阅读做好准备。

# 加密和签名

当我们探讨保护数据的方法，无论是静态还是传输中的数据，我们讨论的许多方法都将涉及密码学保护数据。

“密码学是使用数学来加密和解密数据的科学。密码学使你能够存储敏感信息或通过不安全的网络（如互联网）传输信息，这样除了预期的接收者外，任何人都无法读取。”

- 《密码学简介》，网络合作伙伴公司

作为本章我们将讨论的内容的基础，我们首先必须了解密码学是如何工作的，不是我们需要一个数学学位，而是要了解涉及的各个部分。密码学的安全性取决于所涉及的密钥的安全性，我们需要了解哪些密钥可以自由分发，哪些密钥需要我们用生命来保护。

# 对称密钥加密

对称密钥加密也称为密钥加密或传统密码学：一个密钥用于数据的加密和解密。为了远程端能够解密这些信息，它必须首先拥有这个密钥，并且这个密钥必须安全地保存，因为一个服务器的密钥泄露将导致所有共享这个密钥的服务器被泄露。它也可能使密钥管理更加复杂，因为当你需要更改密钥，而你应该经常更改它时，你需要将这个更改在整个环境中推广。

# 公钥密码学

公钥密码学是由 Whitfield Diffie 和 Martin Hellman 于 1975 年引入的，以避免双方都需要知道秘密的需要。实际上，他们并不是第一个发明这种技术的；它是由英国情报机构多年前开发的，但被当作军事机密保密。

公钥密码学使用一对密钥进行加密；你也会听到它被称为非对称加密。公钥用于加密信息，而私钥只能用于解密。因为无法从公钥中确定私钥，所以通常公钥会被公之于众。

# 数字签名

公钥加密也赋予我们使用数字签名的功能。数字签名通过使用私钥加密消息然后传输已签名的消息来实现。如果消息可以用公钥解密，那么消息一定是从私钥持有者那里发出的。由于加密消息的计算时间和有效载荷大小的增加，一个标准的方法是创建消息的单向哈希，然后使用私钥加密这个哈希。接收者将使用公钥解密哈希，并从消息中生成相同的哈希；然后，可以认为消息来自一个可信的来源。

# X.509 数字证书

公钥的一个问题是您必须小心，您认为属于接收者的密钥确实属于接收者。如果密钥在公共网络上传输，总有可能发生中间人攻击。攻击者可以冒充一个假公钥，作为您认为的可信接收者；然而，他们可以用自己的密钥替换它。这意味着您认为已经安全传输的消息实际上可能被恶意第三方解密和读取。

为了避免这些问题，存在数字证书，简化了确定公钥是否属于报告的所有者的任务。

一个数字证书包含三样东西：

+   一个公钥

+   证书信息，如所有者的名称或 ID

+   一个或多个数字签名

使证书可信的是数字签名。证书由一个受信任的第三方或证书授权机构（CA）签署，该机构证明您的身份以及您的公钥属于您。任何人都可以创建 CA 根证书并签署他们的证书，对于非公开访问系统，如微服务之间的通信，这是一种相当常见的做法。然而，对于公开证书，您需要支付 CA 来签署您的证书。定价的目的是让 CA 确保您确实是您所说的那个人；目前，最受欢迎的 CA 是 Comodo、Symantec（在收购之前是 Verisign）和 GoDaddy。您在浏览器中看到的原因不仅仅是您正在使用安全通信，而是您的浏览器已经验证了证书的签名，这是与它捆绑在一起的 100 多个受信任的第三方之一。

# TLS/SSL

SSL，即两个系统之间安全传输数据的通用术语，是指由 Mozilla 在 1995 年首次开发的一个已弃用的标准。自那时起，它已被 2008 年 8 月发布的 TLS 1.2 所取代；尽管 SSL 3.0 在技术上仍然可行，但在 2015 年 6 月由于**POODLE**（**在降级旧加密中摆动预言机**）攻击的漏洞而被弃用。2014 年由一支谷歌安全研究人员团队发现的 POODLE 攻击是通过攻击者向服务器发送多个请求来工作的；然后分析这些数据并使用它们，这使得他们能够解密传输中的数据。平均而言，只需要进行 256 次 SSL 3.0 调用，就可以解密 1 字节的信息。

这意味着这个漏洞在公开披露之前已经存在了 18 年；你可能会问，为什么在更强的 TLS 1.0 发布 15 年后，人们还在使用 SSL 3.0？这起因于一些浏览器和服务器不支持 TLS 1.0 的问题，因此存在一个回退机制，允许降级到较低级别的加密。尽管在发现时几乎没有人还在使用 SSL 3.0，但回退机制仍然存在于协议中，因此可以被黑客利用。解决这个问题的方法相当简单：在服务器的配置中禁用低于 TLS 1.0 的所有内容。我们有一些 TLS 和 SSL 的历史，但它是如何保护你的数据安全的呢？

TLS 使用对称加密，客户端和服务器都拥有一个用于加密和解密的密钥。如果你还记得上一节，我们介绍了对称加密和密钥分发的问题。TLS 通过在握手的第一部分使用非对称加密来解决这个问题。客户端从服务器检索包含公钥的证书并生成一个随机数；它使用公钥来加密这个随机数并将其发送回服务器。现在双方都有了随机数，他们使用这个随机数来生成对称密钥，这些密钥用于在传输过程中加密和解密数据。

# 外部安全

这是保护您系统安全的第一道防线，通常由第 2 层或第 3 层防火墙、DDoS 保护、Web 应用防火墙以及其他软件和硬件组成。在攻击者能够破坏您的应用程序之前，他们必须首先通过这些硬件和软件层，这些层不是应用程序代码的一部分，而是一个共享的基础设施层，应用程序中的许多组件可能都会共享。在本节中，我们将探讨一些外部安全措施以及可能针对您发起的攻击。通常，保护服务边界是由运维人员完成的任务，然而作为开发者，我们需要了解这些流程和风险，因为这极大地增强了我们加固应用程序代码的能力。在本节中，我们将探讨外部安全的一些常见方法，以及黑客可能利用系统的一些方式。

# 第 2 层或第 3 层防火墙

第 2 层更常用于路由，因为它只处理 MAC 地址，而不处理 IP 地址，而第 3 层则是 IP 地址感知的。传统上，第 2 层是唯一真正的方式，因为它不会增加延迟，其速度与电缆大致相同。随着处理能力和内存的增加，第 3 层现在可以以电缆速度运行，通常，当我们查看边缘防火墙时，这些防火墙通常是进入您系统的第一个入口点，现在它们通常是第 3 层。这给我们带来了什么？首先，它阻止了边缘的不必要流量：我们限制了对外部世界的可访问端口，并阻止了流量到达不允许的目标，这些流量在防火墙处被阻止，没有机会对源头执行攻击。除此之外，它还允许我们限制对某些端口的访问。例如，如果您正在运行服务器，您很可能需要某种形式的远程访问，如 SSH。2015 年出现的 Heartbleed 漏洞利用了 OpenSSH 中的漏洞，直接暴露在互联网上的 SSH 服务器容易受到这种攻击。有效地使用防火墙意味着私有端口，如 SSH，将被锁定到 IP 地址或 IP 地址范围，这可能是您的 VPN、办公 IP 或公共 IP。这大大减少了攻击向量，因此即使您正在运行一个容易受到 Heartbleed 影响的 OpenSSH 版本，攻击者要利用这一点，他们也需要在您的受保护区域内。

Heartbleed 漏洞利用了执行缓冲区溢出攻击的能力。例如，你要求服务器返回一个 4 个字母的单词，但指定长度为 500；你得到的是 4 个字母的单词，剩下的 496 个字符是跟随初始分配内存地址的内存块。在实践中，这允许黑客随机访问服务器中的内存块；这可能包含诸如更改密码请求这样的项目，这些项目给了他们访问服务器的凭证。如果你运行的是一个全球可用的 SSH 服务器，那么你可能发现你有一个问题：

![图片](img/6c128381-747f-4ba7-8e96-be61fe250825.png)

# 网络应用防火墙

网络应用防火墙（**WAF**）在系统中配置为第二或第三道防线。为了理解什么是 WAF，让我们看看来自开放网络应用安全项目（OWASP）的定义：

“网络应用防火墙（WAF）是针对 HTTP 应用的防火墙。它将一系列规则应用于 HTTP 会话。这些规则涵盖了常见的攻击，如跨站脚本（XSS）和 SQL 注入。

当代理保护客户端时，WAF 保护服务器。WAF 被部署以保护特定的网络应用或一组网络应用。WAF 可以被视为反向代理。

WAF 可能以设备、服务器插件或过滤器的形式出现，并且可以根据应用程序进行定制。进行这种定制的努力可能非常显著，并且需要随着应用程序的修改而维护。”

OWASP 是一个非常有用的资源，实际上已经为 ModSecurity 提供了一套核心规则集，ModSecurity 可以防止诸如 SQL 注入、XSS、Shellshock 等攻击。作为最低要求，设置一个如 ModSecurity 和 OWASP CRS 的 WAF 应该是你的基本需求。在 Docker 容器内托管这应该相对简单，这可以形成在第二层防火墙之后的第二道防线。

另外还有一个选择：一些 CDN 公司，如 Cloudflare，提供托管 WAF 服务。这是在网络的边缘提供保护，得益于像 Cloudflare 这样的企业专业知识，你无需担心配置问题。实际上，Cloudflare 支持 OWASP CRS（[`www.modsecurity.org/crs/`](https://www.modsecurity.org/crs/)）。

# API 网关

除了 WAF（Web 应用防火墙）之外，API 网关也是一个非常有用的工具；它可以同时起到将您的公共 API 路由到后端服务以及一些附加功能的作用，例如在边缘进行令牌验证和输入验证与转换。当我们讨论到困惑代理问题，即防火墙后面的攻击者可以执行他们无权执行的命令时，我们探讨了加密 Web 令牌的可能性；但问题是，用于解密这些令牌的私钥需要分布到多个后端服务中。这使得密钥管理比应有的要复杂得多。API 网关可以通过成为唯一可以解密消息的层来简化这种情况；其他服务使用公钥来验证签名。API 网关通常实现许多其他一线功能，例如但不限于以下内容：

+   请求验证

+   授权

+   速率限制

+   记录

+   缓存

+   请求和响应转换

WAF 和 API 网关之间存在一定的交叉；然而，这两个应该被视为您基础设施中两个截然不同的部分。关于 API 网关的提供商，这似乎是一个正在发展的领域；如果您已经购买了 AWS PaS 环境，可以使用 AWS 的高级 API 网关。对于独立部署，Kong（[`getkong.org/`](https://getkong.org/)）、Tyk（[`tyk.io/`](https://tyk.io/)）、Apigee（[`apigee.com/api-management/#/homepage`](https://apigee.com/api-management/#/homepage)）、Mashery（[`www.mashery.com/`](https://www.mashery.com/)）以及 Mulesoft 的 Anypoint 平台（[`www.mulesoft.com/`](https://www.mulesoft.com/)）是该领域的领导者。当然，您也可以使用 Nginx 或 HAProxy 构建自己的 API 网关；然而，我建议您在着手构建自己的之前先检查一下这些特定的平台。

# DDoS 保护

2016 年 10 月 21 日，一次大规模的互联网中断是由攻击者针对 DYN 的 DNS 服务器使用 Mirai 僵尸网络造成的。Mirai 漏洞利用了由名为 XionMai Technologies 的中国公司制造的 IP 摄像头和 DVR 中的漏洞。攻击者没有攻击目标，而是决定摧毁互联网基础设施的主要部分，使美国东海岸和西海岸的大部分地区瘫痪。Mirai 漏洞仅利用了 60 个用户名和密码来尝试更新易受攻击设备的固件。一旦恶意软件被安装，设备就被僵尸网络控制。剩下的只是告诉机器人对 DYN 的名称服务器发起 DNS 攻击。

Mirai 的代码已经在网上公布；你只需稍加努力，用 Google 就可以找到。我希望你看到这段代码时感到惊讶的是它的简单性。现在，我不想从设计这种攻击的复杂性中减去任何东西；我只是在谈论实施。相当大的一部分代码是用 Go 编写的，因此非常易于阅读。其中有一些对通道的出色使用。如果你查看代码，尝试找出可以用信号量改进的区域。

Akamai 发布的一份报告称，今年所有攻击中有 98.34% 是针对基础设施的，只有 1.66% 是针对应用层的。在这 98.34% 中，许多可以通过一点网络卫生来避免。让我们来看看主要的威胁以及它们是如何工作的。

# DDoS 攻击的类型

以下是一些 DDoS 攻击的类型：

+   UDP 分片

+   DNS

+   NTP

+   Chargen

+   UDP

+   SYN

+   SSDP

+   ACK

# UDP 分片攻击

UDP 分片攻击是攻击者利用网络中数据报分片的方式。每个网络都有一个称为最大传输单元（MTU）的限制。如果一个发送到网络的数据报大于 MTU，它将被分片以成功传输。

UDP 分片攻击通过创建包含伪造数据包的数据报来实现；当服务器尝试重新组装这些数据包时，它无法做到，资源很快就会被耗尽。

# UDP 洪水

UDP 洪水攻击通过向一个 IP 地址发送大量带有伪造源地址的 UDP 数据包来实现。服务器将对这些请求做出响应，向伪造的地址发送回复。由于攻击的高强度，路由器将超过每秒 UDP 数据报的限制，并在一段时间内停止向同一安全区域内的所有地址发送数据。

这通常还会利用一种称为反射攻击的技术。当源 IP 地址被伪造时，返回的数据包不会发送回真实源地址，而是发送到伪造的 IP 地址。使用这种技术的理由是，它允许发送者通过仅消耗出站数据包的资源来放大攻击。

# DNS

DNS 攻击利用 UDP 洪水来攻击 DNS 服务器；会发出许多请求来查询 DNS 服务器。这些请求被设计成从一个小请求返回一个非常大的回复，以最大化攻击效率，因为通常发送者不会收到回复。

我们之前讨论过的攻击，它针对的是 Dyn 的基础设施，在 2016 年 10 月导致美国东海岸和西海岸的许多网站瘫痪，这种攻击就是以这种方式进行的。与大多数 DNS 攻击不同，Miraia 网络没有使用反射，它允许响应返回给发送者，这是由于大量被入侵的设备才成为可能。

# NTP

NTP（网络时间协议）是另一种利用 NTP 服务器内置功能的放大攻击，该功能返回与它交互的最后 600 台机器。这种攻击利用了支持 MONLIST 命令且未打补丁的开放 NTP 服务器。该项目[`openntpproject.org/`](http://openntpproject.org/)旨在识别未打补丁的服务器，以鼓励移除这种漏洞。不幸的是，NSFOCUS 在 2014 年进行的研究发现，全球有超过 17,000 台服务器容易受到这种漏洞的攻击。假设所有这些服务器都可以被利用，并使用 2014 年 CloudFlare 遭受的 NTP 攻击的负载大小，我们有能力进行 1.4 Tbps 的 DDoS 攻击。这种流量将是今天已知最大攻击的两倍。NTP 提供了一个强大的应用程序攻击平台，仅因服务器打补丁不当而存在。

# CHARGEN

**CHARGEN**（字符生成协议）攻击是另一种反射放大攻击。攻击利用了开放的 CHARGEN 服务器，这些服务器在端口`19`上运行，每次从连接的主机接收数据报时，都会返回 0 到 512 个字符长度的随机字符数。CHARGEN 被设计为用于调试 TCP 网络代码和带宽测量的字节流源。CHARGEN 攻击通过滥用已启用在网络连接打印机上的 CHARGEN 服务器来工作。

# SYN 洪水

SYN 洪水是一种经典的 DDoS 攻击，向一台机器发送大量数据包，试图阻止连接被关闭。最终，服务器端的连接会超时；然而，目的是反复攻击服务器，消耗所有可用资源，以便真正的连接无法通过。

# SSDP

SSDP（简单服务发现协议）通常用于发现**即插即用**（UPnP）设备。这正是您的家庭路由器实现的协议，所以下次您抱怨您最喜欢的游戏网络离线时，为什么不首先检查您是否无意中暴露了 SSDP 到互联网？

# ACK

ACK 洪水利用了客户端连接到服务器时存在的三次握手。第一步是客户端发送一个 SYN 数据包，服务器回复一个 SYN-ACK 数据包。然后客户端最终回复一个 ACK 数据包，然后连接就为数据开放。ACK 洪水有两种形式：

+   攻击者向服务器发送一个伪造的 SYN 数据包，然后跟随一个伪造的 SYN-ACK 数据包。然后服务器打开并保持连接。如果打开的连接足够多，那么服务器最终会耗尽资源。

+   第二种方法只是发送 ACK 数据包。由于服务器没有开放连接，这个数据包将被丢弃；然而，它仍然消耗资源来处理这些数据包。

这种攻击类似于 SYN 攻击；然而，由于它通过欺骗 DDoS 过滤器将数据包传递到服务器的方式，它可能更加高效。

避免这些攻击并不简单：你需要检测和过滤你网络边缘的这种活动。你还需要大量的带宽来吸收系统进入的流量，在我看来，这不是一个可以或应该由内部解决方案解决的问题。

避免 DDoS 攻击的第一道防线是确保你没有启用它们。配置防火墙以确保你没有暴露易受攻击的服务，并修补你的服务意味着攻击者无法利用你的网络基础设施来攻击他人。第二道防线是利用 Cloudflare、Akamai、Imperva 或其他专家的力量，他们拥有基础设施和网络清洗过滤器，以确保流量永远不会到达你的服务器。

# 应用程序安全

现在，我们希望理解一些加密工作的方式以及我们基础设施的一些脆弱性，但我们的应用程序呢？完全有可能有人想闯入你的系统。虽然 DDoS 攻击可能会让你在一天或两天内感到不便，但一个绕过你的防火墙并进入你的应用程序服务器的黑客可能会造成严重的财务或声誉损害。我们首先需要做的是基于不信任的原则。David Strauss 在他的演讲《不要构建“死亡之星”安全》（2016 年 O'Reilly 软件架构会议）中，研究了维基解密网站，并得出结论，不是第一道防线失败了，而是攻击者能够访问各种后端系统。

在同一场会议上，Sam Newman，他写了优秀的《微服务》一书（如果还没有读过，我鼓励大家阅读），也在谈论《应用安全和微服务》。Sam 表示，*微服务为我们提供了多个边界的功能*；虽然这可能是一个好处，但也可能引起问题。他提出了一种 ThoughtWorks 使用的微服务安全模型；这建议你遵循以下四个步骤：

+   预防

+   检测

+   响应

+   恢复

# 预防

预防是你应该投入最多努力的地方，本章的剩余部分将专注于这一点。这是实施安全通信、授权和认证的技术。

# 检测

检测与你的应用程序日志和 ModSecurity 日志（如果你使用它）有关。我们在上一章讨论了系统中的日志记录方法，我建议你考虑你需要检测恶意意图的日志类型，而不仅仅是用于故障排除。当你规划一个功能时，这应该成为你的非功能性要求的一部分。

# 响应

应对措施是指你如何处理安全漏洞：如果发生事件，你需要立即处理。这不仅包括将攻击者排除在系统之外，还要确定被窃取的内容，在个人信息或信用卡丢失的情况下，联系你的客户并对问题保持透明。想想如果你的公司遇到火灾，你们会如何处理消防演习。你进行练习是为了确保在发生火灾的情况下，每个人都知道该做什么以及如何快速反应。对于一些公司来说，游戏日是标准实践，他们会演练灾难恢复情况。如果你打算练习你的应对流程，你需要确保整个业务都参与其中；虽然技术部门将包括在问题的诊断和修复中，但为了真正有用，还需要在业务层面、法律、公关和沟通方面的参与。

# 恢复

恢复过程应该是最简单的步骤，假设你的基础设施已经得到了良好的备份和自动化。山姆建议不要冒险，*彻底摧毁*它，使用新的密钥和密码重新构建，以避免进一步的攻击。

# 混淆代理

混淆代理问题是指一个系统可以滥用另一个系统对其的信任，并执行它通常不允许执行的命令。考虑一个在你的系统中发放退款系统的例子；你认为这个系统是安全的，因为它是一个位于防火墙后的私有 API，但如果你防火墙被攻击者攻破了呢？如果他们能够检测到向服务器发送带有有效负载的`POST`请求会导致银行或 PayPal 账户退款，那么他们甚至不需要进一步攻击你的基础设施来获得收益。这种情况非常普遍；在构建系统时，我们过度依赖外部防御，并对防火墙后的任何事物都实行信任原则。你也可能假设攻击者实际上在你的组织外部；如果他们合法地访问了服务器怎么办？在美国，内部欺诈占金融损失的 XXX%；我们需要构建使这种情况变得困难的系统，并确保我们有完整的访问和操作审计记录。这并不需要是一个难以解决的问题；我们将看到两种非常简单的方法来解决这个问题，当实施时，既不会增加你的开发时间，也不会增加你的运营时间。

# 攻击者如何绕过防火墙

攻击者拥有多种工具来绕过你的安全系统。我们不是在谈论试图利用互联网上找到的工具利用现有漏洞的人。我们谈论的是复杂且聪明的黑客，无论出于什么原因，都决心对你的公司造成伤害。

# 场景

你是一个利用最新的微服务架构模式构建的电子商务平台。你的应用程序代码正在 Docker 容器中运行，你使用 Kubernetes 在 AWS 上托管一切。系统的前端是一个简单的 Node.js 应用程序，它与各种私有 API 通信，以提供网站上许多交易功能。应用程序本身没有数据库，容器中也没有存储任何机密。

# 攻击

攻击者发现了一个用于前端展示的模板引擎中的远程代码执行漏洞。他们发现系统运行在 Kubenettes 上，并且控制 API 在受损害的容器内部可用。他们使用这个 API 来在你的网络上启动一个恶意容器，该容器以特权模式运行，启动一个反向 SSH 隧道到攻击者的远程服务器，完全绕过了防火墙，并使他们获得了容器的 root 访问权限。从这里，他们嗅探网络流量，并确定支付网关有 `POST` 端点 v1/refunds；通过向此端点发送 JSON 负载数据，可以将大量资金退还到离岸银行账户。

即使防火墙正在保护入站流量，并且只允许端口 `80` 和 `443` 的入站，攻击者还是利用了应用程序内部的一个漏洞，为自己创建了一个后门。在生产环境中启动应用程序周围的不存在的安全性和服务之间开放的未加密通信，给了他们他们需要的一切，以清空公司的银行账户。

这是一个非常真实的威胁，但幸运的是，Go 语言有许多优秀的工具可以帮助我们使攻击者难以得逞。

# 输入验证

在我们的场景中，攻击者使用远程代码执行漏洞来获取对我们环境的访问权限。WAF 之后的第一道防线是输入验证。所有数据都应该进行验证以设置边界；实现它并不需要花费大量时间，并且可以帮助你保护免受此类攻击。Go 语言中有一个优秀的库，它是 go-playground 包的一部分（[`github.com/go-playground/validator`](https://github.com/go-playground/validator)）。

看看这个简单的代码示例，看看实现起来有多容易：

`validation/main.go`

```go
3 // Request defines the input structure received by a http handler 
4 type Request struct { 
5  Name  string `json:"name"` 
6  Email string `json:"email" validate:"email"` 
7  URL   string `json:"url" validate:"url"` 
8 } 

```

验证器包的好处是它与字段标签一起工作，这是一种保持代码整洁的无侵入方式。通过添加验证标签，我们可以为字段指定许多不同的验证函数，包括电子邮件、URL、IP 地址、最小和最大长度以及正则表达式。同一字段上也可以有多个验证器。例如，如果我想验证我的输入是一个电子邮件并且长度至少为三个，我可以添加以下内容：

```go
validate: "email,min=3" 

```

验证器按照列表中的顺序处理，所以检查字段是否包含电子邮件的验证函数会在检查长度之前进行验证。

使用这个包也非常简单：如果我们看看测试中的示例，我们可以看到验证实际上只是一个方法调用：

```go
 9 func TestErrorWhenRequestEmailNotPresent(t *testing.T) { 
10  validate := validator.New() 
11  request := Request{ 
12    URL: "http://nicholasjackson.io", 
13  } 
14 
15  if err := validate.Struct(&request); err == nil { 
16    t.Error("Should have raised an error") 
17  } 
18 } 

```

在最简单的形式中，我们只需要对请求进行验证的两个方法调用。首先，我们使用`New`函数创建一个新的验证器，就像第 10 行那样：

```go
func New() *Validate 

```

`New`函数返回一个具有合理默认值的`validate`新实例。

然后，我们可以调用`validate`方法来检查我们的结构是否有效：

```go
func (v *Validate) Struct(s interface{}) (err error) 

```

`Struct`函数验证结构体的公开字段，并自动验证嵌套结构体，除非另有说明。

对于传递的错误值，它返回`InvalidValidationError`，否则返回 nil 或`ValidationErrors`作为错误。如果您需要断言错误，它不为 nil，例如，`err.(validator.ValidationErrors)`来访问错误数组。

如果结构体有验证错误，`Struct`将返回一个错误；要获取错误详细消息，我们可以将错误转换为`ValidationErrors`对象，它是一个`FieldError`集合。要查看`FieldError`对象的全部可用方法，请查看 godoc（[`godoc.org/gopkg.in/go-playground/validator.v9#FieldError`](https://godoc.org/gopkg.in/go-playground/validator.v9#FieldError)）。

# 模糊测试

当然，我们也应该加强我们的测试技术。一种非常有效的方法是在测试中使用模糊器来测试输入验证的边界；这仅仅扩大了我们测试的范围，以确保我们覆盖了所有边缘情况。潜在的攻击者很可能会使用这种技术来测试你 API 的边界，为什么不先发制人，确保所有输入都得到正确处理呢？

在 Go 中实现 fuzzer 的最流行实现之一是出色的包 `github.com/dvyukov/go-fuzz/go-fuzz`。`go-fuzz` 是一个覆盖率引导的 fuzzer，它使用您的应用程序代码的仪器化构建，暴露出它使用的代码覆盖率，以确保最大化的代码路径被覆盖。fuzzer 生成随机输入，其目的是使应用程序崩溃或产生意外的输出。尽管 fuzzing 是一个高级主题，但在本章的代码示例 `validation/fuzzer` 中，您可以找到一个如何对刚刚覆盖的验证处理程序进行 fuzz 的示例。

# TLS

我们的攻击者利用的另一个漏洞是，防火墙后面的所有流量都没有加密，通过嗅探服务之间的流量，他们发现了一种伪造对支付网关的调用以向远程银行账户发送退款的方法。另一个问题可能是，您正在将敏感信息，如银行详情或信用卡号码，在您的前端服务和支付服务之间传递。即使您没有在您的系统上存储信用卡号码，如果您不小心，您可能会通过假设防火墙后面的所有内容都是安全的，将此流量暴露给攻击者。由于现在服务器可用的处理能力不断提高，TLS 或传输层安全性不再增加任何开销。除此之外，防火墙内部的服务通常只有有限数量的连接；因此，为了减少 TLS 握手的丢失时间，您可以在您的服务中使用持久可重用连接来最小化这个问题。让我们看看我们如何在 Go 中快速实现 TLS。

# 生成私钥

在我们进行任何操作之前，我们需要生成一个密钥和证书。Go 实际上有一个非常棒的实用工具，可以仅使用 Go 生成密钥和证书，但在我们查看这个之前，让我们看看我们如何传统地使用 `openssl` 生成证书：

```go
openssl genrsa -aes256 -out key.pem 4096  

```

这将为我们生成一个 PEM 格式的密钥，它使用 4096 位大小的 RSA 算法；密钥将使用 aes256 格式加密，并提示您输入密码。然而，我们还需要一个 X.509 证书，该证书将与这个密钥一起使用；为了生成这个证书，我们还可以再次使用 `openssl` 并执行以下命令：

```go
openssl req -new -x509 -sha256 -key key.pem -out certificate.pem -days 365  

```

此命令将使用密钥再次以 PEM 格式生成证书，有效期为一年。在实践中，我们不应该为我们的内部服务生成寿命如此长的证书。因为我们控制服务的部署，我们应该尽可能频繁地轮换密钥。关于此证书的另一件事是，虽然它是有效且安全的，但客户端不会自动信任它。这是因为根是自动生成的，而不是来自受信任的权威机构。这对于内部使用来说是完全可以的；然而，如果我们需要服务面向公众，我们就需要请求一个受信任的来源为我们生成证书。

现在我们知道了如何使用`openssl`命令行工具来完成这个操作，接下来让我们看看如何仅使用 Go 的 crypto 库实现相同的功能。示例应用程序可以在[`golang.org/src/crypto/tls/generate_cert.go`](https://golang.org/src/crypto/tls/generate_cert.go)找到，它为我们提供了这个操作的详细信息。现在让我们一步一步地查看这个过程。

如果我们查看`tls/generate_keys`中的示例，我们可以看到我们正在使用来自`crypto/edcsa`包的`GenerateKey`方法：

```go
120 func generatePrivateKey() *rsa.PrivateKey { 
121  key, _ := rsa.GenerateKey(rand.Reader, 4096) 
122  return key 
123 } 

```

第**120**行的`GenerateKey`方法的签名如下：

```go
func GenerateKey(rand io.Reader, bits int) (*PrivateKey, error) 

```

第一个参数是一个 I/O 读取器，它将返回随机数；为此，我们使用`rand.Reader`方法，这是一个全局共享的加密强伪随机生成器的实例。在 Linux 上，这将使用`/dev/urandom`，在 Windows 上使用`CryptGenRandomAPI`。第二个参数是要使用的位数大小：位数越大越安全，但会导致加密和解密操作变慢。

为了将密钥序列化到文件，我们需要执行几个不同的操作：

```go
191 func savePrivateKey(key *rsa.PrivateKey, path string, password []byte) error { 
192  b := x509.MarshalPKCS1PrivateKey(key) 
193  var block *pem.Block 
194  var err error 
195 
196  if len(password) > 3 { 
197    block, err = x509.EncryptPEMBlock(rand.Reader, "RSA PRIVATE  
KEY", b, password, x509.PEMCipherAES256) 
198    if err != nil { 
199      return fmt.Errorf("Unable to encrypt key: %s", err) 
200    } 
201  } else { 
202    block = &pem.Block{Type: "RSA PRIVATE KEY", Bytes: b} 
203  } 
204 
205  keyOut, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0600) 
206  if err != nil { 
207    return fmt.Errorf("failed to open key.pem for writing: %v", err) 
208  } 
209 
210  pem.Encode(keyOut, block) 
211  keyOut.Close() 
212 
213  return nil 
214 } 

```

在第**192**行，我们正在获取从`GenerateKey`函数返回的`PrivateKey`引用，并需要将其转换为字节数组，以便将其序列化到磁盘。`crypto/x509`包有许多有用的函数，可以启用此类操作；我们需要使用的函数是`MarshalPKCS1PrivateKey`，它将我们的基于 RSA 的私钥序列化为 ASN.1，DER 格式：

```go
func MarshalPKCS1PrivateKey(key *rsa.PrivateKey) ([]byte, error) 

```

一旦我们有了字节的密钥格式，我们就可以准备将其写入文件；然而，仅仅将字节写入文件是不够的；我们需要能够以 PEM 格式写入，如下面的示例所示：

```go
-----BEGIN RSA PRIVATE KEY----- 
Proc-Type: 4,ENCRYPTED 
DEK-Info: AES-256-CBC,c4e4be9d17fcd2f44ed4c7f0f6a9b7a8 

cfsmkm4ejLN2hi99TgxXNBfxsSQz6Pz8plQ2HJ1XToo8uXGALFlA+5y9ZLzBLGRj 
... 
zVYQvWh5NytrP9wnNogRsXqAufxf4ZLehosx0eUK4R4PsMy/VTDvcNo9P3uq2T32 

-----END RSA PRIVATE KEY----- 

```

此文件的格式如下所示，虽然我们可以手动创建此文件，但 Go 的 crypto 库已经为我们提供了支持：

```go
-----BEGIN Type----- 
Headers 
base64-encoded Bytes 
-----END Type----- 

```

我们还需要确保我们的私钥安全，所以如果指定了密码，我们将像使用命令行选项一样加密密钥。在第**196**行，我们检查是否指定了密码，如果是的话，我们将调用该方法：

```go
func EncryptPEMBlock(rand io.Reader, blockType string, data, password []byte, alg PEMCipher) (*pem.Block, error) 

```

此方法返回一个用于给定 DER 编码数据的 PEM 块，该数据使用给定的密码加密。我们在示例中使用的是 AES256 算法；然而，Go 也支持以下加密方式：

```go
const ( 
        PEMCipherDES PEMCipher 
        PEMCipher3DES 
        PEMCipherAES128 
        PEMCipherAES192 
        PEMCipherAES256 
) 

```

如果我们不想用密码加密密钥，我们需要做一些稍微不同的事情。在第 202 行，我们需要自己创建 PEM 块；`pem`包通过以下结构体为我们提供了这个功能：

```go
type Block struct { 
        Type    string            // The type, taken from the preamble (i.e. "RSA PRIVATE KEY"). 
        Headers map[string]string // Optional headers. 
        Bytes   []byte            // The decoded bytes of the contents. Typically a DER encoded ASN.1 structure. 
} 

```

无论我们使用加密的 PEM 块还是未加密的，我们都使用同一包中的`Encode`函数，它将我们的数据转换为正确的格式：

```go
func Encode(out io.Writer, b *Block) error 

```

# 生成 X.509 证书

现在我们有了私钥，我们可以继续生成我们的证书。我们已经看到使用`openssl`创建它有多容易，在 Go 中也是如此：

```go
125 func generateX509Certificate( 
126  key *rsa.PrivateKey, 
127  template *x509.Certificate, 
128  duration time.Duration, 
129  parentKey *rsa.PrivateKey, 
130  parentCert *x509.Certificate) []byte { 
131 
132  notBefore := time.Now() 
133   notAfter := notBefore.Add(duration) 
134 
135  template.NotBefore = notBefore 
136  template.NotAfter = notAfter 
137 
138  serialNumberLimit := new(big.Int).Lsh(big.NewInt(1), 128) 
139  serialNumber, err := rand.Int(rand.Reader, serialNumberLimit) 
140  if err != nil { 
141    panic(fmt.Errorf("failed to generate serial number: %s", err)) 
142  } 
143 
144  template.SerialNumber = serialNumber 
145 
146  subjectKey, err := getSubjectKey(key) 
147  if err != nil { 
148    panic(fmt.Errorf("unable to get subject key: %s", err)) 
149  } 
150 
151  template.SubjectKeyId = subjectKey 
152 
153  if parentKey == nil { 
154    parentKey = key 
155  } 
156 
157  if parentCert == nil { 
158    parentCert = template 
159  } 
160 
161  cert, err := x509.CreateCertificate(rand.Reader, template, parentCert, &key.PublicKey, parentKey) 
162  if err != nil { 
163    panic(err) 
164  } 
165 
166  return cert 
167 } 

```

我们将一些参数传递给这个方法。第一个参数可能有点奇怪，那就是模板。因为我们需要生成不同类型的证书，例如那些可以签署其他证书以创建信任链的证书，我们需要创建一个模板来使用，其中包含一些默认值。如果我们查看定义在第 22 行的`rootTemplate`，我们可以检查一些这些选项：

```go
22 var rootTemplate = x509.Certificate{ 
23  Subject: pkix.Name{ 
24    Country:            []string{"UK"}, 
25    Organization:       []string{"Acme Co"}, 
26    OrganizationalUnit: []string{"Tech"}, 
27    CommonName:         "Root", 
28  }, 
29 
30  KeyUsage: x509.KeyUsageKeyEncipherment | 
31    x509.KeyUsageDigitalSignature | 
32    x509.KeyUsageCertSign | 
33    x509.KeyUsageCRLSign, 
34  BasicConstraintsValid: true, 
35  IsCA: true, 
36 } 

```

主题，是`pkix.Name`结构体的一个实例，具有以下字段：

```go
type Name struct { 
        Country, Organization, OrganizationalUnit []string 
        Locality, Province                        []string 
        StreetAddress, PostalCode                 []string 
        SerialNumber, CommonName                  string 

        Names      []AttributeTypeAndValue 
        ExtraNames []AttributeTypeAndValue 
} 

```

这些是 X.509 区分名称的常见元素；这些元素中的大多数都很直接，代表了证书所有者的详细信息。`SerialNumber`是最重要的之一。序列号对于证书链必须是唯一的；然而，它不需要是顺序的。如果我们查看第 138 行的示例，我们正在生成一个 128 位长的大随机整数，但你可以将它改为任何你喜欢的。

我们证书生成的下一个有趣的部分是`SubjectKey`；这是为了使信任链正确工作所必需的。如果一个证书由另一个证书签署，那么权限密钥标识符将与父证书的主题密钥标识符匹配：

```go
X509v3 Subject Key Identifier: 
                5E:18:F9:33:BB:7B:E0:73:70:A5:3B:13:A8:40:38:3E:C9:4C:B4:17 
X509v3 Authority Key Identifier: 
                keyid:72:38:FD:0F:68:5C:66:77:C0:AF:CB:43:C7:91:4C:5A:DD:DC:4D:D8 

```

要生成主题密钥，我们需要将密钥的公共版本序列化为 DER 格式，然后提取仅包含密钥部分的字节：

```go
174 func getSubjectKey(key *rsa.PrivateKey) ([]byte, error) { 
175  publicKey, err := x509.MarshalPKIXPublicKey(&key.PublicKey) 
176  if err != nil { 
177    return nil, fmt.Errorf("failed to marshal public key: %s", err) 
178  } 
179 
200  var subPKI subjectPublicKeyInfo 
201  _, err = asn1.Unmarshal(publicKey, &subPKI) 
202  if err != nil { 
203    return nil, fmt.Errorf("failed to unmarshal public key: %s", err) 
204  } 
205 
206  h := sha1.New() 
207  h.Write(subPKI.SubjectPublicKey.Bytes) 
208  return h.Sum(nil), nil 
209 } 

```

在第 174 行，我们使用`x509`包中的`MarshalPKIXPublicKey`函数将公钥转换为字节数组：

```go
func MarshalPKIXPublicKey(pub interface{}) ([]byte, error) 
MarshalPKIXPublicKey serialises a public key to DER-encoded PKIX format. 

```

这返回一个表示 ASN.1 数据结构的字节数组；为了获取密钥的底层数据，我们需要将其解包到定义在第 169 行的结构体格式中：

```go
169 type subjectPublicKeyInfo struct { 
170  Algorithm        pkix.AlgorithmIdentifier 
171  SubjectPublicKey asn1.BitString 
172 } 

```

为了执行此转换，我们可以使用`Unmarshal`函数，该函数位于`encoding/asn1`包中。此方法尝试将 ASN.1 数据格式转换为：

```go
func Unmarshal(b []byte, val interface{}) (rest []byte, err error) 
Unmarshal parses the DER-encoded ASN.1 data structure b and uses the reflect package to fill in an arbitrary value pointed at by val. Because Unmarshal uses the reflect package, the structs being written to must use upper case field names. 

```

最后，在第**161**行，我们可以使用`crypto/x509`包上的`CreateCertificate`方法创建证书。此方法接受一个父证书，该证书将用于签名子证书。对于我们的根证书，我们希望它是自签名的，因此我们将父证书和私钥都设置为根证书的私钥和模板。对于中间和叶证书，我们会使用父证书的详细信息：

```go
func CreateCertificate(rand io.Reader, template, parent *Certificate, pub, priv interface{}) (cert []byte, err error) 

```

`CreateCertificate`函数基于模板创建一个新的证书。以下模板成员被使用：`SerialNumber`、`Subject`、`NotBefore`、`NotAfter`、`KeyUsage`、`ExtKeyUsage`、`UnknownExtKeyUsage`、`BasicConstraintsValid`、`IsCA`、`MaxPathLen`、`SubjectKeyId`、`DNSNames`、`PermittedDNSDomainsCritical`、`PermittedDNSDomains`和`SignatureAlgorithm`。

证书由父证书签名。如果父证书等于模板，则证书是自签名的。参数`pub`是签发者的公钥，`priv`是签发者的私钥。

现在我们有了证书，让我们看看如何使用 TLS 来保护一个网络服务器。在第一章，“微服务简介”中，你可能还记得介绍了来自标准 HTTP 包的`http.ListenAndServe`，它启动了一个 HTTP 网络服务器。当然，Go 有一个同样出色的包用于创建一个使用 TLS 加密的网络服务器。实际上，它只需要比标准的`ListenAndServe`多两个参数：

```go
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error 

```

我们需要做的只是传递我们的证书和相应的私钥的路径以及服务器启动时将使用 TLS 来处理流量。如果我们使用自签名证书，在我们的例子中就是这样，那么我们需要为我们的客户端编写一些额外的代码，否则当我们尝试连接到服务器时，我们会收到如下错误信息：

```go
2017/03/19 14:29:03 Get https://localhost:8433: x509: certificate signed by unknown authority 
exit status 1 

```

为了避免这种情况，我们需要创建一个新的证书池并将它传递给客户端的 TLS 设置。默认情况下，Go 将使用主机的根 CA 集合，它将不包括我们的自签名证书：

```go
13 roots := x509.NewCertPool() 
14 
15 rootCert, err := ioutil.ReadFile("../generate_keys/root_cert.pem") 
16 if err != nil { 
17  log.Fatal(err) 
18 } 
19 
20 ok := roots.AppendCertsFromPEM(rootCert) 
21 if !ok { 
22   panic("failed to parse root certificate") 
23 } 
24 
25 applicationCert, err := ioutil.ReadFile("../generate_keys/application_cert.pem") 
26 if err != nil { 
27  log.Fatal(err) 
28 } 
29 
30 ok = roots.AppendCertsFromPEM(applicationCert) 
31 if !ok { 
32  panic("failed to parse root certificate") 
33 } 
34 
35 tlsConf := &tls.Config{RootCAs: roots} 
36 
37 tr := &http.Transport{TLSClientConfig: tlsConf} 
38 client := &http.Client{Transport: tr} 

```

在第**13**行，我们创建了一个新的证书池，然后读取证书，这些证书被 PEM 编码成字节数组。在第**20**行，我们可以将这些证书添加到新的证书池中；为了使证书被识别为有效，我们需要中间证书和根证书。然后我们可以创建一个新的 TLS 配置并添加证书；然后将其添加到传输中，最终在第**38**行添加到客户端。

当我们现在运行客户端时，它没有任何问题地连接，我们将看到服务器正确返回的`Hello World`响应。

# 保护静态数据

假设我们的系统已经连接到数据库来存储诸如用户账户之类的信息，攻击者将能够访问完整的密码数据库。当我们把数据存储在数据库中时，我们应该考虑的一个问题是我们的数据加密。毫无疑问，加密数据比不加密数据更昂贵，有时很难确定我们应该加密哪些字段或表，哪些应该保持未加密状态。

微服务给我们带来的许多好处之一是我们可以在系统中分离功能和数据。这使得决定要加密哪些数据变得更加容易，因为与其试图理解数据存储中哪些数据需要加密，你只需做出一个更简单的决定：在这个数据存储中是否有任何需要加密的数据？如果有，那么只需简单地加密所有数据。在应用层而不是数据存储中执行此加密可能更有益，因为应用通常比数据存储扩展得更好，你必须考虑缓存可能引入的边缘情况。如果你为了减轻数据存储的压力，使用 Elasticache 或其他技术添加一个中间缓存层，你需要考虑你数据的安全性。如果数据在数据库中加密，那么你需要确保缓存也应用相同级别的加密。

# 物理机器访问

当我说“物理上”时，我的意思是人类可以访问；代码可能运行在虚拟机上。然而，问题是一样的：我常常发现，公司给开发者访问在生产环境中运行的数据库和其他信息源。即使他们没有访问数据库密码，他们可能有权访问配置存储或能够通过 SSH 连接到应用程序服务器并从应用程序中读取配置。有一个名为**最小权限**的安全原则；这建议账户和服务应具有执行其业务功能所需的最小权限。即使你已经确保了机器到机器的通信是安全的，并且你的防火墙有适当的保护措施，攻击者总有通过后门访问你的系统的机会。考虑以下场景。你公司的一名非技术员工打开了一封电子邮件或下载了一些软件，这些软件在他们的笔记本电脑上安装了恶意软件。攻击者利用这一点来获取对他们的机器的访问权限，并从那里成功地在网络中横向移动，最终到达你的笔记本电脑。现在，由于你登录并忙于工作，并通过 VPN 连接到生产环境，他们设法在你的机器上安装了一个键盘记录器，这使他们能够访问你的密码，他们从你的磁盘中检索你的 SSH 密钥，因为你几乎可以完全访问生产环境，现在他们也一样。虽然这听起来像是科幻小说，但这是完全可能的。当然，你可以保护你的内部网络，但避免这种攻击的最佳方法是严格限制对生产环境和数据的访问。你永远不需要这种级别的访问权限；在我的代码中进行稳健的测试时，我常常发现，当一项服务表现不佳时，生产访问并不能帮助我解决问题。我应该能够在预演环境中重现几乎任何错误，并且该服务发出的日志和度量数据应该足以让我诊断任何问题。我并不是说我从未在生产环境中实时调试过，但幸运的是，在过去的十年中我没有这样做。工具和实践如此之先进，以至于我们永远不需要回到那些行为。

# OWASP

无论何时你在寻找关于安全的实际网络安全建议，OWASP 几乎总是你的首选。

对于 API 的帮助，OWASP 也可以提供帮助：他们已经发布了*REST 安全速查表*([`www.owasp.org/index.php/REST_Security_Cheat_Sheet`](https://www.owasp.org/index.php/REST_Security_Cheat_Sheet))。

当然，正如我们在本书中已经讨论过的，构建 API 有许多不同的标准，REST 只是其中之一；然而，我们可以从本指南中利用一些有用的通用技巧。

# 永远不要在 URL 中存储会话令牌

JWT（JSON Web Token），可能是您在 API 中最常见的会话令牌，编码成 URL 安全的格式。然而，将令牌存储或传递到 URL 中并不推荐，并且它始终应存储在 cookie 中或作为`POST`变量。这样做的原因是，如果您在 URL 中传递会话令牌，这些令牌可能会泄漏到您的服务器日志中，并且根据您如何管理令牌的持续时间，如果攻击者访问了您的日志文件，他们也可能能够获得完全访问权限来执行用户的命令。

# 跨站脚本（XSS）和跨站请求伪造（CRSF）

XSS（跨站脚本）和 CRSF（跨站请求伪造）仅在 API 将从网络浏览器中使用时适用，例如在单页应用或直接 JavaScript 调用中。然而，为了防止攻击者注入恶意 JavaScript 以检索您的会话令牌，您应确保它存储在标记为 HTTP-only 的 cookie 中，并且您始终通过 HTTPS 发送它们以防止它们在传输中被捕获。除此之外，我们还可以添加一层安全措施，该措施检查浏览器发送的 HTTP 引用与预期域名是否匹配。虽然使用类似 cURL 的工具可以伪造 HTTP 引用，但在浏览器中的 JavaScript 中这样做是不可能的或极其困难的。

# 不安全直接对象引用

当您构建 API 时，您需要确保您正在检查认证用户是否可以修改请求中的对象。这将由服务器端执行；我们不希望给我们的攻击者创建一个真正的登录并能够操纵请求以代表其他用户执行操作的能力。

OWASP 文档会定期更新，因为发现了新的攻击和漏洞；经常检查网站并保持自己最新。

# 认证和授权

认证是检查某事是否为真的过程或行为，例如：此用户名是否与该密码匹配？授权是指定访问权限或关于用户的策略的功能。

认证是一个被广泛理解的概念；然而，我们需要理解一些概念以确保此操作不会被破坏，例如永远不要在数据存储中以纯文本形式存储密码，并防止通过将活动令牌转移到第三方来劫持登录会话。然而，授权同样重要；我们之前讨论了困惑代理问题：即使用户已通过认证，我们仍然必须控制他们在系统上可以执行的操作。在自身之间建立信任原则且不独立验证用户权利的服务，如果攻击者破坏了您的防火墙，就会很容易受到滥用。在本节中，我们将探讨解决这两个问题有多么容易，并提供模式，以确保您的服务永远不会暴露。

# 密码散列

哈希是一种单向加密：你取一系列字母和数字，通过运行哈希算法，你得到一个序列，虽然可以通过相同的原始输入重新生成，但无法从数学上逆向。那么，为什么你会使用哈希而不是直接加密数据呢？首先，哈希不需要任何密钥，因此它们不会因为私钥的丢失而受到威胁，它们非常适合存储不需要逆向的数据。考虑密码：你的系统永远不需要知道用户的密码是什么，它只需要知道用户传递给你的值与存储的值匹配。哈希密码是完美的安全方法：你哈希用户的输入，并将这个哈希值与你的数据存储中的值进行比较。如果数据库被破坏，攻击者将无法解码密码。当然，攻击者可以尝试暴力破解密码，但截至目前，地球上没有足够的计算能力来解码一个合理的哈希值。这意味着哈希是不可攻破的吗？不。事实上，许多人认为 MD5 哈希是不可逆的；然而，这个算法已经被破坏。可以在几秒钟内找到碰撞。2011 年有一个案例，攻击者利用这个漏洞创建了假的 SSL 证书，这允许他们利用用户的信任。幸运的是，我们不再使用 MD5 或 SHA-1 进行加密目的。你仍然可以在像 git 提交这样的签名中找到它，那里的碰撞可能性被计算速度所抵消，但对于安全性，我们需要使用更现代的算法。

# 添加一点调味料

虽然单独的哈希值可以提供相当的安全级别，但我们还可以添加盐和胡椒。盐与加密数据一起存储在数据库中。这样做背后的意图是使暴力破解数据变得更加计算密集。这阻止了攻击者使用彩虹表来攻击你的数据。彩虹表是预先计算好的哈希值表，因此，你不需要在每次尝试时都计算哈希值，只需在表中查找加密字符串并返回原始值即可。为了应对这种情况，我们添加了一个盐，这个盐是随机生成的，并在哈希之前附加到数据上。尽管我们需要将它与哈希值一起存储在数据库中以便稍后检查值，但它阻止了彩虹表的使用，因为每个表都必须为每个盐值重新计算，这非常计算密集。为了进一步增强安全性，我们通常还会添加一个胡椒，这是一个存储在盐和哈希值之外的预计算值。

常规做法是预先生成一个胡椒列表并将其存储在配置存储中。当你第一次哈希密码或其他值时，你会随机选择一个胡椒并将其附加到值上，就像你为盐做的那样。然后，当你检查提供的值是否与哈希匹配时，你会遍历胡椒列表并为每个胡椒生成一个哈希值进行比较。这会在你的服务中检查密码时增加一点计算时间；然而，这远不如它会给试图暴力破解你的值的攻击者带来的工作量。让我们看看我们如何使用盐和胡椒来哈希一个值。

如果我们查看`hashing/hash.go`中的源代码，我们可以使用以下`GenerateHash`方法从一个输入字符串创建一个哈希值。`GenerateHash`方法具有以下签名，并且给定一个输入字符串，它使用 sha512 算法返回一个随机盐和哈希字符串：

```go
func GenerateHash(input string) (hash string, salt string) 

```

要使用这个功能，我们只需用我们的字符串来调用方法，然后我们会得到以下输出：

```go
h:= New(peppers) 
hash, salt := h.GenerateHash("HelloWorld1") 

fmt.Println("Salt: ", salt) 
fmt.Println("Hash: ", hash) 

---Output 
Salt:  15f42f8b4f1c71dc6183c822fcf28b3c34564c32339509c2c02fa3c4dda0ed4f 
Hash:  b16a89d3c41c9fe045a7c1b892d5aa15aee805d64b753e003f7050851ef4d374e3e16ce23500020746174f7b7d8aeaffebf52939f33c4fda505a5c4e38cdd0e1 

```

让我们更深入地看看这个函数在做什么：

```go
22 // GenerateHash hashes the input string using sha512 with a salt and pepper. 
23 // returns the hash and the salt 
24 func (h *Hash) GenerateHash(input string) (hash string, salt string) { 
25  pepper := h.getRandomPepper() 
26  salt = h.generateRandomSalt() 
27  hash = h.createHash(input, salt, pepper) 
28 
29  return 
30 } 

```

我们首先做的事情，在第**25**行，是从传递给结构体的胡椒切片中随机获取一个胡椒。胡椒不需要存储在数据库中；这纯粹是为了通过要求攻击者将他们的暴力破解尝试次数增加五倍来减缓潜在的攻击者。在我们的例子中，我们可以增加胡椒的数量，正如你从下面的基准测试中可以看到，即使有 1,000 个胡椒，我们仍然能够在 1 毫秒内比较一个哈希值。但是，为了获得这种额外的安全性，可能并不值得。生成一个哈希值需要 4,634 纳秒，由于字符串的长度，我们需要生成最大为 6.2e19 或 62 千万亿种排列组合。这是假设有 63 个允许的字符和一个 11 个字符长度的密码。这是一个相当大的数字，要生成这么多的哈希值，对于一个单核 CPU 来说，大概需要 9 百万年的时间来暴力破解。

# 字典攻击

然而，并非所有密码都复杂，许多密码容易受到一种称为字典攻击的攻击。字典攻击不会尝试 62 千万亿种排列组合，而是集中在最有可能成功的那些上。这些字典本身通常是从之前被利用的密码数据库中派生出来的，由于人类的行为有一定的可预测性，我们经常使用相同的密码。因为我们的密码`HelloWorld1`已经在包含 1400 万个其他密码的字典中，当我尝试使用`John the Ripper`破解加盐的哈希时，只用了 2.4 秒就检索到了密码。

# 添加胡椒

在阻止用户使用简单密码方面，我们能够做的事情之间有一条很细的界限。我们应该始终有一个政策来定义什么是一个好的密码--最小长度、大小写混合、添加符号等等--但是随着密码的复杂度增加，可用性可能会受到影响。然而，添加 pepper 可以帮助减缓攻击者的速度：pepper 或 peppers 为系统所知，但不会与密码和盐一起存储。它们可以硬编码到应用程序代码中，作为启动配置存储，或者在运行时从安全保险库中存储。同样，我们在用户的密码中附加了盐，我们也对 pepper 做同样的事情。如果数据库表因 SQL 注入攻击而受损，除非攻击者能够检索到 peppers，否则数据库将毫无用处。当然，攻击者可能能够获取到你的 peppers；然而，在安全领域，几乎一切都是关于使事情变得困难并减缓某人的速度。

# bcrypt

bcrypt 是另一种流行的密码散列方法，它使用可变次数的轮次来生成散列，这既减缓了暴力破解攻击的能力，也减缓了生成散列的时间。Go 有一个 bcrypt 的实现，由实验性包提供，位于[`godoc.org/golang.org/x/crypto/bcrypt`](https://godoc.org/golang.org/x/crypto/bcrypt)。要使用 bcrypt 散列密码，我们使用`GenerateFromPassword`方法：

```go
func GenerateFromPassword(password []byte, cost int) ([]byte, error)

```

`GenerateFromPassword`方法返回给定成本的密码的 bcrypt 散列。成本是一个变量，允许你在生成散列时增加处理时间以换取返回散列的安全性。

要检查 bcrypt 散列的相等性，我们不能再次使用给定的密码调用`GenerateFromPassword`并比较输出与存储的散列，因为`GenerateFromPassword`每次运行都会创建不同的散列。为了比较相等性，我们需要使用`CompareHashAndPassword`方法：

```go
func CompareHashAndPassword(hashedPassword, password []byte) error

```

`CompareHashAndPassword`方法比较 bcrypt 散列密码与其可能的纯文本等效密码。bcrypt 是保护密码的一种安全方法，但它很慢，让我们更深入地看看生成散列的成本。

# 基准测试

下表说明了使用我们迄今为止讨论的方法生成和比较散列字符串的大致时间。即使有 1,000 个 peppers，我们也会看到大约 1.5 毫秒的处理时间来运行比较。这可能看起来不是很多时间；然而，我们需要带着一点盐来对待这些基准，因为它们正在运行单个操作，而你的服务器将同时处理多个请求。我们知道的是，与 100 个列表相比，比较 1,000 个 peppers 的列表需要 10 倍的时间，而这比 10 个列表的时间还要长 10 倍：

即使有 1,000 个辣椒，我们运行比较的时间大约为 1.5 毫秒。这可能看起来不是很多时间；然而，我们需要带着一点盐来对待这些基准，因为它们正在运行一个单一操作，而你的服务器将同时处理多个请求。我们知道的是，与 1,000 个辣椒的哈希比较需要比 100 个的哈希比较长 10 倍，而这比 10 个的哈希比较长 10 倍：

```go
BenchmarkGeneratePlainHash-8             30000000     1069 ns/op
BenchmarkGenerateHashWithSaltAndPepper-8  5000000     5223 ns/op
BenchmarkGenerateBcrypt-8                     500 68126630 ns/op
BenchmarkCompareSaltedHash-8             20000000     1276 ns/op
BenchmarkComparePlainHash-8              20000000     1174 ns/op
BenchmarkCompareHash5Peppers-8           20000000     4980 ns/op
BenchmarkCompareHash10Peppers-8          10000000     4669 ns/op
BenchmarkCompareHash100Peppers-8          1000000    22150 ns/op
BenchmarkCompareHash1000Peppers-8           20000  1492037 ns/op
BenchmarkCompareBCrypt-8                      500 70942742 ns/op

```

根据这些信息，我们可以平衡我们的服务在速度和安全性之间的平衡；然而，我们应该始终倾向于更安全的选择。要了解 Facebook 如何管理哈希，我建议您观看 Alec Muffett 的演讲 Facebook：*密码哈希和身份验证*（[`www.youtube.com/watch?v=NQDo2e3gj1A`](https://www.youtube.com/watch?v=NQDo2e3gj1A)）。

# JWTs

**JSON Web Token**（JWT）是一种在环境中安全传递与用户相关的声明或数据的标准。这是一个极其流行的标准，几乎适用于所有主要的语言和框架，当然也包括 Go。JWT 有两个主要优势。一是声明的一个标准格式，这使得可靠框架的可用性成为可能。另一个是使用非对称加密，这意味着因为令牌是经过签名的，所以接收者只需要签名人公钥来验证令牌确实来自可信源，这允许我们将对私钥的访问锁定到授权服务器。

# JWT 的格式

JWT 被分成三个不同的部分，这些部分被编码为 Base64-URL。像标准 Base64 一样，Base64-URL 用`-`和`_`替换了`+`和`/`，并移除了所有填充。这使得令牌可以在 URL 中安全地传输。

结果是一个看起来像以下示例的令牌：

```go
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhY2Nlc3NMZXZlbCI6InVzZXIiLCJleHAiOjE4MDc3MDEyNDYsInVzZXJJRCI6ImFiY3NkMjMyamZqZiJ9.iQxUbQuEy4Jh4oTkkz0OPGvS86xOWJjdzxHHDBeAolv0982pXKPBMWskSJDF3F8zd8a8nMIlQ5m9tzePoJWe_E5B9PRJEvYAUuSp6bGm7-IQEum8EzHq2tMvYtPl9uzXgOU4C_pIjZh5CjFUeZLk5tWKwOOo8pW4NUSxsV2ZRQ_CGfIrBqEQgKRodeLTcQ4wJkLBILBzmAqTVl-5sLgBEoZ76C_gcvS6l5HAwEAhmiCqtDMX46o8pA72Oa6NiVRsgxrhrKX9rDUBdJAxNwFAwCjTv6su0jTZvkYD80Li9aXiMuM9NX7q5gncbEhfko_byTYryLsmmaUSXNBlnvC_nQ 

```

令牌的三个不同部分是头部、载荷和签名。头部声明了编码对象的类型和加密签名的算法：

```go
{ 
  "alg": "RS256", 
  "typ": "JWT" 
} 

```

第二个对象载荷，包含与令牌相关的声明的详细信息：

```go
{ 
  "userID": "abcsd232fjfj", 
  "accessLevel": "user" 
} 

```

最后，第三部分是签名，这是一个可选元素，在解码状态下如下所示：

```go
Tm 
  <a=<kNX[d\1k$H_3w5C7NAIR1b 
                           Hy 
1TyՊ5D]Ehuq0&B s 
V_{@! 39Tl5t17@(ӿ.پF5~ H_6+&\[1m% 

```

JWT 中的每个元素都是 base64URL 编码的（[`en.wikipedia.org/wiki/Base64#URL_applications`](https://en.wikipedia.org/wiki/Base64#URL_applications)）；表示为其二进制形式的签名是以下格式的消息的 sha256：

```go
Base64URL(header).Base64URL(payload) 

```

签名的格式可以是对称的（HS256），使用共享密钥，或者非对称的（RS256），使用公钥和私钥。对于 JWT，最佳选项是非对称选项，因为对于需要验证 JWT 的服务，它只需要密钥的公钥部分。

我们可以使用命令行仅验证我们的 JWT。首先，我们需要将我们的 base64URL 编码的签名转换为标准 base64 编码，通过将 `_` 替换为 `/` 和 `-` 替换为 `+`。然后，我们可以将此通过管道传递到 base64 命令行应用程序，并传递 `-D` 标志以解码输入；然后将其输出到一个文件中：

```go
cat signature.txt | sed -e 's/_/\//g' -e 's/-/+/g' | base64 -D > signature.sha256 

```

下一步是验证签名是否由正确的密钥签名，通过将其与公钥进行验证：

```go
openssl dgst -sha256 -verify ../keys/sample_key.pub -signature signature.sha256 data.txt 

```

使用 Go 生成 JWT 非常简单，多亏了一些出色的社区包。在示例代码中我们将使用的包名为 **jose**，由 Eric Largergren 创建（[`github.com/SermoDigital/jose`](https://github.com/SermoDigital/jose)）。根据 `jwt.io` 上的列表，此包实现了标准中定义的所有功能，并且在写作时是一个显而易见的选择。

如果我们查看文件 `chapter8/jwt/jwt.go` 并查看 `GenerateJWT` 方法，我们可以看到使用 jose 创建 JWT 是多么简单：

```go
30 // GenerateJWT creates a new JWT and signs it with the private key 
31 func GenerateJWT() []byte { 
32  claims := jws.Claims{} 
33  claims.SetExpiration(time.Now().Add(2880 * time.Minute)) 
34  claims.Set("userID", "abcsd232jfjf") 
35  claims.Set("accessLevel", "user") 
36 
37  jwt := jws.NewJWT(claims, crypto.SigningMethodRS256) 
38 
39  b, _ := jwt.Serialize(rsaPrivate) 
40 
41  return b 
42 } 

```

我们首先需要生成一个声明列表并设置一个过期时间；我们将过期时间设置为两周。然后我们可以使用 `Set` 函数设置一个声明列表：

```go
func (c Claims) Set(key string, val interface{}) 

```

最后，在行 **39** 中，我们可以通过将声明和签名方法传递给 `NewJWT` 函数来创建一个新的 JWT：

```go
func NewJWT(claims Claims, method crypto.SigningMethod) jwt.JWT 

```

然后，我们可以调用 `Serialize` 方法，该方法以私钥作为参数——在我们的例子中是一个 `rsa.PrivateKey` 的实例——并返回一个编码格式的字节数组：

```go
func (j *jws) Serialize(key interface{}) ([]byte, error) 

```

使用 jose 验证 JWT 与创建 JWT 一样简单：

```go
46 func ValidateJWT(token []byte) error { 
47  jwt, err := jws.ParseJWT(token) 
48  if err != nil { 
49    return fmt.Errorf("Unable to parse token: %v", err) 
50  } 
51 
52  if err = jwt.Validate(rsaPublic, crypto.SigningMethodRS256); err != nil { 
54    return fmt.Errorf("Unable to validate token: %v", err) 
55  } 
56 
57  return nil 
58 } 

```

我们首先需要做的是使用 `ParseJWT` 函数将我们的 JWT 从字节数组解析到 `jwt` 结构体中：

```go
func ParseJWT(encoded []byte) (jwt.JWT, error) 

```

然后，我们可以调用 `Validate` 方法，传递与私钥对应的公钥以及签名方法。可选地，我们可以提供一个自定义验证函数；默认验证将仅检查签名以及令牌是否未过期：

```go
func (j *jws) Validate(key interface{}, m crypto.SigningMethod, v ...*jwt.Validator) error 

```

当验证失败时，将返回一个错误；如果错误为 nil，则令牌有效，其中的声明可以信赖。

# 安全消息

当我们需要发送加密消息时，最好的方法之一是使用非对称加密，其中我们使用公开信息加密消息，这些信息可以轻松分发，然后使用由单个方安全持有的私钥解密它。

Go 中的加密包包含了我们保护数据所需的所有功能。如果我们查看示例文件 `chapter8/asymmetric/asymmetric.go`，`EncryptDataWithPublicKey` 方法是 `rsa` 包公钥加密的一个简单实现：

```go
func EncryptOAEP(hash hash.Hash, random io.Reader, pub *PublicKey, msg []byte, label []byte) ([]byte, error) 

```

第一个参数是一个加密哈希，用作加密前处理消息的随机预言机。这个函数在加密和解密时必须相同，文档建议使用 sha256。下一个参数是随机数生成器；它用作熵的来源，以确保如果你两次加密相同的消息，不会返回相同的密文。`pub` 是我们想要用来加密消息的 `rsa.PublicKey`；消息本身作为字节数组切片传递。最后一个参数是可选的，并且在结果密文中不加密；它可以用来帮助接收者理解信息，例如使用了哪个密钥来加密消息，但必须极端小心，不要向标签中添加可能危及加密消息安全性的数据：

```go
41 // EncryptMessageWithPublicKey encrypts the given string and retuns the encrypted 
42 // result base64 encoded 
43 func EncryptMessageWithPublicKey(message string) (string, error) { 
44 
45  modulus := rsaPublic.N.BitLen() / 8 
46  hashLength := 256 / 4 
47  maxLength := modulus - (hashLength * 2) - 2 
48 
49  if len(message) > maxLength { 
50    return "", fmt.Errorf("The maximum message size must not exceed: %d", maxLength) 
51  } 
52 
53  data, err := EncryptDataWithPublicKey([]byte(message)) 
54  return base64.StdEncoding.EncodeToString(data), err 
55 }  

```

在这种方法中，我们首先要检查消息是否短于这种加密方法允许的最大长度。最大长度必须不大于公模数减去两倍的哈希长度再减去两个。由于公钥密码学中涉及的数学，我们只能允许加密短消息。我们稍后会看看如何解决这个问题。在第 **53** 行，我们调用另一个内部函数，该函数简单地调用 `rsa` 包中的 `EncryptOAEP` 函数。然后我们将数据编码为 base64 并返回结果。

解密数据很简单：

```go
57 // DecryptMessageWithPrivateKey decrypts the given base64 encoded ciphertext with 
58// the private key and returns plain text 
59 func DecryptMessageWithPrivateKey(message string) (string, error) { 
60  data, err := base64.StdEncoding.DecodeString(message) 
61  if err != nil { 
62    return "", err 
63  } 
64 
65  data, err = DecryptDataWithPrivateKey(data) 
66  return string(data), err 
67 } 

```

由于我们的加密方法实现返回了一个 base64 编码的字符串，我们在解密消息之前首先要做的是将其解码回二进制形式。然后我们调用内部方法 `DecryptDataWithPrivateKey`；这是一个 `rsa.DecryptOAEP` 方法的包装器：

```go
func DecryptOAEP(hash hash.Hash, random io.Reader, priv *PrivateKey, ciphertext []byte, label []byte) ([]byte, error) 

```

这种方法与加密方法的参数相同，只是这次我们使用的是私钥。如果我们回顾非对称加密的工作原理，我们可以用公钥加密，但不能用公钥解密消息。必须使用私钥来完成这个目的。

# 共享密钥

对称加密也有其用途：一方面，它更快，另一方面，它可以处理任何大小的消息。在 Go 中实现对称加密，正如你所期望的，相当简单：我们有优秀的 `crypto/aes` 包，它为我们处理所有繁重的工作。让我们看看如何使用 AES 加密消息。查看示例文件 `symmetric/symmetric.go`：

```go
12 func EncryptData(data []byte, key []byte) ([]byte, error) { 
13  if err := validateKey(key); err != nil { 
14    return make([]byte, 0), err 
15  } 
16 
17  c, err := aes.NewCipher(key) 
18  if err != nil { 
19    return make([]byte, 0), err 
20  } 
21 
22  gcm, err := cipher.NewGCM(c) 
23  if err != nil { 
24    return make([]byte, 0), err 
25  } 
26 
27  nonce := make([]byte, gcm.NonceSize()) 
28  if _, err = io.ReadFull(rand.Reader, nonce); err != nil { 
29    return make([]byte, 0), err 
30  } 
31 
32  return gcm.Seal(nil, nonce, data, nil), nil 
33 } 

```

在第 **13** 行，我们首先需要做的是验证密钥的长度。密钥的长度决定了加密的强度；16 字节密钥将使用 AES-128 加密，24 字节 AES-192，32 字节 AES-256。然后我们创建一个新的 **GCM**（**Galois/Counter Mode**）加密器，并将我们的 AES 加密器的引用传递给它：

```go
func NewGCM(cipher Block) (AEAD, error) 

```

然后，我们需要创建一个用于防止重放攻击的 nonce，最后我们可以调用`Seal`方法来加密我们的数据：

```go
Seal(dst, nonce, plaintext, additionalData []byte) []byte 

```

与 RSA 公钥加密不同，AES 可以处理的消息大小几乎是无限的；然而，问题在于密钥必须由作者和读者共享，这引入了向双方分配密钥的问题。

解密的工作方式与加密方法相反，一个例子可以在下一个代码块中看到：

```go
35 // DecryptData decrypts the given data with the given key 
36 func DecryptData(data []byte, key []byte) ([]byte, error) { 
37  c, err := aes.NewCipher(key) 
38  if err != nil { 
39    return make([]byte, 0), err 
40  } 
41 
42  gcm, err := cipher.NewGCM(c) 
43  if err != nil { 
44    return make([]byte, 0), err 
45  } 
46 
47  nonceSize := gcm.NonceSize() 
48  if len(data) < nonceSize { 
49    return make([]byte, 0), fmt.Errorf("ciphertext too short") 
50  } 
51 
52  nonce, ciphertext := data[:nonceSize], data[nonceSize:] 
53  return gcm.Open(nil, nonce, ciphertext, nil) 
54 } 

```

在这个代码块中，我们需要注意的主要是`gcm.Open`方法：

```go
Open(dst, nonce, ciphertext, additionalData []byte) ([]byte, error) 

```

我们将要用来解密消息的 nonce 需要与加密消息时使用的相同。当我们调用`Seal`时，从方法返回的字节切片是加密消息和 nonce，因此要检索它，我们只需要计算 nonce 的大小，然后将字节切片分成两部分。

# 处理大消息的非对称加密

我们已经讨论了非对称加密的问题，即它只能用于相对较小的消息；然而，不需要处理密钥分配的好处相对于对称加密来说是非常有优势的。对此问题有一个常见的解决方案；该解决方案是创建一个随机密钥，然后对称加密一个消息，然后非对称加密密钥并将两部分都分发给接收者。只有私钥的持有者才能解密对称密钥，只有当对称密钥被解密后，接收者才能解密主消息：

```go
69 // EncryptLargeMessageWithPublicKey encrypts the given message by randomly generating 
70 // a cipher. 
71 // Returns the ciphertext for the given message base64 encoded and the key 
72 // used to encypt the message which is encrypted with the public key 
73 func EncryptLargeMessageWithPublicKey(message string) (ciphertext string, cipherkey string, err error) { 
74  key := utils.GenerateRandomString(16) // 16, 24, 32 keysize, random string is 2 bytes per char so 16 chars returns 32 bytes 
75  cipherData, err := symetric.EncryptData([]byte(message), []byte(key)) 
76  if err != nil { 
77    return "", "", err 
78  } 
79 
80  cipherkey, err = EncryptMessageWithPublicKey(key) 
81  if err != nil { 
82    return "", "", err 
83  } 
84 
85  return base64.StdEncoding.EncodeToString(cipherData), cipherkey, nil 
86 } 

```

查看`asymmetric/asymmetric.go`中的示例，我们可以看到我们正在做的是确切这样。这个函数简单地封装了我们在本章前面看到的对称和非对称加密的两种方法。解密很简单：

```go
 88 // DecryptLargeMessageWithPrivateKey decrypts the given base64 encoded message by 
 89 // decrypting the base64 encoded key with the rsa private key and then using 
 90 // the result to decrupt the ciphertext 
 91 func DecryptLargeMessageWithPrivateKey(message, key string) (string, error) { 
 92   keystring, err := DecryptMessageWithPrivateKey(key) 
 93  if err != nil { 
 94    return "", fmt.Errorf("Unable to decrypt key with private key: %s", err) 
 95  } 
 96 
 97  messageData, err := base64.StdEncoding.DecodeString(message) 
 98  if err != nil { 
 99    return "", err 
100  } 
101 
102  data, err := symetric.DecryptData(messageData, []byte(keystring)) 
103 
104  return string(data), err 
105 } 

```

# 维护

保持你的系统安全的一个重要元素是确保你使用所有最新的安全补丁来更新它。这种方法需要应用于你的应用程序代码和服务器操作系统以及应用程序，如果你使用 Docker，你还需要确保你的容器是最新的，以确保你免受漏洞的侵害。

# 修补容器

确保你的容器安全的最简单方法之一是定期构建和部署它们。很多时候，如果一个服务没有处于活跃开发状态，那么它可能连续数月都不会被部署到生产环境中。正因为这个问题，你可能需要修补主机级别的应用程序库，比如 OpenSSL，但由于容器提供的应用隔离，你可能在容器级别有易受攻击的二进制文件。保持事物更新的最简单方法是定期运行构建和部署，即使应用程序代码没有变化。你还需要确保，如果你在 Dockerfile 中使用基础容器，这个容器也需要构建和更新。

Docker Hub、`quay.io` 和其他几个软件即服务注册表具有在链接容器发生变化时自动重建容器的功能。如果你正在构建基于 `golang:latest` 的镜像，你可以在上游镜像推送到注册表时自动触发构建。你还可以运行自动化的安全扫描，该扫描检查你的镜像层，并扫描任何 CVE 漏洞。它会告诉你漏洞存在于哪一层，通常你会发现这通常是在基础层，例如 Ubuntu 或 Debian。

# 软件更新

在你的主机和 Docker 镜像上修补软件可以帮助你免受诸如 Heartbleed 这样的漏洞的侵害，该漏洞在 OpenSSL 中被发现。修补软件更新相对简单。你可以配置你的主机自动更新自己；另一个选项，我更喜欢，是确保你的基础设施是自动化的，这样你就可以将其烧毁并重建。

# 修补应用程序代码

就像主机上的软件需要更新一样，你也必须更新你的应用程序代码，以确保你总是拥有最新的更新。通常，人们会锁定应用程序依赖项到某个版本，并且随着 Go 1.5 中引入的 vendoring 支持功能，这个过程在社区中越来越流行。这个问题的一个主要原因是，在 go 1.5 之前的版本中，vendoring 并未出现，这是为了鼓励你针对最新的包构建应用程序代码，并且尽早而不是晚些时候修复任何由于破坏性 API 变化而产生的问题。如果你确实使用了 vendoring，我当然不会建议你不使用它，那么你应该运行一个夜间构建，将所有库更新到最新版本。你不必一定将此部署到生产环境中；然而，如果测试通过，为什么不呢？如果测试失败，那么，即使这是一个未处于积极开发状态的服务，这也应该成为你进行一些维护的触发器。

# 日志记录

即使我们已经保护了密码并实施了良好的安全措施，我们仍然需要知道我们何时受到威胁。在前一章中，我们介绍了日志记录，日志记录可以作为你安全策略的一部分的有用工具。考虑有人试图暴力破解你的应用程序登录；当需要对此类威胁做出反应时，跟踪高水平的身份验证错误以及源 IP 地址可能是有用的。攻击者的 IP 地址可以被防火墙阻止。

日志文件的内容需要考虑以下属性：

+   谁在执行操作

+   什么失败了或成功了

+   行动发生的时间

+   为什么会失败或成功

+   你如何处理这个问题

以下示例包含的信息远远不足以有用，实际上，除了让您知道可能存在故障外，您甚至可以不占用此类日志的空间：

```go
Aug 18 11:00:57 [AuthController] Authentication failed. 

```

以下示例要好得多；它更深入地展示了问题，并详细描述了用户为访问系统所采取的事件。现代日志评估工具，如 Kibana，允许过滤和分组此类日志文件，这使您能够构建仪表板或事件列表：

```go
Aug 18 11:00:57 [AuthController] Authentication failure 
for nicj@example.com by 127.0.0.1 - user unknown - 
/user/login /user/myaccount 
Aug 18 11:01:18 [AuthController] Authentication failure 
for nicj@example.com by 127.0.0.1 - invalid password - 
/user/login?err=1 /user/login 
Aug 18 11:02:01 [AuthController] Authentication failure 
for nicj@example.com by 127.0.0.1 - incorrect 2FA code 
- /user/login?err=2 /user/login 

```

例如，您可以创建一个仪表板，查看来自单个 IP 地址的失败尝试，如果超过某个阈值，这可能表明恶意尝试暴力破解系统访问。通常可以在此类事件上设置警报，让您能够主动识别威胁并阻止访问。

# 摘要

在本章中，我们学习了您的服务可能面临的来自入侵者的攻击。我们希望有一个关于加密工作原理以及如何利用 Go 的标准包来实现这些功能以保护我们的服务的介绍。您几乎无法完全保护自己免受决心攻击者的攻击；然而，使用本章中描述的简单技术应该形成您标准的工作实践。实施许多这些技术不会显著减慢您的开发周期；然而，这将为您提供一个保持安全的优势。
