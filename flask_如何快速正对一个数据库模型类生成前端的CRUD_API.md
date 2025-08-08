需要安装几个模块：

```
flask-sqlalchemy
flask-migrate
marshmallow
marshmallow-sqlalchemy
```

1.配置数据库

创建一个app_config.py的文件，该文件用来设置app的配置。这样的好处是其他文件导入db不会造成太多的循环导入的问题。

app_config.py

```python
db: SQLAlchemy = SQLAlchemy()
SECRET_KEY = "secret!"
SQLALCHEMY_DATABASE_URI = f'sqlite:///store.lib'
SQLALCHEMY_TRACK_MODIFICATIONS = False
```

2.在app.py中初始化数据库的内容：

```python
from flask_migrate import Migrate

app = Flask(__name__)
import app_config
app.config.from_object(app_config)
db.init_app(app)#初始话数据库
migrate = Migrate(app, db)#创建一个migrate来管理数据的表的迁移
```



3.迁移相关的命令

```
flask --app .\app.py  db init #初始化生成迁移文件夹
flask --app .\app.py  db migrate #修改模型后生成迁移内容
flask --app .\app.py  db upgrade #执行迁移内容
```



4.表模型的创建

```python
class Project(db.Model):
    __tablename__ = 'projects'
    id = db.Column(db.String(64), primary_key=True)
    name = db.Column(db.String(32), nullable=False)
    project_file_path = db.Column(db.String(256), nullable=False)
    file_size = db.Column(db.String(256),default="", nullable=True)
    create_time = db.Column(db.DateTime, nullable=True, default=datetime.now())
    update_time = db.Column(db.DateTime, nullable=True, default=datetime.now(), onupdate=datetime.now())
```

常用的字段类型

| 类型                        | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `Integer`                   | 整型（int）                                                  |
| `SmallInteger`              | 小整型（适合范围较小的整数）                                 |
| `BigInteger`                | 大整型                                                       |
| `Float`                     | 浮点型                                                       |
| `Numeric(precision, scale)` | 高精度小数，适用于金额（如 `Numeric(10, 2)` 表示最多 10 位，其中 2 位小数） |
| `Boolean`                   | 布尔型（True / False）                                       |
| `String(length)`            | 可变长度字符串                                               |
| `Text`                      | 不限制长度的文本                               
| `DateTime`                  | 日期时间（`yyyy-mm-dd HH:MM:SS`）                            |
| `Interval`                  | 时间间隔（如 timedelta）                                     |
| `Enum(...)`                 | 枚举类型，例如 `Enum('A', 'B', 'C')`                         |
| `LargeBinary`               | 二进制数据（如文件、图像等）                                 |

| 字段类型           | 说明                                          |
| ------------------ | --------------------------------------------- |
| `JSON`             | 存储 JSON 数据（PostgreSQL、MySQL 5.7+ 支持） |
| `ARRAY(item_type)` | 数组类型（PostgreSQL 支持）                   |
| `UUID`             | 存储 UUID（部分数据库支持）                   |

5.序列化类型的创建

```python
from marshmallow_sqlalchemy import SQLAlchemySchema,auto_field
class ProjectSchema(SQLAlchemySchema):
    class Meta:
        from apps.system.db_model import Project
        model = Project
        load_instance = False
    id = auto_field(dump_only=True)
    name = auto_field(dump_only=True)
    project_file_path = auto_field(dump_only=True)
    file_size = auto_field(dump_only=True)
    create_time = auto_field(format="%Y-%m-%d %H:%M:%S", dump_only=True)
    update_time = auto_field(format="%Y-%m-%d %H:%M:%S", dump_only=True)
     
```

如何模型表中的字段名为name,而序列化想要修改这个名字为new_name，可以如下定义字段：

```python
from marshmallow import fields
new_name = fields.String(attribute="name")
```

如果向自定义一个表中不存在的字段，可以使用marshmallow.fields.Method 字段来通过一个方法返回序列化字段内容

```python
# 自定义一个不在数据库中的虚拟字段
is_admin_str = fields.Method("get_is_admin_str")

def get_is_admin_str(self, obj):
    return "是" if obj.is_admin else "否"
```



6.使用序列化类实现序列化和反序列化

- 序列化

```python
schema = ProjectSchema()
schema.dump(instance) #序列化单条数据
schema.dump(query_list,many=True)#序列化多条数据
```

- 反序列化

```python
instance = schema.load(instance_data, session=db.session)
db.session.add(instance)
db.session.commit()
```

7.创建一个Method_View类

