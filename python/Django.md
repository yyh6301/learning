
# 安装&初始化

```shell
#pip 安装django
pip install django

#把django命令添加进环境变量
vim /etc/environment
PATH="$PATH:/usr/lib/python3.7.14/bin"

#重新加载配置文件
source /etc/environment

#初始化项目
django-admin startproject xxx

#创建app
python3 manage.py  startapp app01
```



# 目录结构
```shell
├── app01
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py      [数据库映射模型]
│   ├── tests.py       [测试用例]
│   └── views.py       [业务逻辑]
├── djangoExample
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-37.pyc
│   │   └── settings.cpython-37.pyc
│   ├── asgi.py
│   ├── settings.py    [配置信息]
│   ├── urls.py        [路由编写]
│   └── wsgi.py
└── manage.py
```



# 快速上手
1. 在settings.py 引入app
![[Pasted image 20240531090623.png]]


2. 在url.py当中编写接口路由
![[Pasted image 20240531090636.png]]


3.在app当中的views来编写具体业务逻辑 
![[Pasted image 20240531090652.png]]


4. 执行 python manage.py runserver 8000 即可启动服务



# orm操作数据库

## 配置信息
1. 安装mysqlclient
```shell
pip install mysqlclient
```


2. settings.py配置数据库连接信息


3. 迁移models
```shell
python manage.py makemigrations
python manage.py migrate
```

![[Pasted image 20240531090711.png]]



## 查询列表
```python
def user_list(request):
    data_list = models.UserInfo.objects.all()
    data_dict_list = list(data_list.values())
    return JsonResponse({"data":data_dict_list})
```



## 新增数据
```python
def add_user(request):
    json_data = json.loads(request.body)
    print(json_data)
    new_user = models.UserInfo.objects.create(username=json_data['username'],password=json_data['password'],age=json_data['age'])
    if new_user is not None:
        return JsonResponse({"message": "创建用户成功"})
    else:
        return JsonResponse({"error": "创建用户失败"})
```


## 更新数据
```python
def update_user(request):
    user_id = request.GET.get('id')
    json_data = json.loads(request.body)
    updated_count = models.UserInfo.objects.filter(id = user_id).update(username=json_data['username'],password = json_data['password'],age = json_data['age'])
    if updated_count == 1:
        return JsonResponse({"message": "更新用户成功"})
    else:
        return JsonResponse({"error": "未找到匹配的用户或更新失败"})
```



## 删除数据
```python
def delete_user(request):
    user_id = request.GET.get('id')
    print(user_id)
    delete_result =  models.UserInfo.objects.filter(id = user_id).delete()


    if delete_result[0] == 1:
        return JsonResponse({"message": "删除用户成功"})
    else:
        return JsonResponse({"error":"未找到匹配的用户id"})
```

