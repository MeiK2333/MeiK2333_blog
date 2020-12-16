---
title: Django Model field reference - Field types
date: 2018-02-22 00:00:00
tags: [Python, Django]
---

翻译自[官方文档](https://docs.djangoproject.com/en/2.0/ref/models/fields/)，django 中可用的模型字段及其含义。
<!--more-->


# AutoField


一个 id 自动递增的 IntegerField 字段，您通常不需要直接使用它，如果不指定主键字段，那么一个 AutoField 字段将被自动添加到模型中。


---


# BigAutoField


和 AutoField 类似，不同之处在于，它的数字范围是 1 到 9223372036854775807 。


---


# BigIntegerField


一个 64 位整数，类似 IntegerField ，不同之处在于他的合法数字范围是 -9223372036854775808 到 9223372036854775807 。该字段的默认表单部件是 TextInput 。


---


# BinaryField


用于存储原始二进制数据的字段，它只支持 bytes 。请注意，该字段的功能有限，比如不支持过滤查询集，也不可能包含在一个 ModelForm 中。


尽管您可能会考虑将数据存储在数据库中，但 99% 的情况下这是糟糕的设计，请考虑使用静态文件来代替。


---


# BooleanField


True / False


该字段的默认表单部件是 CheckboxInput


如果您需要接受 null 值，则应该使用 NullBooleanField 。


---


# CharField


用于存储字符串，对于大量的文字，请使用 TextField 。


该字段的默认部件是 TextInput 。


CharField 有一个额外的必要参数：


## CharField.max_length


字段的最大长度（以字符为单位）。在数据库级别和 django 验证中执行。


---


# DateField


日期，由 Python 中的 datetime.date 实例表示，有一些额外的可选参数：


## DateField.auto_now


每次保存对象是自动将字段设置为现在，仅在 model.save() 时更新，使用其他方式更新字段时不会自动更新，比如 QuerySet.update() 。


## DateField.auto_now_add


首次创建对象时，自动将字段设置为现在。如果设置了此选项，则字段的 default 设置将失效。如果您希望自动保存其他形式的日期：
- For DateField: default=date.today - from  datetime.date.today()
- For DateTimeField: default=timezone.now - fromdjango.utils.timezone.now()


该字段的默认部件为 TextInput。


---


# DateTimeField


日期和时间，由 Python 的 datetime.datetime 实例表示。默认部件为 TextInput 。


与 DateField 有相同的额外参数（auto_now 和 auto_now_add）。


---


# DecimalField


一个固定精度的十进制数，由 Python 中的 Decimal 实例表示，有两个必需的参数：


## DecimalField.max_digits


允许的最大位数，此数字必须大于等于 decimal_places 。


## DecimalField.decimal_places


用数字存储的小数位数。


例如，存储数字 999 ，精确到小数点后两位：


```python
models.DecimalField(..., max_digits=5, decimal_places=2)
```


存可能到十亿的数字，精确到小数点后十位：


```python
models.DecimalField(..., max_digits=19, decimal_places=10)
```


该字段的默认表单控件为 NumberInput 。


---


# DurationField


用于存储时间段的字段，由 Python 中的 timedelta 表示。


在不同的数据库中表现可能不同，请谨慎使用。


---


# EmailField


顾名思义……将会由 EmailValidator 验证。


---


# FileField


文件上传字段，不支持 primary_key 参数，如果使用将出错。


有两个可选的参数：


## FileField.upload_to


该属性提供了设置上传目录和文件名的方法，有两种设置方式：
您可以指定一个字符串值，它可以包含 strftime() 格式，将会被上传的时间所取代。例如：


```python
class MyModel(models.Model):
    # file will be uploaded to MEDIA_ROOT/uploads
    upload = models.FileField(upload_to='uploads/')
    # or...
    # file will be saved to MEDIA_ROOT/uploads/2017/02/21
    upload = models.FileField(upload_to='uploads/%Y/%m/%d/')
```


您也可以使用默认值 FileSystemStorage ，然而我没看懂使用默认值会如何处理。


upload_to 可以被调用，返回上传路径。关于此项更详细的介绍，可以查看官方文档。


## FileField.storage


用于处理文件的存储和检索。请参阅[Managing files](https://docs.djangoproject.com/en/2.0/topics/files/)


该字段默认的部件是 ClearableFileInout 。


在模型中使用 FileField 或者 ImageField 需要几个步骤：


1. 在 settings.py 中，需要定义 MEDIA_ROOT ，是您想要 django 存储上传文件的目录的完整路径，（出于性能考虑，这些文件不应当存在数据库中）。定义 MEDIA_URL （例如："/media/"），确保该目录可以由 Web 服务器的用户账户写入。


2. 将 FileField 或 ImageField 添加到您的模型中，定义 upload_to 选项以指定 MEDIA_ROOT 下用于上传文件的子目录。


3. 所有存在数据库中的数据都是路径， django 提供了一个名为 url 的便利属性，如果你的 ImageField 的字段名为 mug_shot ，那么你可以在模板中获得图像的绝对路径： {{ object.mug_shot.url }} 。


例如您 MEDIA_ROOT 的设置为 '/home/media'，并且 upload_to 设置为 'photos/%Y/%m/%d'，其中的'%Y/%m/%d'将会被 strftime() 格式化，假设在2007年1月15日上传文件，那么文件将被保存至目录 /home/media/photos/2007/01/15 。


如果您想要检索上传文件的文件名或者文件大小，可以使用 name 和 size 属性，有关更多的可用属性和方法，请参阅 [File](https://docs.djangoproject.com/en/2.0/ref/files/file/#django.core.files.File)类的定义和[Managing files](https://docs.djangoproject.com/en/2.0/topics/files/) 。


请注意，无论何时，都应该密切的关注上传的文件以及它们的类型，以避免安全漏洞。


FileField 在数据库中创建为 varchar 的默认最大长度为 100 个字符的列，可以使用 max_length 参数来修改其最大长度。


## FileField 和 FieldFile


当访问 FileField 模型时，会提供一个 FieldFile 实例作为访问基础文件的代理。


实例从 File 类继承了诸如 read 和 write 等方法，save 与 delete 方法将会影响到 FieldFile 数据库中关联的模型对象。


### FieldFile.name


文件的名称


### FieldFile.size


文件大小


### FieldFile.url


访问文件的相对 URL


### FieldFile.open(mode='rb')


pass


### FieldFile.close()


pass


### FieldFile.save(name, content, save=True)


此方法获取文件名和文件内容，并将它们传递给字段的存储类。


coutent 参数 应该是 django.core.files 对象的一个实例：


```python
from django.core.files import File
# Open an existing file using Python's built-in open()
f = open('/path/to/hello.world')
myfile = File(f)


# 使用 Python 字符串构造
from django.core.files.base import ContentFile
myfile = ContentFile("hello world")


```


### FieldFile.delete(save=True)


删除与此实例相关联的文件并清除字段上的所有属性，可选参数 save 空值在删除文件后是否保存模型实例，默认为 True 。


请注意：删除模型时，相关的文件不会被删除，您需要自己处理它。


---


# FilePathField


请参照官方文档（我确实不知道是干啥的）。


---


# FloatField


由 Python 中的 float 表示的浮点数。


默认控件为 NumberInput 。


---


# ImageField


继承 FileField 的所有属性和方法，同时会验证上传的是有效的图像。


有两个额外的可选参数：


- ImageField.height_field


- ImageField.width_field


需要 pillow 库。


该字段的默认表单部件是 ClearableInput 。


---


# IntegerField


一个整数 -2147483648 到 2147483647 ，默认表单控件为 NumberInput 。


---


# GenericIpAddressField


字符串格式的 IPv4 或者 IPv6 地址，默认表单控件为 TextInput 。


具体的请参照官方文档。


---


# NullBooleanField


类似 BooleanField ，但允许 NULL 作为其中一个选项，默认表单部件为 NullBooleanSelect 。


---


# PositiveIntegerField


类似 IntegerField ，但是必须是正数或 0 ，范围是 0 到 2147483647 。


---


# PositiveSmallIntegerField


类似 PositiveIntegerField ，范围是 0 到 32767 。


---


# SlugField


一个短的标签，只包含字母、数字、下划线和连字符，通常被用于 URL ，长度限制为 50 。


---


# SmallIntegerField


类似 IntegerField ，范围 -32768 到 32767 。


---


# TextField


一个大文本字段，默认表单小部件为 Textarea 。


---


# TimeField


一个时间，由 Python 中的 datetime.time 实例表示，与 DateField 可以使用相同的自动填充选项。


该字段的默认表单部件为 TextInput 。


---


# URLField


一个 CharField ，代表一个 URL ，默认长度限制为 200 。


默认部件为 TextInput 。


---


# UUIDField


用于存储通用唯一标识符的字段，使用 Python 的 uuid 类。


可用于 AutoField 的 primary_key


```python
import uuid
from django.db import models

class MyUUIDModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    # other fields
```
