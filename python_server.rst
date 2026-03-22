.. _python-server:

Python 快速搭建简易服务器
=========================

本文档介绍如何使用 Python 快速搭建简易的 HTTP 服务器，适用于开发和测试场景。

Python 3 内置服务器
--------------------

Python 3 提供了内置的 HTTP 服务器模块，可以快速启动一个简单的静态文件服务器。

基本用法
~~~~~~~~

- 在当前目录启动服务器（默认端口 8000）：
  .. code-block:: bash

     python3 -m http.server

- 指定端口启动服务器：
  .. code-block:: bash

     python3 -m http.server 8080

- 指定目录启动服务器：
  .. code-block:: bash

     python3 -m http.server --directory /path/to/directory

- 后台运行服务器：
  .. code-block:: bash

     python3 -m http.server 8000 &

- 查看服务器进程：
  .. code-block:: bash

     ps aux | grep http.server

- 停止服务器：
  .. code-block:: bash

     kill <进程ID>

Python 2 内置服务器
--------------------

如果使用 Python 2，可以使用以下命令启动服务器：

.. code-block:: bash

   python -m SimpleHTTPServer 8000

使用 Flask 搭建简单 API 服务器
------------------------------

对于需要处理动态内容的场景，可以使用 Flask 框架搭建简单的 API 服务器。

安装 Flask
~~~~~~~~~~

.. code-block:: bash

   pip install flask

创建简单 API 服务器
~~~~~~~~~~~~~~~~~~~

创建 `server.py` 文件：

.. code-block:: python

   from flask import Flask, jsonify, request

   app = Flask(__name__)

   # 简单的 GET 接口
   @app.route('/api/hello', methods=['GET'])
   def hello():
       return jsonify({'message': 'Hello, World!'})

   # 带参数的 GET 接口
   @app.route('/api/greet/<name>', methods=['GET'])
   def greet(name):
       return jsonify({'message': f'Hello, {name}!'})

   # POST 接口
   @app.route('/api/data', methods=['POST'])
   def receive_data():
       data = request.json
       return jsonify({'received': data})

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000, debug=True)

启动服务器：
.. code-block:: bash

   python server.py

测试 API：
.. code-block:: bash

   # 测试 GET 接口
   curl http://localhost:5000/api/hello
   curl http://localhost:5000/api/greet/John

   # 测试 POST 接口
   curl -X POST http://localhost:5000/api/data -H "Content-Type: application/json" -d '{"name": "John", "age": 30}'

使用 FastAPI 搭建高性能 API 服务器
----------------------------------

FastAPI 是一个现代、快速的 Web 框架，提供自动 API 文档生成等特性。

安装 FastAPI 和 Uvicorn
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   pip install fastapi uvicorn

创建 FastAPI 服务器
~~~~~~~~~~~~~~~~~~~

创建 `fastapi_server.py` 文件：

.. code-block:: python

   from fastapi import FastAPI
   from pydantic import BaseModel

   app = FastAPI()

   class Item(BaseModel):
       name: str
       price: float
       is_offer: bool = None

   @app.get('/')
   def read_root():
       return {'Hello': 'World'}

   @app.get('/items/{item_id}')
   def read_item(item_id: int, q: str = None):
       return {'item_id': item_id, 'q': q}

   @app.post('/items/')
   def create_item(item: Item):
       return item

启动服务器：
.. code-block:: bash

   uvicorn fastapi_server:app --host 0.0.0.0 --port 8000 --reload

访问 API 文档：
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

使用场景
--------

- **静态文件服务**：快速共享本地文件
- **前端开发**：为前端项目提供静态文件服务
- **API 测试**：快速搭建临时 API 服务器进行测试
- **数据共享**：在局域网内共享文件

注意事项
--------

- 内置服务器仅适用于开发和测试场景，不建议在生产环境使用
- 生产环境应使用专业的 Web 服务器，如 Nginx、Apache 等
- 注意服务器的安全设置，避免暴露敏感文件

更多信息
--------

- `Python http.server 文档 <https://docs.python.org/3/library/http.server.html>`_
- `Flask 官方文档 <https://flask.palletsprojects.com/>`_
- `FastAPI 官方文档 <https://fastapi.tiangolo.com/>`_