```python
def filter_params(f):
    """
    支持自动过滤get请求的参数完成条件查询,自动实现分页查询
    :param f:
    :return:
    """
    @wraps(f)
    def wrapper(*args, **kwargs):

        self = args[0]
        model = getattr(self,"model")
        blur_search_keys = getattr(self,"blur_search_keys")

        filters = {}
        query = model.query

        # 处理常见过滤参数
        for param in request.args:
            if hasattr(model, param):
                value = request.args.get(param)
                if value is not None:
                    if param in blur_search_keys:
                        query = query.filter(getattr(model, param).like(f"%{value}%"))
                    else:
                        filters[param] = value
        # 应用过滤
        if filters:
            query = query.filter_by(**filters)

        page = int(request.args.get("page",0))
        if page>0:#如果设置了page_size
            page_size = int(request.args.get("page_size",PAGE_SIZE))
            total = query.count()
            items = query.offset((page - 1) * page_size).limit(page_size).all()
            pagination = {
               "total": total,
               "list":items,
            }
            kwargs["pagination"] = pagination
        kwargs['query'] = query
        return f(*args, **kwargs)

    return wrapper



class CustomMethodView(MethodView):
    """
    自定义MethodView
    """
    serializer_schema_object = None
    model = None
    allowed_methods = ('GET', 'POST', 'PUT', 'DELETE')
    blur_search_keys = []

    def get_serializer_data(self,data ,many=False):
        serializer_data = self.serializer_schema_object.dump(data,many=many)
        return serializer_data

    @filter_params
    def get(self, id=None, **kwargs):
        if request.method not in self.allowed_methods:
            return ErrorResponse(msg="GET Method is not allowed")
        if self.serializer_schema_object is None:
            raise BaseAPIException("serializer_schema_object can not be None")
        pagination = kwargs.get('pagination')
        if pagination:
            pagination['list'] = self.get_serializer_data(pagination['list'],many=True)
            return DetailResponse(msg="ok", data=pagination)

        query = kwargs.get('query')
        if query is None:
            if id:
                instance = self.model.query.get(id)
                serializer_data = self.serializer_schema_object.dump(instance)
                return DetailResponse(msg='ok', data=serializer_data)
            else:
                instance_all = self.model.query.all()
                serializer_data = self.serializer_schema_object.dump(instance_all, many=True)
                return DetailResponse(msg='ok', data=serializer_data)
        else:
            instance = query.all()
            serializer_data = self.get_serializer_data(instance, many=True)
            return DetailResponse(msg='ok', data=serializer_data)

    def post(self):
        """
        新增
        :return:
        """
        if request.method  not in self.allowed_methods :
            return ErrorResponse(msg="POST Method is not allowed")

        data = request.get_json()
        serializer_instance = self.perform_create(data)
        return self.get(id=serializer_instance.id)

    def perform_create(self, data):
        """
        创建模型对象具体方法
        :param data: 前端传递的json数据
        :return: 创建的模型实例对象
        """
        instance = self.serializer_schema_object.load(data,session=db.session)
        db.session.add(instance)
        db.session.commit()
        return instance

    def put(self,id):
        """
        默认的更新方法
        如何存多对一或者多对多的更新，需要实现perform_update方法，在该方法中添加新增的方法
        如何仅仅是单个模型对象的更新，不需要实现perform_update方法，默认读取json中的所有数据对象更新到对应的模型对象中。
        :param id:
        :return:
        """
        if request.method  not in self.allowed_methods :
            return ErrorResponse(msg="PUT Method is not allowed")

        instance = self.model.query.get(id)

        try:
            data = request.get_json()
            instance_update = self.perform_update(instance,data)
            return self.get(id = instance_update.id)
        except Exception as e:
            return ErrorResponse("更新失败".format(e))

    def perform_update(self,instance,data):
        """
        更新方法
        :param instance:
        :param data:
        :return:
        """
        for key, value in data.items():
            setattr(instance, key, value)
            db.session.add(instance)
        db.session.commit()
        return instance

    def delete(self, id):
        """
        删除
        如果存在需要删除关联对象，请实现perform_delete手动删除
        如果不存在关联对象。会自定删除
        :param id:
        :return:
        """
        if request.method  not in self.allowed_methods :
            return ErrorResponse(msg="DELETE Method is not allowed")

        try:
            instance = self.model.query.get(id)
            if hasattr(self,'perform_delete'):
                getattr(self,'perform_delete')(instance)
            else:
                db.session.delete(instance)
                db.session.commit()
            return DetailResponse(msg='ok')
        except Exception as e:
            db.session.rollback()
            return ErrorResponse(msg=str(e))


    def dispatch_request(self, **kwargs: t.Any) -> ft.ResponseReturnValue:
        try:
            return super().dispatch_request(**kwargs)
        except Exception as e:
            response = self.handle_exception(e)
            return response

    def handle_exception(self, exc):
        """
        通用异常处理器，如果定义异常自定义错误前端提示可以在此处进行定义。
        :param exc:
        :return:
        """

        return ErrorResponse(msg = str(exc))
```

