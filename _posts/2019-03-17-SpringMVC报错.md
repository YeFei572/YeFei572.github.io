---
layout: post
tags: SpringMVC
categories: 犯错记录
title:  "SpringMVC报错:The given id must not be null"
---

普通看这个错误肯定是入参id为空了, 但是有一种情况是你入参没有id这个参数, 系统还是报这个错误, 比如如下代码:
```java
  @RequestMapping(
		  value = "/messages/{userId}",
		  method = RequestMethod.GET,
		  produces = MediaType.APPLICATION_JSON_VALUE
  )
  public PageData<AppMessage> getMessages(
		  @PathVariable String userId,
		  @RequestParam(value = "page", required = false, defaultValue = "0") int page,
		  @RequestParam(value = "size", required = false, defaultValue = "20") int size
  ) {
	  SecurityUtil.checkIsUserHimself(userId);
	  List<AppMessage> messages = appMessageService.getMessages(page, size, userId);
	  fetchProfile(messages);
	  return new PageData<>(page, size, (appMessageService.countMessage(userId) / size) + 1, messages);
  }
```

以上代码的入参是userId, 但是项目任然报这个错误!

问题根源:

**这个时候去检查下面的代码, springData的接口中只要你用了主键(索引)去查数据,那么springmvc也会报这个错误!**