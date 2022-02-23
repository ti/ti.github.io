---
picture: image/dropbox_geoblast.jpg
tags: [Markdown, 模板]
date: 2019-03-05
---
# 分享几个常用的 Markdown 模板

Markdown 在我们应用日常使用中基本上已经替代其他文本工具了, 但是和其他文本编辑器不同的是, 没有太多新建项目模板, 前几天看到[Dropbox Paper](https://www.dropbox.com/paper)上

```

```

面的几个 Markdown 模板,做的很不错，　我将其整理下来，　并附上常用的几个Markdown分档分享．

模板包含:

* [头脑风暴](#头脑风暴)
* [会议记录](#会议记录)
* [项目计划](#项目计划)
* [RestAPI文档](#restapi文档)

最后，附上英文文档源码

* [英文文档源码](#英文文档源码)

## 头脑风暴

**源码**

```markdown
# 💡头脑风暴： 添加主题

| **目标**  | 您想要什么样的想法？              |
| ------- | ----------------------- |
| **参与者** | @提及您自己和其他人              |
| **相关**  | - 🔗 [插入一些文档或链接]() |

## 灵感

使用您点击“返回”时显示的工具栏来添加图片、视频等


## 想法

向团队提出一个问题，让事情顺利进行

----------
## 后续步骤

- [ ] 打破陈规 @某人
- [ ] 从待办到完成 @某人

```

**显示效果** (已调整缩进)

### 💡头脑风暴： 添加主题

| **目标**   | 您想要什么样的想法？                                                |
| ---------------- | ------------------------------------------------------------------- |
| **参与者** | @提及您自己和其他人                                                 |
| **相关**   | -[相关文档１](document1.md) `<br>`- [插入相关文档](http://document2.md) |

#### 灵感

使用您点击“返回”时显示的工具栏来添加图片、视频等

#### 想法

向团队提出一个问题，让事情顺利进行

---

#### 后续步骤

- [ ] 打破陈规 @某人
- [ ] 从待办到完成 @某人

## 会议记录

**源码**

```markdown
# 🕘 会议记录： 添加事件名称

# 2019-03-02

****
## 与会者

@提及您自己并添加其他人


## 议程
- 会议议程


## 讨论
- 我们实际讨论的内容


## 操作项目

- [ ] 我们来完成这项任务 @某人
```

**显示效果** (已调整缩进)

### 🕘 会议记录： 添加事件名称

### 2019-03-02

---

#### 与会者

@提及您自己并添加其他人

#### 议程

- 会议议程

#### 讨论

- 我们实际讨论的内容

#### 操作项目

- [ ] 我们来完成这项任务 @某人

---

## 项目计划

```
# 📋 项目计划： 添加项目

| **说明** | 您有什么想法？                 |
| ------ | ----------------------- |
| **状态** | 提前 🌱 | 进行中 🔨 | 已完成 ⭐  |
| **团队** | 身份: @某人<br>身份: @某人      |
| **相关** | - [相关文档１](document1) <br>- [插入相关文档](http://document) |

## 时间线

| 标题 | 日期(Deadline) | 已指派给 | 说明 |
| --- | :---------: | ------- | --- |
|     | 03.01-03.12 |         |     |

## 详细信息

使用您点击“返回”时显示的工具栏来添加图片、视频等


## 待办事项

- [ ] 我们将会完成这项任务 @某人
- [ ] 我们今天就来解决这项任务 @某人
```

**显示效果** (已调整缩进)

### 📋 项目计划： 添加项目

| **说明** | 您有什么想法？                                                      |
| -------------- | ------------------------------------------------------------------- |
| **状态** | 提前 🌱                                                             |
| **团队** | 身份: @某人`<br>`身份: @某人                                      |
| **相关** | -[相关文档１](document1) `<br>`- [插入相关文档](http://x.com/document2) |

#### 时间线

| 标题 |    日期    | 已指派给 | 说明 |
| ---- | :---------: | -------- | ---- |
|      | 03.01-03.12 |          |      |

#### 详细信息

使用您点击“返回”时显示的工具栏来添加图片、视频等

#### 待办事项

- [ ] 我们将会完成这项任务 @某人
- [ ] 我们今天就来解决这项任务 @某人

---

## RestAPI文档

```
### The title of your rest api

Some description of your api

**Method:** POST

**Content-Type:** application/json

**Endpoint**

```bash
https://apis.example.com/v1/example
```

**Request Body Payload**

| Property Name | Type    | Description                |
| ------------- | ------- | -------------------------- |
| name          | string  | the name of this model   |
| admin         | boolean | Whether the model is admin |

**Response Payload** 

| Property Name | Type   | Description                 |
| ------------- | ------ | --------------------------- |
| id            | string | the id of the server        |
| name          | string  | the name of this model     |
| admin         | boolean | Whether the model is admin |

**Sample request**

```json
curl 'https://apis.example.com/v1/example' \
-H 'Content-Type: application/json' \
-H 'Authorization: ${TOKEN}' \
-d \
'{
    "name": "test",
    "admin": true
}' 
```
In the example above, you would replace `${TOKEN}`  with the generated Auth token,  Refer:  [User Authorization ](ref/auth/auth.md).

A successful request is indicated by a `200 OK` HTTP status code. The response contains id of your model

**Sample response**

```json
{
  "id": "A163D1C8AA1FD43CAF521B74FC5A46BE",
  "name": "test",
  "admin": "true",
  "expiresIn": "3600"
}
```

**Common error codes**

- `invalid_argument`: the [name] may be tool long for database
- `permission_denied`: you may have no permission for this operation

Other errors you may need to refer to the [global error code](ref/errors/global.md).
```

**显示效果** (已调整缩进)

### The title of your rest api

Some description of your api

**Method:** POST

**Content-Type:** application/json

**Endpoint**

```bash
https://apis.example.com/v1/example
```

**Request Body Payload**

| Property Name | Type    | Description                |
| ------------- | ------- | -------------------------- |
| name          | string  | the name of this model     |
| admin         | boolean | Whether the model is admin |

**Response Payload**

| Property Name | Type    | Description                |
| ------------- | ------- | -------------------------- |
| id            | string  | the id of the server       |
| name          | string  | the name of this model     |
| admin         | boolean | Whether the model is admin |

**Sample request**

```json
curl 'https://apis.example.com/v1/example' \
-H 'Content-Type: application/json' \
-H 'Authorization: ${TOKEN}' \
-d \
'{
    "name": "test",
    "admin": true
}' 
```

In the example above, you would replace `${TOKEN}`  with the generated Auth token,  Refer:  [User Authorization ](ref/auth/auth.md).

A successful request is indicated by a `200 OK` HTTP status code. The response contains id of your model

**Sample response**

```json
{
  "id": "A163D1C8AA1FD43CAF521B74FC5A46BE",
  "name": "test",
  "admin": "true",
  "expiresIn": "3600"
}
```

**Common error codes**

- `invalid_argument`: the [name] may be tool long for database
- `permission_denied`: you may have no permission for this operation

Other errors you may need to refer to the [global error code](ref/errors/global.md).

## 英文文档源码

**Brainstorm**

```
# 💡 Brainstorm: Add a topic

| **Goal**         | What kinds of ideas do you want?    |
| ---------------- | ----------------------------------- |
| **Participants** | @mention yourself and others        |
| **Related**      | - + (some Related documents ) |

## Inspiration

Add images, videos, and more by using the toolbar that appears when you hit return


## Ideas

Ask the group a question to get things flowing


----------
## Next steps
[ ] Think outside the box @someone
[ ] From to do to done @someone
```

**Project plan**

```
# 📋 Project plan: Add a project

| **Description** | What do you have in mind?                                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Status**      | Early 🌱 | In progress 🔨 | Finished ⭐                                                                                                                   |
| **Team**        | Role: @someone<br>Role: @someone                                                                                                                         |
| **Related**     | - [document1](document1.md) <br>- [document2](http://document2.md) |

## Timeline
Title | Dates | Assigned to | Description
--- | --- | --- | ---
 | March 1, 2019 - March 6, 2019 |  | 
## Details

Add images, videos, and more by using the toolbar that appears when you hit return


## To-dos
[ ] We’ll get this done @someone
[ ] Let’s tackle this today @someone
```

**Meeting notes***

```
# 🕘 Meeting notes: Add an event name

# Mar 2, 2019

****
## Attendees

@mention yourself and add others


## Agenda
- Stuff to talk about


## Discussion
- Stuff we actually talked about


## Action items
[ ] Let’s get this done @someone

```

**REST API Templates**

```
### The title of your rest api

Some description of your api

**Method:** POST

**Content-Type:** application/json

**Endpoint**

```bash
https://apis.example.com/v1/example
```

**Request Body Payload**

| Property Name | Type    | Description                |
| ------------- | ------- | -------------------------- |
| name          | string  | the name of this model   |
| admin         | boolean | Whether the model is admin |

**Response Payload** 

| Property Name | Type   | Description                                                  |
| ------------- | ------ | ------------------------------------------------------------ |
| id            | string | the id of the server |
| name          | string  | the name of this model     |
| admin         | boolean | Whether the model is admin |

**Sample request**

```json
curl 'https://apis.example.com/v1/example' \
-H 'Content-Type: application/json' \
-H 'Authorization: ${TOKEN}' \
-d \
'{
    "name": "test",
    "admin": true
}' 
```
In the example above, you would replace `${TOKEN}`  with the generated Auth token,  Refer:  [User Authorization ](ref/auth/auth.md).

A successful request is indicated by a `200 OK` HTTP status code. The response contains id of your model

**Sample response**

```json
{
  "id": "A163D1C8AA1FD43CAF521B74FC5A46BE",
  "name": "test",
  "admin": "true",
  "expiresIn": "3600"
}
```

**Common error codes**

- `invalid_argument`: the [name] may be tool long for database
- `permission_denied`: you may have no permission for this operation

Other errors you may need to refer to the [global error code](ref/errors/global.md).
```
