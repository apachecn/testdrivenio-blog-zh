# Django REST 框架教程

> 原文：<https://testdriven.io/blog/topics/django-rest-framework/>

## 描述

Django REST Framework (DRF)是一个广泛使用的全功能 API 框架，旨在用 Django 构建 RESTful APIs。在其核心，DRF 集成了 Django 的核心特性——模型、视图和 URLs 使得创建 RESTful HTTP 资源变得简单而无缝。

DRF 由以下部分组成:

1.  **序列化器**用于将 Django 查询集和模型实例转换为 JSON(以及许多其他数据呈现格式，如 XML 和 YAML )(序列化)和(反序列化)。
2.  **视图**(以及视图集)，类似于传统的 Django 视图，处理 RESTful HTTP 请求和响应。视图本身使用序列化程序来验证传入的有效负载，并包含返回响应的必要逻辑。视图与**路由器**耦合，路由器将视图映射回公开的 URL。

TestDriven.io 上的文章和教程是比较中级到高级的，涵盖了权限、序列化器和 Elasticsearch。

深入了解 Django REST 框架最强大的视图 ViewSets。

使用 Django REST Framework 的通用视图来防止一遍又一遍地重复某些模式。

深入探讨 Django REST 框架的视图是如何工作的，以及它最基本的视图 APIView。

*   由 [发布![Nik Tomazic](img/06d2b958481802809b8daf74b93ff2c8.png)尼克·托马齐奇](/authors/tomazic/)
*   最后更新于2021 年 8 月 31 日

查看如何将 Django REST 框架与 Elasticsearch 集成。

如何在 Django REST 框架中构建自定义权限类？

Django REST 框架中内置权限类的工作方式。

Django REST 框架中权限的工作方式。

深入研究 Django REST 框架(DRF)序列化程序。