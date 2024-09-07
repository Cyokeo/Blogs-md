---
title: Pytest-简单上手
categories: 嵌入式-开发
---
## 简介
Pytest是基于Python的一套测试框架，`venv`虚拟环境配置好/并激活后，安装好Pytest时，可以直接命令行执行pytest指令

## 参考文档
-  [Pytest测试框架基础及进阶](https://www.cnblogs.com/superhin/p/17755515.html "发布于 2023-10-10 19:16")


## 测试执行
- 默认识别以`test_`开头的函数和类，进行执行

## @pytest.fixture
- 一般用于修饰函数，而且修饰的函数一般集中存放，供其他模块复用
- 其修饰的函数可以作为其他模块函数定义时的参数；修饰函数的返回值作为实际的参数传递给定义的函数
- @pytest.fixture(autouse=True)
	- 平常写自动化用例会有一些前置的fixture操作，用例需要用到就直接将该函数的作为参数传递即可。但当用例很多的时候，会比较麻烦。可以使用该参数，这样用例(修饰的函数)就会被自动调用
- 调用fixture的三种方法
	- 作为函数或类的参数
	- 使用装饰器`@pytest.mark.usefixture()`
	- autouse=True

## @pytest.mark.(user_defined_mark)
- 用于修饰测试函数，只有命令执行时使用`-m`指定的用例才会执行
- 配合自定义的pytest插件，实现一些个性化的测试用例过滤策略
- 一个测试函数可能会有多个mark
	- 只要有一个mark被指定，该测试项就会被选中执行
	- 但是也需要服从自定义mark过滤规则

## 插件开发

### 插件加载
- `conftest.py`文件存在时，加载其`pytest_plugins`变量声明的插件：按照声明的顺序加载
