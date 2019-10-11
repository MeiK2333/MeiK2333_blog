---
title: Django Model field reference - Field options
date: 2018-02-22 00:00:01
tags: [Python, Django]
---

翻译自[官方文档](https://docs.djangoproject.com/en/2.0/ref/models/fields/)，django 中模型字段可用的参数及其含义。

<!--more-->

以下参数适用于所有字段类型，全部都是可选的。


# null


## Field.null


如果此项为 True ， django 将在数据库中以 NULL 来存储空值，默认为 False 。


避免在基于字符串的字段中使用 null ，如 CharField 和 TextField 。如果一个基于字符串的字段有 null=True ，则意味这它的“无数据”有两个可能的值： NULL 和空字符串。在大多数情况下，为“无数据”提供两个可能的值是多余的， django 的约定是使用空字符串，而不是 NULL 。有一个例外：当一个 CharField 字段有 unique=True 和 blank=True 的设置时， null=True 需要被设置以防止使用空值保存多个对象是出现的唯一约束。


对于基于字符串的字段和非基于字符串的字段，还需要设置 blank=True 来选择是否允许表单中的空值，因为 null 参数仅影响数据库的存储。


如果您想要使 BooleanField 字段接受 null ，则需要使用 NullBooleanField 字段来代替。


---


# blank


## Field.blank


如果此选项为 True ，该字段被允许为空白，默认为 False 。


请注意，这与 null 不同。 null 只与数据库相关，而 blank 是和验证相关的，如果一个字段中的 blank=True ，表单验证将允许输入一个空值，否则，该字段的值是必须的。


---


# choices


## Field.choices


一个可迭代的对象（比如列表或者元组），迭代的项是包括两个项目。例如 [(A, B), (A, B) ...] 。如果给定了此设置，则其对应的默认的表单部件是一个选择框与这些选项，而不是普通的文本字段。


每个元组的第一个元素是要在模型上设置的实际的值，第二个元素是可读的一个名称。例如：


```python
YEAR_IN_SCHOOL_CHOICES = (
    ('FR', 'Freshman'),
    ('SO', 'Sophomore'),
    ('JR', 'Junior'),
    ('SR', 'Senior'),
)
```


最好在模型类中定义选项，并为每个值定义一个适当命名的常量：


```python
from django.db import models

class Student(models.Model):
    FRESHMAN = 'FR'
    SOPHOMORE = 'SO'
    JUNIOR = 'JR'
    SENIOR = 'SR'
    YEAR_IN_SCHOOL_CHOICES = (
        (FRESHMAN, 'Freshman'),
        (SOPHOMORE, 'Sophomore'),
        (JUNIOR, 'Junior'),
        (SENIOR, 'Senior'),
    )
    year_in_school = models.CharField(
        max_length=2,
        choices=YEAR_IN_SCHOOL_CHOICES,
        default=FRESHMAN,
    )

    def is_upperclass(self):
        return self.year_in_school in (self.JUNIOR, self.SENIOR)
```


尽管可以在模型类之外定义一个选择列表并且引用它，但在模型类中定义将是的这些信息与类相关联，并且容易引用。


您还可以像下面这样定义它：


```python
MEDIA_CHOICES = (
    ('Audio', (
            ('vinyl', 'Vinyl'),
            ('cd', 'CD'),
        )
    ),
    ('Video', (
            ('vhs', 'VHS Tape'),
            ('dvd', 'DVD'),
        )
    ),
    ('unknown', 'Unknown'),
)
```


对于 choices 中设置的每个模型字段， django 将添加一个方法来检索当前字段对应的可读名称，即 get_FOO_display()：


```python
from django.db import models

class Person(models.Model):
    SHIRT_SIZES = (
        ('S', 'Small'),
        ('M', 'Medium'),
        ('L', 'Large'),
    )
    name = models.CharField(max_length=60)
    shirt_size = models.CharField(max_length=2, choices=SHIRT_SIZES)

>>> p = Person(name="Fred Flintstone", shirt_size="L")
>>> p.save()
>>> p.shirt_size
'L'
>>> p.get_shirt_size_display()
'Large'
```


选择可以是任何的可迭代对象，不一定是列表或者元组。这让你可以动态的构建选择。但如果你的 choices 是动态的，那么最好有一个合适的数据库表来存储。


除非 blank=False 在字段中被设置，否则选择框默认将显示 default 的标签"---------"，要覆盖此行为，请在 choices 元组中添加 None，比如：(None, 'Your String For Display')。或者，你也可以使用空字符串。


---


# db_column


## Field.db_column


用于设置此字段在数据库中的名称，如果没有给定， django 将使用该字段的名称。


此设置可以为 SQL 保留字或者 Python 变量名不允许使用的字符， django 将自动处理好后台的联系。


---


# db_index


## Field.db_index


如果此设置为 True ，则为此字段创建数据库索引。


---


# db_tablespace


## Field.db_tablespace


如果此字段已经设置了索引，则用此字段索引数据库表空间的名称。默认值是项目的DEFAULT_INDEX_TABLESPACE设置（如果已设置）或 db_tablespace模型（如果有）。如果后端不支持索引的表空间，则忽略此选项。


注意：SQLite 和 MySQL 不支持数据库表空间的设置，PostgreSQL和Oracle支持。此设置是通过组织磁盘布局来优化数据库性能的。


---


# default


## Field.default


字段的默认值，可以是一个常量值或者一个可调用的对象。如果是一个可调用的对象，则在每次创建新对象的时候都会调用它。


default 的值不能是可变对象（模型实例、list、set等），因为对该对象的同一实例的引用将被用作所有新模型实例中的默认值。除非将所需的默认值包装成一个可调用的对象。例如，如果想要为 JSONField 字段指定一个默认的 dict ，使用函数：


```python
def contact_default():
    return {"email": "to1@example.com"}

contact_info = JSONField("ContactInfo", default=contact_default)
```
lambda 不能用于该字段选项，因为他们无法被迁移序列化（关于迁移序列化，详见 django 官网文档）。


对于像ForeignKey映射到模型实例的字段，默认值应该是它们引用的字段的值（pk，除非通过to_field设置引用了其他键），而不是模型实例。


default 是在未为该字段提供值的时候使用的默认值，如果该字段是主键，则即使有该设置，也会使用默认值。


---


# editable


## Field.editable


如果该设置为 False ，则该字段不会在管理员或者 [ModelForm](https://docs.djangoproject.com/en/2.0/topics/forms/modelforms/#django.forms.ModelForm) 中被显示，在模型验证中也会被跳过。默认为 True 。


---


# error_messages


## Field.error_messages


该参数允许您覆盖该字段异常时引发的默认消息，传入一个字典，其中包含与要覆盖的错误信息相匹配的键。


错误信息键包括：null，blank，invalid，invalid_choice，unique，unique_for_date。


这些错误通常不会被传递到表单，官方文档也没有关于此字段的使用事例和更多的说明。


---


# help_text


## Field.help_text


使用默认的窗体小部件时显示的额外的“帮助”文本，即使此字段并没有被用于表单，作为字段的文档也很有用。


如果在自动生成的表单中显示的话，则不会被 HTML 转义，如：


```python
help_text="Please use the following format: <em>YYYY-MM-DD</em>."
```


如果有必要的话，也可以通过 django.utils.html.escape() 来转义 HTML 特殊字符，确保避开来自不受信任的用户的跨站脚本攻击。


---


# primary_key


## Field.primary_key


如果为 True ，则此字段为模型的主键。


如果没有指定一个字段的 primary_key=True ，则 django 会自动添加一个 AutoField 来保存主键。因此除非必要，请不要在其他字段上设置 primary_key=True 。


primary_key=True 同时代表了 null=False 和 unique=True ，一个对象值允许有一个主键。


主键的字段是只读的，如果您更改了一个已有的对象的主键并保存，那么会保存一个额外的对象（而不会修改之前的对象）。


---


# unique


## Field.unique


如果此参数为 True ，则该字段在整个表中必须是唯一的。


这是在数据库级别与模型验证级别实施的，如果在 unique=True 的字段中保存具有重复值的对象，则会引发一个 django.db.IntegrityError 的错误。


此选项在除了 ManyToManyField 和 OneToOneField 以外的任何字段类型中都有效。


请注意：如果 unique 为 True ，则不需要指定 db_index ，因为 unique 意味着创建唯一索引。


---


# unique_for_date


## Field.unique_for_date


将其值设置为一个 DateField 或者 DateTimeField 的名称，要求此字段对于给定的日期字段的值是唯一的。


举例来说：如果有一个 CharField 字段 title 和 DateField 字段 pub_time ， title 字段的 unique_for_date 值为 'pub_time'，那么不允许 title 和 pub_time 同时相同的数据出现。


这个设置是在模型验证期间进行的验证，不会在数据库级别验证。因此，如果字段的 editable=False 的话，将跳过此验证。


---


# unique_for_month


## Field.unique_for_month


类似 unique_for_date，但该设置要求月份是唯一的。


---


# unique_for_year


## Field.unique_for_year


我不说你们也明白了吧...


---


# verbose_name


## Field.verbose_name


该字段的可读名称，如果没有给出详细名称， django 将会自动使用该字段的属性名，并把下划线转换为空格。


---


# validators


## Field.validators


要为此字段运行的验证程序列表。


```python
from django.core.exceptions import ValidationError
from django.utils.translation import gettext_lazy as _

def validate_even(value):
    if value % 2 != 0:
        raise ValidationError(
            _('%(value)s is not an even number'),
            params={'value': value},
        )
```


```python
from django.db import models

class MyModel(models.Model):
    even_field = models.IntegerField(validators=[validate_even])
```


可以在窗体中使用同样的验证函数


```python
from django import forms

class MyForm(forms.Form):
    even_field = forms.IntegerField(validators=[validate_even])
```


## Registering and fetching lookups


这个我看不懂，就直接放官方文档吧。


Field implements the [lookup registration API](https://docs.djangoproject.com/en/2.0/ref/models/lookups/#lookup-registration-api). The API can be used to customize which lookups are available for a field class, and how lookups are fetched from a field.
