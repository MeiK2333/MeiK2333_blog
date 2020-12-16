---
title: Django 动态更改文件上传路径
date: 2018-04-03 21:42:47
tags: [Python, Django]
---

Django 中的 FileField 模型可以提供方便的文件上传等操作方法，其中，可以通过修改 `upload_to` 设置来动态的选择文件上传路径。


<!--more-->

之前我曾经翻译过 [Django 中的模型字段定义](https://meik2333.com/index.php/archives/4/)，当时只是照着翻译了 FileField 字段定义，对于具体的用法没有深究。而之后再具体使用这些模型时出现了一些问题，我想要根据上传的用户来动态的选择文件上传路径，而在搜索引擎进行搜索的时候发现，绝大部分人对于这个模型的理解很差，他们所说的动态其实就是把一个固定的文件夹改成了另一个固定的文件夹。

我不认为 Django 会没有根据上传用户来动态修改文件路径的办法，因此我回去翻了翻英文文档 [FileField.upload_to](https://docs.djangoproject.com/en/2.0/ref/models/fields/#django.db.models.FileField.upload_to) ，果然发现了解决方法。

FileField 可以通过给 `upload_to` 设置一个可调用的方法来实现动态的路径，这个方法接收两个参数：

| Argument | Description |
| --- | --- |
| instance | FileField定义模型的一个实例 。更具体地说，这是当前文件所在的特定实例。 |
| filename | 最初提供给该文件的文件名。在确定最终目的地路径时，这可能会也可能不会被考虑在内。 |

一个例子：

```python
from django.contrib.auth import get_user_model

def user_directory_path(instance, filename):
    # file will be uploaded to MEDIA_ROOT/user_<id>/<filename>
    return 'user_{0}/{1}'.format(instance.user.id, filename)

class MyModel(models.Model):
    user = models.ForeignKey(get_user_model(), on_delete=models.CASCADE)
    upload = models.FileField(upload_to=user_directory_path)
```

这样可以把文件动态的存储为 `user_{user_id}/{filename}` ，这是一个对于 `setting.py` 中设置的 `MEDIA_ROOT` 目录的相对路径。

在设置时也可以使用模型中定义的其他字段，但是要注意的是，这个路径的计算是在对象存到数据库之前进行的，因此，如果模型中定义了 `AutoField` 自增字段，那么，在设置路径时，这个字段还没有值。