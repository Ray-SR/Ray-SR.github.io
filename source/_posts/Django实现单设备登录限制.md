---
title: Django实现单设备登录限制
date: 2017-09-05 20:58:59
tags: [Django,Python]
categories: Django
---

本文旨在介绍Django项目实现单设备登录限制的一种方法，基于Django的session机制。

<!--more-->

## 基本思路
要保证同一账号只能在一台设备登录，基本思路就是维护一个用户登录状态记录，用户每次请求或登录时，先从登录状态记录中查询，如果该账号已经在其他设备登录，则删除或禁止之前的记录。
## 使用Mysql数据表
为保证用户账号安全，限制同一账号只能在唯一一台设备登录，之前的实现思路是建立一个用户登录状态数据表，每次用户登录都根据用户id查询，判断是否有用户id相同的记录，有则删除，同时，在用户请求后端视图时，从session中取出用户id，再查询登录状态记录表，如果该session对应的记录已被删除，则失效该session。
实际测试时发现，维护用户登录状态记录表过于繁琐，并且由于用户每次请求都需要查询数据库，对性能也有一定影响，该方案不太可行。

## 使用Redis缓存session
Django项目可以使用Redis作为缓存数据库，Redis是内存数据库，它读写速度快、灵活方便的特点很适合少量数据频繁读写。因此，可以将session保存在Redis中，用户登录时，首先生成session，然后查询是否有用户id相同的记录，如果用户id相同而session_key不同，则说明该账号已经在其他地方登录，此时直接删除之前的session，只保留当前session，因此之前登录的用户就会被挤出。关键代码如下：

```python
# 获取当前session的session_key
session_key = request.session.session_key

# 获取Redis中所有key
key_list = cache.keys("*")

# 遍历获取到的所有key，通过正则筛选django的session记录
for key in key_list:
    s_key = re.match(r'django\.contrib\.sessions\.cache(.*)', key)
    # 如果session_key和当前session不同，则进行判断
    if s_key and s_key.group(1) != session_key:
        cache_session_dict = cache.get(key)
        # 如果session信息中保存的user_id和当前用户id相同，则表明该账号已登录
        if cache_session_dict.get('user_id') == user.id:
            cache.delete(key)
            logger.info('Account [{}] has logged in elsewhere, delete old session [{}]'.format(username, key))
            else:
                continue
```

## 视图装饰器
为了实现登录状态的判断，可以在Django视图上加上装饰器，从而判断每次request对应的session是否被删除，同时进行其他权限校验。

```python
def auth_required(perm):
    def decorator(view_func):
        def _wrapped_view(request, *args, **kwargs):

            try:
                session_key = request.session.session_key

                # session_flag = Session.objects.filter(session_key=session_key)
                if not session_key:
                    return JsonResponse({"respCode": 4001})
                else:
                    role = request.session.get("role_id")
                    if not role:
                        return JsonResponse({"respCode": 4002})
                    else:
                        if int(role) > int(perm):
                            return JsonResponse({"respCode": 4003})

            except Exception as e:
                logger.error(e)
                return JsonResponse({"respCode": 4004})

            return view_func(request, *args, **kwargs)
        return _wrapped_view
    return decorator
```

