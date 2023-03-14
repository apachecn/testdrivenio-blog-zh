# 任务队列教程

> 原文：<https://testdriven.io/blog/topics/task-queue/>

## 描述

任务队列在用户请求之外异步管理后台工作。对于 web 开发，它们用于处理典型的请求/响应周期之外的任务。它们在微服务架构中非常流行，用于微服务间的通信。Celery 和 RQ 是两种最流行的基于 Python 的任务队列。

TestDriven.io 上的教程和文章关注于设置和配置任务队列，以便与 Docker 和各种 Python web 框架(如 Django、Flask 和 FastAPI)一起工作。

本文着眼于如何在 Django 应用程序中配置 Celery 来处理长时间运行的任务。

本教程展示了如何将 Celery 集成到基于 Python 的 Falcon web 框架中。

查看如何配置 Celery 来处理 Flask 应用程序中的长时间运行的任务。

查看如何配置 Redis 队列(RQ)来处理 Flask 应用程序中的长时间运行的任务。

*   发帖者 [![Michael Yin](img/360e937c2c34345187e6016b33e66d4e.png)迈克尔·尹](/authors/yin/)
*   最后更新于2021 年 12 月 13 日

让 Celery 很好地与 Django 数据库事务一起工作。

*   发帖者 [![Michael Yin](img/360e937c2c34345187e6016b33e66d4e.png)迈克尔·尹](/authors/yin/)
*   最后更新于2021 年 12 月 11 日

自动重试失败的芹菜任务。

如何使用 Python 多重处理库和 Redis 实现几个异步任务队列？

向 Flask、Redis Queue 和 Amazon SES 的新注册用户发送确认电子邮件。

这篇文章着眼于如何管理 Django、Celery 和 Docker 的周期性任务。

这篇文章着眼于如何配置 Celery 来处理 FastAPI 应用程序中的长时间运行的任务。