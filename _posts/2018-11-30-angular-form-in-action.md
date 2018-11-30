---
layout: post
keywords: blog
description: blog
title: "Angular 表单在 K8S 资源对象上的实践"
categories: [frontend]
tags: [Angular, Reactive Form]
icon: code
---

# 简介
Kubernetes 集群的使用者日常工作中经常需要与 Deployment 等 Kubernetes 对象接触。熟悉 Kubernetes 的朋友都会知道，Kubernetes 的对象结构虽然重视可移植性，不同的对象有着相似的设计理念，但就算是最熟练的系统运维或者开发工程师也不一定能将 Kubernetes 的对象玩转。
为了使用户更方便的操作 Kubernetes 对象，灵雀云的 Kubernetes 发行版前端界面提供了对用户友好的 UI 表单解决方案。用户可以通过 UI 表单更容易的编辑 Kubernetes 对象，同时还提供了 YAML 格式与 UI 表单实时互转的功能。

读者可以先从[这个Deployment表单demo](https://pengx17.github.io/k8s-form-in-action/demo/)里面感受一下实时互转的效果（注：这个 demo 是为这篇文章单独开发，与灵雀云产品无关）：

![图片](https://uploader.shimo.im/f/TNGlAwy6gyISzrU3.png!thumbnail)
# 难点
YAML 是 Kubernetes 对象最常见的展现和修改形式。对于前端来说，假如我们需要同时支持表单和 YAML 的方式编辑 Kubernetes 资源，那么最终我们都得落回到编辑  YAML 上。
从数据表现格式上来看，表单的模型数据是 JSON 对象，而 YAML 为字符串。YAML 与表单数据的互转，类似于序列化与反序列化的过程，我们需要在转化的边界借助如[js-yaml](https://github.com/nodeca/js-yaml)之类的库进行转换。
实际上数据形式的问题比较好解决，难点在于 YAML 所代表的 Kubernetes 资源对象模型与表单数据进行转换的问题。实践过程中，表单数据与 K8S 对象互转有不少问题需要攻克。比如：
* UI 表单状态与 K8S 对象状态不一定是完全一致的，比方说：
  * 我们不会或者不需要通过 UI 编辑 K8S 的所有字段，非 UI 可编辑字段在互转的时候可以得到正确保留
  * UI 展现的形式并非与 YAML 严格对应，比如 metadata 的 label 字段在 K8S 对象 Schema 中表现为一个 StringMap，但在 UI 上表现为数组
  * 针对实际业务场景，有时候会为 YAML 进行隐式的修改或者填充
* 表单字段嵌套层次深，同时表单字段之间可能有关联性
* 局部表单复用。 比如 Workload/Pod 相关的资源都可以编辑 PodSpec 或者 Container
* 实时同步表单与 YAML 内容，保证两种数据表现形式在任何时间点都是一致的
  * 考虑到正确性与可维护性，这个功能点驱使我们的表单实现方案必须往单项数据流方向靠拢。
# 实现方案推导
## 模板驱动表单
每个 Angular 表单的开发者应该都接触过**模板驱动**表单与**响应式**表单这两个不同的Angular表单实现思路。有些开发者可能会把响应式表单与动态表单混淆，实际上这两个概念没有什么联系。不熟悉的同学可以看看 [Angular 官网这篇关于表单的介绍](https://angular.cn/guide/forms-overview)，其给出如下两者的区别：
>一般来说：
>
>响应式表单更健壮：它们的可扩展性、可复用性和可测试性更强。 如果表单是应用中的关键部分，或者你已经准备使用响应式编程模式来构建应用，请使用响应式表单。
>
>模板驱动表单在往应用中添加简单的表单时非常有用，比如邮件列表的登记表单。它们很容易添加到应用中，但是不像响应式表单那么容易扩展。如果你有非常基本的表单需求和简单到能用模板管理的逻辑，请使用模板驱动表单。

用模板驱动表单写前端表单确实很容易：给定任意一个数据对象，将需要的字段与模板的表单控件通过 [(ngModel)] 指令进行数据绑定；根据实际需要，再绑定一下诸如 required 的表单验证指令就完事了。[鹅妹子嘤](https://www.google.com/search?q=tim+cook+amazing)！

不过一旦这么做，用户就将数据的“权威”就交给了模板，脱离了数据的实际控制权，也就只能被动的接受来自于模板的数据更新、表单状态与生命周期、数据验证等事件。对于复杂表单的业务逻辑，你很难通过这种模式扩展到大规模而复杂的表单数据逻辑处理之中。
## 响应式表单与受控组件
使用 Angular 响应式表单对于初学者来说有些啰嗦和麻烦：为了维护表单的状态，我们需要显式地创建一套完整的表单控制器对象层级结构，并将此对象通过 FormGroup / FormControl 之类的指令绑定到模板上的表单控件上。初看 Angular 的响应式表单的思想，似乎有点违背如今 MV* 的设计模式，因为它把一些本来可以通过框架隐式管理的工作暴露给了开发者自己，额外的增加了不少工作量。

熟悉 React 的表单控件实现的人应该了解React 的[受控组件](https://reactjs.org/docs/forms.html)和[非受控组件](https://reactjs.org/docs/uncontrolled-components.html)概念。通过受控组件，用户可以通过单项数据流的思路，掌握表单控件数据的实际控制权。不过对于实际的完整表单应用场景，用户还需要处理表单的提交状态、表单验证逻辑等信息。Angular 框架内置了一套相对成熟稳定的响应式表单解决方案，帮助开发者更可控的管理表单的状态的同时进行表单相关业务的开发。
## Angular 表单控件的根基：ControlValueAccessor
Angular 的表单控件的魔法与 React 的受控组件的思路十分类似，是典型的单项数据流的处理模式。另外，不管是模板驱动表单还是响应式表单，都是围绕着 ControlValueAccessor 接口设计实现的。假如一个组件提供了 [NG_VALUE_ACCESSOR](https://angular.io/api/forms/NG_VALUE_ACCESSOR) 令牌注入到模板的 DI 上下文，并实现了 [ControlValueAccessor](https://angular.io/api/forms/ControlValueAccessor) 接口，那么这个组件就可以绑定任意 Angular 的表单指令。

ControlValueAccessor 最关键的有两点: registerOnChange 和 writeValue，这两个函数分别对应了单项数据流从表单内到外和从表单外到内两个方向的数据变化。
* registerOnChange：初始化表单的过程中 Angular 会通过此接口，请求目标组件注册一个 onChange 回调。用户可以通过这回调，从内到外，将表单控件的数据更新事件发射到控件外部，更新表单控件对象的数据。
* writeValue：Angular 的表单控件对象更新时会主动调用此函数。可以看成外部的数据状态流入表单内部。用户可以自定义这次数据更新的作用，绑定到组件内部模板的表单控件上。
## 适配器模式
聪明的朋友一定会注意到，ControlValueAccessor 接口并没有要求 onChange 与 writeValue 调用的时候表单数据格式与输入一致。因此我们可以在一个业务表单控件组件内实现局部的资源对象与UI表单数据转换的逻辑。比方说上面提到的，我们可以通过它实现一个键值对表单控件。

![图片](https://uploader.shimo.im/f/2hlXZVDHKlU9vcpN.png!thumbnail)

它对外暴露为正常的键值对控件，值类型为 { [key: string]: string }。 数据由外到内时，可以通过 writeValue 将键值对通过 Object.entries 改变为 [string, string] 的数组，最后将绑定到表单内部的 FormArray 控件上；同时将内部状态改变时，在调用 onChange 之前将 [string, string][] 转化为外部的 { [key: string]: string } 对象类型。

假如我们以表单控件为界限，这个界限的两侧分别为 Host 上绑定表单控件 NgControl 与表单控件内部的响应式表单。借助 onChange 和 writeValue，我们可以围绕着 ControlValueAccessor 实现一个局部的[适配器模式](https://zh.wikipedia.org/wiki/%E9%80%82%E9%85%8D%E5%99%A8%E6%A8%A1%E5%BC%8F)。
### 通过这个思路，我们可以继续引申：
由于有了基于适配器模式的数据转化思路， 对于每一个表单控件，我们可以通过 onChange 和 writeValue 这两个接口进行数据与表单的模型变化，实现UI的数据模型和实际对外暴露的数据模型的不一致需求。

针对于一个复杂的资源对象表单，我们可以把问题拆解为多子表单控件组件，每一个子表单控件组件都去实现适配器模式。同时，每个子表单控件组件依然可以用同样方式进行拆解组合。这样，通过将资源对象不断往下递归拆解，借助表单的组合和嵌套，我们可以最终实现一个复杂的表单树。

内嵌表单的实现隔离了复杂表单的实现逻辑。每一个子表单控件虽然对外暴露是一个表单控件数据，但其内部是一个完整的表单。父级（host）组件可以完全不了解子表单组件控件内部处理逻辑，比如表单的错误处理、数据转换等。同时由于K8S的设计，许多子表单是可以在不同的资源里复用的，减轻了我们的开发成本。

表单控件本身在提供 K8S 数据的同时，也可以表现为一个独立的 K8S 对象资源。我们可以把局部相关的业务逻辑完整的封装在此表单控件组件内部，做到神行合一。这点很重要，通过这一点，我们可以更容易的划分出 K8S 资源的问题范围，更快做出代码结构的判断，减少开发的脑力负担和维护成本。坊间也有其他开发者倾向于将业务处理逻辑独立出来，不放到组件内部，这样组件就可以只负责薄薄一层视图渲染逻辑。我认为不是不可行，不过在复杂表单组件嵌套和复用角度，可能本文采用的方式更容易维护。

由于上述实现思路过程有比较规范的思路，我们可以设计出来一个标准的开发 Kubernetes 资源对象表单的实现范式。这个范式可以大大降低开发人员对于开发、维护复杂表单实现的思维负担。以上的过程借助递归的思路，把一个问题拆解的简单又有效：
* 不管任何模式的复杂表单，可以立刻开始着手开发
* 强调开发体验的共识、抽象与封装
* 避免开发出新的错误类型
# Kubernetes对象的响应式表单开发范式
## 中心思想
我总结了这个开发范式里几个关键点：
* 神形合一：组件即是资源，也是表单控件
* 分形：局部子对象表单组件处理与整体对象表单组件处理保持一致
* 递归: 由于分形的特性，我们可以用递归的方式自上而下，用统一的方式处理表单组件
* 问题隔离：一次只处理一个问题
* 响应式表单：严格执行单向数据流，同步处理，以达到实时同步的目的
## 流程
为任意一个 Kubernetes 对象开发表单的过程可以总结如下：
1. 学习目标 Kubernetes 对象的基本功能, 对它的 YAML Schema 有基本概念。
  1. 由于我们前端人员对于 YAML 字段的高透明度和充分的修改灵活度, 我们需要了解相关 k8s 对象的业务/特性.
2. 书写目标 API 对象 TypeScript 的类型 ( interface / type 等)。
3. 拆解 k8s 对象类型成一系列子对象，为每个可复用的子对象封装为单独的表单组件。
  1. 比如 PodSpec, Container, Env 等
4. 为拆解出来的每个子对象表单组件实现表单到对象的互转。
5. 组合子对象表单，最终组合成完整的 K8S 对象表单

稍后我们会以部署表单为例，详细说明流程细节。
## 用例分析: Deployment 表单
### 熟悉 Deployment 对象的结构
首先参考官网对于 [Deployment 的 API 文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#deployment-v1-apps)，输出一套 TypeScript 的接口，方便后续参照：
```
interface Deployment {
  apiVersion: string;
  kind: string;
  metadata?: ObjectMeta;
  spec?: DeploymentSpec;
}
export interface ObjectMeta {
  name?: string;
  namespace?: string;
}
export interface DeploymentSpec {
  replicas?: number;
  template?: PodTemplateSpec;
  // ...
}
export interface PodTemplateSpec {
  metadata?: ObjectMeta;
  spec?: PodSpec;
}
export interface PodSpec {
  containers: Container[];
  // ...
}
export interface Container {
  name?: string;
  image?: string;
}
```
### 部署表单拓扑
对于部署表单，我们拆分为3个主要表单：
* [DeploymentForm, PodSpecForm, ContainerForm]
```
+[DeploymentForm]----------------------+
|                                      |
| metadata.name: input                 |
| metadata.namespace: select           |
| spec.replicas: input                 |
|                                      |
| +spec.template.spec: [PodSpecForm]-+ |
| |                                  | |
| | containers[0]:+[ContainerForm]-+ | |
| |               | name: input    | | |
| |               | image: input   | | |
| |               +----------------+ | |
| | containers[1]:+[ContainerForm]-+ | |
| |               | name: input    | | |
| |               | image: input   | | |
| |               +----------------+ | |
| +----------------------------------+ |
+--------------------------------------+

art: http://asciiflow.com/
```
### K8S 资源对象表单控件组件 - 模板
最外层组件，对象的使用者可以依然使用模板驱动表单，将视图双向绑定到数据上：

```
<deployment-form [(ngModel)]="deployment"></deployment-form>
```
内部模板书写上比较容易：由普通表单控件 (如select, input等) 和其他子对象表单控件（如pod-spec-form）组成为一个单独的表单。

部署模板使用响应式表单：

```
<form formGroup="form">
  <ng-container formGroupName="metadata">
    Name:     <input formControlName="name">
    Namespace <select formControlName="namespace"></select>
  </ng-container>
  <ng-container formGroupName="spec">
    <ng-container formGroupName="template">
      <pod-spec-form formControlName="spec"></pod-spec-form>
    </ng-container>
  </ng-container>
</form>
```
### K8S 资源对象表单控件组件 - 控制器
资源对象组件控制器（也就是 TS 部分）的职责如下：
* 对外暴露为一个单独的表单控件
  * Host 模板可以绑定表单相关指令到对象表单控件
* 对内表现为一个完整的表单组件
  * 根据视图创建出一个表单控件树
  * 协同各个表单控件，响应数据变化
* 使用单向数据流处理流入表单的数据
* 使用单向数据流处理流出表单的数据

组件初始化时，需要生成一个响应式表单控件树。根据实战，我总结如下经验：
* 有且只有一个根部 form 控件对象， 根据情况可能是 FormGroup 、FormArray、FormControl。但最终都要绑定到模板的 FormGroupDirective 指令上。
* FormGroup 对象结构一般与当前对象 schema 结构相似，这样可以
  * 通过 form.patchValue 来设置表单数据
  * 在控制器或者模板里更容易的与原始数据进行对照
* 在模板内可以组合使用 formGroupName, formControlName 等指令绑定于响应表单控件

比如对于部署表单，我们需要生成这样结构的表单控件：
```
const metadataForm = this.fb.group({
  name: ['', [Validators.required]],
  namespace: ['', [Validators.required]],
  labels: [{}],
  annotations: [{}]
});
const specForm = this.fb.group({
  selector: this.fb.group({ matchLabels: [{}] }),
  template: this.fb.group({ spec: [{}] })
});
const deployForm = this.fb.group({
  metadata: metadataForm,
  spec: specForm,
});
```
控件需要对外暴露为一个普通的表单控件，同时将内部表单的错误向上传递到 Host 上的 NgControl 指令上。最关键的就是要实现 ControlValueAccessor 接口：

* writeValue: 由外部写入内部时，需要将资源对象适配为表单可用的模型结构。
  * 大部分时候表单的 FormModel 与资源对象的 schema 一致。
  * 假如业务需要，比如 k8s 的 metadata.labels 字段是 { [key: string]: string } 键值映射对象，但在视图中他的表单模型是键值对数组 [string, string][]，可以在这个阶段进行数据适配。
* onChange: 由内部写回外部时，需要将表单模型适配为资源对象模型，同时将 UI 不可见的字段写回资源对象模型中。
* 同时由于实现的原因，需要监听上层模板的 Form 指令，以得到提交嵌套模板的功能
### setFormByResource 和 setResourceByForm
刚才提到，为表单设置资源对象数据时可以直接通过调用 form.patchValue(formModel) ，使得一个结构化的表单被能快速的填充。 有一个问题是，Angular 限制调用 patchValue 方法时 formModel 的 schema 必须是 form 结构的一个子集， 但通常来讲 form 的控制器结构有时候不需要覆盖完整的 schema (比如 status 字段等)。

我设计了 setFormByResource 函数解决这个问题，方法是通过遍历表单层级里面所有的控制器，以控制器所在的路径作为线索查找资源对象上的相应的值， 然后设置到表单控制器上；同时在 form 的某个控制器是 FormArray 的情况下，根据数据来源的大小进行伸缩。

而 setResourceByForm 函数与 setFormByResource 作用相反。 在表单数据写回资源对象时，利用它遍历表单层级控制器，将值设置到资源对象上。通过 setResourceByForm， 我们还可以做到从 UI 数据写回资源对象时，不去触碰 UI 表单没有的字段，避免了数据转化过程中数据可能会丢失的情况。
### ng-resource-form-util 资源表单辅助工具库
表单的单项数据流基本上可以用一张简单的图表示：

```
 +--------+
 |Resource|<<<-----+
 +---+----+        |
     |             |
 writeValue    onChange
     |             |
adaptResource  adaptForm
     |             |
  setForm     setResource
     |             |
     |         +------+
     +------>>>+ Form |
               +------+
```
由于控制器大多数情况下使用方式和行为高度相似，于是灵雀云前端将表单的这些功能和行为抽象、封装到了 BaseResourceFormComponent 基类内，并将 [代码开源在此](https://github.com/pengx17/k8s-form-in-action/tree/master/projects/ng-resource-form-util)。

上面的流程里还剩一些关键细节遗漏。我简要提一下思路，整理为如下Q&A：
Q: 表单是如何处理表单验证，或者甚至是异步表单验证逻辑，并向上传递表单验证状态的？
A: 将内部表单的错误处理封装为验证器，绑定到 DI 中获取到 NgControl 的 Validator 上
Q: 内部表单的提交逻辑如何处理？
A: 从 DI 中获取到 FormGroupDirective，监听表单提交事件
Q: 有的表单内部有些数据不属于当前 Kubernetes 对象。这种情况下如何处理？
A: 借助 ng-content 、TemplateRef 或是 Portal，将资源无关模板映射到组件内。

你可以通过 [DEMO](https://pengx17.github.io/k8s-form-in-action/demo) 和 [DEMO 的源码](https://github.com/pengx17/k8s-form-in-action/tree/master/src)继续了解一个比较完整的解决方案是怎样的。
# 写在最后
基于本文表单开发范式，灵雀云的前端开发可以非常快速的进行 K8S 相关资源对象表单的实现，并且得到 YAML 与资源对象互转的需求实现。

本文介绍了一种通用的基于 Angular 响应式表单编辑 Kubernetes 对象的实现思路与范式。实际上，这个思路并不只局限于 Angular 或者 Kubernetes 对象，读者甚至可以根据自己的需要，将此文章的思路使用[ final form ](https://github.com/final-form/final-form)带入到 React 或者 Vue 应用之中。

感谢阅读！
