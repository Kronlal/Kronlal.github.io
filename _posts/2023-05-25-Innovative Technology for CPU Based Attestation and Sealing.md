---
layout: post
title: Innovative Technology for CPU Based Attestation and Sealing论文翻译
subtitle: 对于Intel SGX论文的翻译
date: 2023-05-25 19:50:00 +0800
categories: 机密计算
author: 月梦
cover: 'https://s1.ax1x.com/2023/05/25/p9Hy7p6.jpg'
cover_author: 'Intel'
cover_author_link: 'https://www.intel.com/content/www/us/en/homepage.html'
tags:
- SGX  
- 机密计算  
- 内存安全
---

**Innovative Technology for CPU Based Attestation and Sealing**是Intel SGX技术的官方论文，本文将翻译这篇文章。  
SGX技术提供了enclave环境，当今比较火的机密计算技术一般就是基于SGX技术来实现，当然也有其他的可以提供enclave环境的技术，例如TrustZone等，但是SGX应用更多，且相比之下更安全些。  
![sgx论文封面](https://s1.ax1x.com/2023/05/25/p9Hy7p6.jpg)

本文中形似**（注：···）**的批注是我的批注，而非原文  

# Innovative Technology for CPU Based Attestation and Sealing(基于CPU的认证与密封新技术)

## Abstract
Intel正在开发Intel®Software Guard Extensions(Intel®SGX)技术，这是Intel®架构的扩展，**用于生成受保护的软件容器。容器被称为飞地。在飞地内部，软件的代码、数据和堆栈受到硬件强制访问控制策略的保护，这些策略可以防止对飞地内容的攻击。**在一个软件和服务通过互联网部署的时代**（注：也就是云计算时代）**，关键是能够通过有线或空中**安全地远程提供飞地**，有把握地知道机密受到保护，并能够将机密保存在非易失性存储器中以备将来使用。  
本文描述了允许向飞地提供机密**（注：应该指用户想要保护的数据）**的技术组件。这些组件包括**用于对在飞地内运行的软件生成一个基于硬件的证明的方法，以及用于飞地软件密封机密并将其导出到飞地之外（例如将其存储在非易失性存储器中）的方法，以便只有相同的飞地软件能够将其解封回其原始形式。**   

## 1. INTRODUCTION
在一个软件和服务通过互联网部署的时代，Intel®Software Guard Extensions(Intel®SGX)和Extensions to Intel®Architecture）使服务提供商能够通过有线或无线提供具有敏感内容的应用程序，并确信其机密得到了适当保护。**为了做到这一点，服务提供者必须能够确切地知道远程平台上运行的是什么软件，以及它在哪个环境中执行。**   

## 1.1 Software Lifecycle
启用Intel®SGX的软件不附带敏感数据**（注：指用户的软件，但是还未开始使用，软件可以开辟enclave）**。在这个软件被安装后，它向服务提供商通信，以便将数据远程提供给飞地。然后软件对数据进行加密并存储以备将来使用。 图1阐明了软件完成此过程所需的步骤。
![sgx002](https://s1.ax1x.com/2023/05/25/p9H6fv8.png)
1. **飞地启动**--**不受信任的应用程序启动飞地环境以保护服务提供商的软件。在构建飞地时，会记录一个安全日志，反映飞地的内容及其加载方式。** 这个安全日志是飞地的“度量”(即**Measurement**)。   
2. **Attestation**--飞地联系服务提供者，以便将其敏感数据提供给飞地。 该平台生成一个安全断言（**secure assertion**），用于标识硬件环境和飞地。**（注：安全断言被插入到程序中，用于检测安全问题）**  
3. **Provisioning**--服务提供者评估飞地的可信度。它使用**Attestation**来建立安全通信并向飞地提供敏感数据。使用安全通道，服务提供者将数据发送到飞地。   
4. **Sealing/Unsealing** --飞地使用一个持久的基于硬件的加密密钥来安全地加密和存储其敏感数据，以确保只有在可信环境恢复时才能检索数据。**（注：可信环境未就绪，不解密。）**  
5. **Software Upgrade** -- 服务提供商可能需要飞地软件更新。为了简化数据从旧软件版本到新版本的迁移，软件可以从旧版本请求密封(seal)密钥来解封数据，并请求新版本的密封，这样密封的数据就不会被以前版本的软件使用。   

最后，当平台所有者计划转移平台所有权时，应使其所有权期间可用的秘密不可访问。 Intel®SGX包含一个用户拥有的特殊持久值，当更改该值时，将更改软件可用的所有密钥。   

## 1.2 安全模型
希望向远程平台提供秘密的服务提供者必须事先知道远程平台的保护策略满足要部署的秘密的保护要求。 根据Intel®SGX的安全模型[1]，负责保护机密的可信计算基础(TCB)包括处理器的固件和硬件，仅包括飞地内的软件。   
一个Enclave Writer可以使用Intel®SGX的专用sealing和attestation组件，这些组件支持以下安全模型断言，以向服务提供商证明机密将根据预期的安全级别受到保护：   
- Intel®SGX为Enclave实例提供了向平台请求Enclave’ identity的安全断言的方法。   
**（注：Enclave’ identity就是Enclace Measurement，Enclave实例可以通过请求安全断言来验证自己的Enclave Measurement是否正确，并确保自己所在的环境是安全和可信任的。）**  
Intel®SGX还允许飞地将飞地瞬时的数据绑定到断言。   
- Intel®SGX为Enclave实例提供了验证来自同一平台上其他Enclave实例的断言的方法。   
- Intel®SGX为远程实体提供了验证来自Enclave实例的断言的方法。   
- Intel®SGX允许Enclave实例获取绑定到平台和Enclave的密钥。  
Intel®SGX阻止软件访问其他飞地标识的密钥    

## 1.3 Intel® SGX Instructions指令
Intel®SGX体系结构[1]提供了硬件指令EREPORT和EGETKEY，以支持认证attestation和密封sealing。接受SGX安全模式的秘密拥有者可以依靠这些指示向负责秘密的TCB报告。   
为了创建飞地环境，不受信任的软件使用Intel®SGX指令。这些指令还计算已启动环境的加密Measurement。本文第2节进一步描述了这些过程。   
为了启用attestation和sealing，硬件提供了两个附加指令EREPORT和EGETKEY。 EREPORT指令提供了一个证据结构，该结构以加密的方式绑定到硬件上，以供认证验证者使用。 EGETKEY为Enclave软件提供了访问认证和密封过程中使用的“Report”和“Seal”密钥的权限。 第3节讨论了如何使用这些指令来提供飞地的证明，第4节讨论了如何保护传递给飞地的秘密。  
在第5节中，我们简要回顾了在平台中建立远程信任的相关工作。   

## 2 MEASUREMENT
Intel®SGX架构负责建立用于认证和密封的identities。对于每个飞地，它提供两个measurement寄存器，MRENCLAVE和MRSIGNER；MRENCLAVE提供了所构造的飞地代码和数据的identity，MRSIGNER提供了飞地权限的identity。这些值在构建飞地时被记录，并在飞地执行开始前最终确定。只有TCB有权写入这些寄存器，以确保在认证和密封时能够准确反映identities。    

## 2.1 MRENCLAVE - Enclave Identity
“Enclave Identity”是MRENCLAVE的值，它是内部日志的SHA-256[2]摘要，记录了在构建Enclave时所做的所有活动[1]。 日志由以下信息组成：   
- 页的内容（代码、数据、堆栈、堆）。   
- 飞地中页的相对位置。   
- 与页关联的任何安全标志。   

一旦通过EINIT指令完成了enclave初始化，就不再对MRENCLAVE进行更新。 最后MRENCLAVE的值是一个SHA-256摘要，它以加密方式标识放置在飞地中的代码、数据和堆栈、飞地页的放置顺序和位置，以及每个页的安全属性。对这些变量的任何更改都将导致MRENCLAVE中的值不同。   
**（注：MRENCLAVE中存的应该就是Enclave Measurement）**  

## 2.2 MRSIGNER - Sealing Identity
该飞地有第二个用于数据保护的identity，称为“Sealing Identity”。Sealing Identity包括“Sealing Authority”、产品ID和版本号。 Sealing Authority是在分发前对飞地进行签名的实体，通常是飞地构建者。飞地构建者向硬件提供一个RSA签名的飞地证书（SIGSTRUCT），该证书包含Enclave Identity，MRENCLAVE和密封机构的公钥。 硬件使用证书中包含的公钥检查证书上的签名，然后将测量的MRENCLAVE值与签名版本进行比较。如果这些检查通过，则在MRSIGNER寄存器中存储封签机构的公钥的哈希值。需要注意的是，如果多个飞地由同一封签机构签名，它们都将具有相同的MRSIGNER值。如第4节所示，Sealing Identity的值可用于封存数据，在某种程度上，来自同一封存机构的飞地（例如，同一飞地的不同版本）可以共享和迁移其封存数据。   

## 3 ATTESTATION
证明是证明一个软件已经在平台上被正确实例化的过程。 在Intel®SGX中，这是一种机制，通过这种机制，另一方可以获得信任，即正确的软件安全地运行在启用的平台上的飞地内。 为此，Intel®SGX体系结构生成一个attestation assertion（认证断言）（如图2所示），它传达以下信息：   
- 软件环境的identities被证明  
- 任何不可测量状态的细节（例如，软件环境可能运行的模式）   
- 软件环境希望与自身相关联的数据   
- 与平台TCB绑定的密码来制作这个assertion  

![sgx003](https://s1.ax1x.com/2023/05/25/p9HcLod.png)

Intel®SGX体系结构提供了一种机制，用于在运行在同一平台上的两个飞地（本地认证）之间创建authenticated assertion，以及另一种机制，用于扩展本地认证，以向平台外的第三方提供断言（远程认证）。**（注：SGX提供两种机制，本地证明和远程证明）**  
最后，为了在系统中获得最大的可信度，认证密钥应该只被绑定到一个特定的平台TCB上。如果平台TCB发生变化，例如通过微码更新，应替换平台认证密钥，以正确地代表TCB的可信度。  

## 3.1 Intra-Platform Enclave Attestation
应用程序开发人员可能希望多个飞地相互协作来来执行一些更复杂的功能。为了实现这个，他们需要一种机制来让一个enclave证明另一个。为了这个目的，SGX提供了EREPORT指令。  
当这个指令被一个飞地调用时，EREPORT产生一个签名的结构，称为REPORT。REPORT结构包括飞地的两个identities，与飞地相关联的属性（属性标识模式和在ECREATE期间建立的其他属性），硬件TCB的可信度，以及飞地开发者希望传递给目标飞地的额外信息，以及一个消息认证码（MAC）标记。目标飞地将验证MAC，允许它确定创建REPORT的飞地是否在同一平台上运行。  
MAC被一个称作“Report Key”的密钥产生。如表1所示，Report Key只被目标飞地和EREPORT指令知道。目标飞地可以得到他自己的Report Key通过EGETKEY指令，EGETKEY为飞地提供一系列密钥，包括Report Key，可用于对称加密和身份验证。目标飞地使用Report Key通过REPORT结构重新计算MAC，核实这个REPORT是由证明（reporting）飞地产生的。Intel®SGX架构使用AES128-CMAC [3]作为MAC算法。  
![sgx004](https://s1.ax1x.com/2023/05/25/p9HczSP.png)

每个REPORT结构也包括一个256bit的用户数据字段。这个字段将enclave内的数据绑定到enclave的identity（如REPORT中所述）。该字段可用于使用辅助数据扩展REPORT，通过将其填充为辅助数据的哈希摘要，然后该摘要与REPORT一起提供。使用用户数据字段使飞地能够构建更高层次的协议，在自身和另一个实体之间形成安全通道。  
例如，通过交换在飞地内使用双方同意的参数随机生成的公共Diffie-Hellman密钥的报告，飞地可以生成一个经过验证的共享秘密，并使用它来保护它们之间的进一步通信。Intel®体系结构支持通过可供飞地软件使用的RDRAND指令[4]生成真正的随机值。  
下图显示了一个示例流程，说明两个飞地在同一平台上如何相互验证，并验证对方在同一平台上的一个飞地内运行，因此符合Intel®SGX的安全模型。  
![sgx005](https://s1.ax1x.com/2023/05/25/p9Hgpy8.png)

1. 在飞地A和B之间建立通信路径后，飞地A获得飞地B的MRENCLAVE值。请注意，此步骤中的通信路径不必是安全的。  
2. 飞地A与飞地B的MRENCLAVE一起调用EREPORT指令来为飞地B创建一个签名的REPORT。飞地A通过不可信的通信路径将其报告传输到飞地B。  
3. 收到来自A飞地的报告后：  
飞地B调用EGETKEY来检索它的Report Key，重新在REPORT结构上计算MAC，并将结果与REPORT携带的MAC进行比较。MAC值的匹配肯定了A确实是一个与飞地B运行在同一平台上的飞地，因此A也运行在一个遵循Intel®SGX的安全模式的环境中。  
**（注：飞地B应该是用飞地A的Report Key来重新计算MAC，实际上这块说调用EGETKEY我不太明白，因为EGETKEY按理只能拿到自己的Report Key）**  
一旦验证了TCB的固件和硬件组件，飞地B就可以检查飞地A的报告，以验证TCB的软件组件：  
- MRENCLAVE反映了在飞地内运行的软件映像的内容。  
- 签名先生反映了密封人的identity  
飞地B可以回复一个为飞地A创建的REPORT，通过使用刚刚收到的REPORT中的MRENCLAVE。  
飞地B向飞地A发送其REPORT，然后，飞地A可以以类似于飞地B的方式验证该报告，确认飞地B与飞地A存在于同一平台上。  

## 3.2 Inter-Platform Enclave Attestation
用于平台内飞地认证的验证机制使用对称密钥系统，其中只有验证REPORT结构的飞地和创建REPORT的EREPORT指令才能访问验证密钥。创建一个可以在平台之外进行验证的认证需要使用非对称密码学。英特尔®SGX启用了一个特殊的飞地，称为Quoting Enclave，专门用于远程证明。Quoting Enclave在平台上验证来自其他enclave的REPORTs，使用上面描述的平台内飞地验证方法，然后用使用特定于设备（私有）非对称密钥创建的签名替换这些报告上的MAC。这个过程的输出被称为QUOTE。  

### 3.2.1 Intel® Enhanced Privacy ID (EPID)
当在平台的整个生命周期中使用少量密钥时，使用标准非对称签名方案的认证引起了隐私问题。为了克服这个问题，英特尔对TPM [5] &[6]使用的直接匿名认证方案引入了一个扩展，称为Intel®增强隐私ID（EPID）[7]，被Quoting Enclave用来签名飞地认证。  
EPID是一种组签名方案，它允许平台对对象进行签名，而不需要唯一地标识平台或链接不同的签名。相反，每个签名者都属于一个“组”，验证者使用该组的公钥来验证签名。EPID支持两种签名模式。在EPID的完全匿名模式下，验证者无法将给定的签名与组中的特定成员关联起来。在伪名模式下，EPID验证者能够确定它之前是否已经验证了该平台。  

### 3.2.2 The Quoting Enclave
Quoting Enclave创建了用于签名平台认证的EPID密钥，然后由EPID后端基础设施进行认证。EPID密钥不仅表示平台，还表示底层硬件的可信度。  
当飞地系统运行时，只有Quoting Enclave可以访问EPID密钥，并且EPID密钥被绑定到处理器的固件版本。因此，可以看成一个QUOTE是由处理器本身发出的。  

### 3.2.3 Example Remote Attestation Process
图4显示了一个示例，说明在用户平台上具有安全处理元素的应用程序如何向具有挑战性的服务提供者提供认证，以便从提供者接收一些增值服务。请注意，许多用法很少使用此过程（例如在注册时）为飞地提供一个通信密钥，然后直接在后续连接中使用。  
![sgx006](https://s1.ax1x.com/2023/05/25/p9Hg8YR.png)
1. 最初，应用程序需要来自平台外部的服务，并与服务提供系统建立通信。服务提供商向应用程序发出challenge，以证明它确实在一个或多个飞地内运行必要的组件。challenge本身包含一个nonce（一次性数字），用于验证通信的活跃性。**（注：challenge应该就是nonce，一个一次性的数字）**    
2. 该应用程序提供了Quoting Enclave的Enclave Identity，并将其与提供者的challenge一起传递到该应用程序的飞地。  
3. 这个飞地产生了一个manifest，包括对challenge的响应和一个临时生成的公钥，由challenger用来将秘密传回这个飞地。然后，它生成manifest的hash摘要，并将其作为EREPORT指令的用户数据包含在内，该指令将生成一个将manifest绑定到飞地的报告，如第3.2节所述。然后，该飞地会将REPORT发送到应用程序。  
4. 应用程序将REPORT转发给Quoting Enclave进行签名。  
5. Quoting Enclave使用EGETKEY检索它的Report Key并且核实REPORT。Quoting Enclave创建QUOTE结构并且使用EPID密钥签名。Quoting Enclave返回QUOTE结构给应用程序。  
6. 应用程序发送QUOTE结构和一些相关的支持数据manifest给服务challenger。  
7. challenger使用EPID公钥证书和撤销信息或者认证验证服务来验证QUOTE上的签名。他之后使用USERDATA来核实manifest的完整性，并且检查manifest是否有对它在步骤1中发送的challenge的响应。  

## 4 Sealing  
当飞地被实例化时，当其维护在飞地边界内时，硬件为其数据提供保护（机密性和完整性）。然而，当飞地进程退出时，飞地将被摧毁，在飞地内得到安全的任何数据都将丢失。如果这些数据意味着以后会被重用，那么该飞地必须做出特殊安排，在该飞地之外存储这些数据。  
上面的表1显示，EGETKEY提供了对持久性Sealing Keys的访问，飞地软件可以使用这些密钥来加密和完整性保护数据。Intel®SGX对该飞地使用的加密方案没有任何限制。当与平台提供的其他服务结合时，如单调计数器，对数据也有可能进行Replay保护。  

## 4.1 Intel® SGX supported Sealing Policies
当调用EGETKEY时，飞地选择条件或策略，飞地可以访问此sealing key。这些政策有助于控制敏感数据对该飞地未来版本的可访问性。  
Intel®SGX支持Seal Keys的两种策略：  
- Sealing to the Enclave Identity  
- Sealing to the Sealing Identity  

Sealing to the Enclave Identity产生一个密钥，可用于这个确切的飞地的任何实例.**（注：如果这个seal key是基于Enclave Identity生成的，则该密文可以被部署在相同enclave Identity的任何实例所使用。在SGX中，将数据密封到enclave的身份标识上生成的密钥可以在相同身份标识的不同enclave实例中共享。）**这并不允许未来的软件访问这个飞地的秘密。Sealing to the Sealing Identity产生一个密钥，可被用于一些别的由相同Sealing Authority签名的enclaves。这可用于允许较新的飞地访问以前版本存储的数据。  
只有飞地的后续实例化，执行具有相同策略规范的EGETKEY，才能检索Sealing Key并解密以前实例化使用该密钥密封的数据。  

### 4.1.1 Sealing to the Enclave Identity
当Sealing to the Enclave Identity，EGETKEY操作会基于enclave的MRENCLAVE的值生成密钥。任何影响飞地measurement的变化都将产生不同的密钥。这导致每个飞地都有不同的密钥，提供了飞地之间的完全隔离。使用此策略的一个副产品是，同一飞地的不同版本也将具有不同的密封密钥，从而阻止脱机数据迁移。
此策略对于在发现漏洞后不应该使用旧数据的用法非常有用。例如，如果数据是身份验证凭据，则服务提供者可以撤销这些凭据并提供新的凭据。访问旧的凭证可能是有害的。  

### 4.1.2 Sealing to the Sealing Identity
当Sealing to the Sealing Identity，EGETKEY操作会基于enclave的MRSIGNER的值和enclave的版本生成密钥。MRSIGNER反映了签署该飞地证书的Sealing Authority的key/identity。  
这种方式相比上一种的优点是，它允许将封闭的数据在飞地版本之间离线迁移。Sealing Authority可以签署多个飞地，并使他们可以检索相同的seal key。这些飞地可以透明地访问被其他飞地封闭的数据。  
当sealing to a Sealing Authority，不应该允许较旧的软件访问由较新的软件创建的数据。当发布新软件的原因是为了修复安全问题时，这是正确的。为了方便这一点，Sealing Authority可以选择规定一个安全版本号（SVN）作为密封件标识的一部分。EGETKEY允许飞地指定在生成Seal Key时使用哪个SVN。它将只允许飞地为其Sealing Identity或以前的Identity指定SVNs。当飞地密封数据时，它可以选择设置允许访问该Sealing Key的飞地的最小SVN值。这可以保护未来的秘密不受易受攻击的旧软件的访问，但仍然支持无缝升级过渡，即以前的所有秘密都在升级后可用。  
SVN与产品版本号不同。一个产品可能有多个版本，具有不同的功能，但具有相同的SVN。飞地Writer有责任与他们的客户沟通（如果必要的话），哪些产品版本具有相同的安全等价性。  

## 4.2 Removing Secrets from a Platform
该体系结构提出了一种被称为OwnerEpoch的机制，它允许平台所有者通过更改单个值来更改系统中的所有键。由于在通过EGETKEY指令请求密钥时会自动包含OwnerEpoch，使用特定的OwnerEpoch值密封的数据对象，只有在将OwnerEpoch设置为相同的值时才能打开。  
该机制的主要目的是允许平台所有者在将平台转移给其他人（永久或暂时）之前，以一个简单且可恢复的步骤拒绝对平台上所有密封秘密的访问。在转移平台之前，平台所有者可以通过使用OEM提供的hooks，将OwnerEpoch更改为一个不同的值。如果临时转移（如平台维护），平台返回后，平台所有者可以将OwnerEpoch恢复到原始值，并恢复对密封秘密的访问。  

## 5. RELATED WORK
略···  

## 6. CONCLUSIONS
在本文中，提出了一种新的硬件辅助机制，以Intel®架构的新ISA扩展形式，允许在安全环境（称为飞地）中执行的应用软件进行安全attestation和sealing。  
ISA扩展为飞地软件提供了手段，以向另一方证明它已在平台上正确实例化，并扩展为正确的软件，并在一个已启用的平台上的飞地内安全运行。这种认证不需要验证者理解飞地正在执行的平台软件上下文，并且仅限于信任飞地软件。  
Intel®SGX体系结构还提供了一种机制来获得持久的唯一密钥，软件可以使用该密钥来密封秘密并稍后打开秘密。密封机制还提供了在一个飞地的软件升级时无缝转换秘密的能力。  
认证和密封机制以一种可扩展的方式定义，支持多个飞地同时运行，每个飞地处理自己的秘密，并与远程各方进行安全通信，并向他们证明他们符合他们的安全策略。  

