## Python 版本管理

`pyenv` 是一个 Python 版本管理工具，用于在同一台机器上安装和切换不同版本的 Python。不会修改或影响系统的默认 Python，安装的版本都与系统独立。

### 安装

```shell
# macOS
brew install pyenv

# linux
curl https://pyenv.run | bash
# 或手动安装：git clone https://github.com/pyenv/pyenv.git ~/.pyenv

# Windows（通过 WSL）： 在 WSL 中按照 Linux 安装步骤操作
```

### 环境变量配置

安装完成后，需将 `pyenv` 添加到 shell 的启动脚本中（`~/.zshrc`）

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"

# 运行
source ~/.zshrc

# 验证
pyenv --version
```

### 使用

```sh
# 列出可安装的 Python 版本
pyenv install --list

# 安装指定版本的 Python
pyenv install 3.10.9

# 设置全局 Python 版本
pyenv global 3.10.9

# 为某个项目设置 Python 版本（在项目目录中运行）
pyenv local 3.9.7

# 查看当前正在使用的 Python 版本
pyenv version

# 卸载某个 Python 版本
pyenv uninstall 3.10.9
```



## 包管理工具

`pip` 是 Python 官方推荐的包管理工具，全称是 **Pip Installs Packages**。它用于安装、升级、卸载和管理 Python 包。

### 常见的 `pip` 命令

```sh
# 安装 Python 包
pip install requests

# 升级已安装的包
pip install --upgrade requests

# 安装特定版本的包
pip install requests==2.28.1
# 安装大于某个版本的包
pip install "requests>=2.0.0"

# 配置镜像源
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple requests
# 或永久配置镜像源： 在 ~/.pip/pip.conf 或 C:\Users\<用户名>\pip\pip.ini 中添加：
# [global]
# index-url = https://pypi.tuna.tsinghua.edu.cn/simple

# 将包写入需求文件
pip freeze > requirements.txt

# 从需求文件安装包
pip install -r requirements.txt

# 卸载包
pip uninstall requests

# 列出已安装的包
pip list

# 检查包是否有更新
pip list --outdated

# 查看包的详细信息（获取某个包的安装路径、版本等信息）
pip show requests
```

| 命令                              | 作用                                   |
| --------------------------------- | -------------------------------------- |
| `pip install <包名>`              | 安装指定包                             |
| `pip uninstall <包名>`            | 卸载指定包                             |
| `pip list`                        | 列出所有已安装的包                     |
| `pip show <包名>`                 | 查看包的详细信息                       |
| `pip freeze`                      | 列出当前环境中安装的包（生成依赖文件） |
| `pip install -r requirements.txt` | 从需求文件安装包                       |
| `pip list --outdated`             | 查看所有可更新的包                     |
| `pip install --upgrade pip`       | 升级 `pip` 自身                        |

执行 `pip install` 命令后，下载的 Python 包会被安装到当前 Python 环境的 **`site-packages`** 目录中

### 安装目录对比

| 环境类型                | 安装路径                                          | 示例路径                                                   |
| ----------------------- | ------------------------------------------------- | ---------------------------------------------------------- |
| **系统默认 Python**     | 系统的 `site-packages` 目录                       | `/usr/lib/python3.8/site-packages/`                        |
| **pyenv 安装的 Python** | `pyenv` 管理的 Python 版本的 `site-packages` 目录 | `~/.pyenv/versions/<version>/lib/python3.9/site-packages/` |
| **virtualenv 虚拟环境** | 虚拟环境的 `site-packages` 目录                   | `<your-virtualenv-directory>/lib/python3.9/site-packages/` |

### 查看安装路径

`python -m site` 该命令会显示当前 Python 环境的 `site-packages` 路径，其中就包含了虚拟环境的路径

无论使用的是哪种环境，都可以通过以下 Python 代码来查看某个已安装包的具体路径，如想查看 `requests` 包的安装路径，可以运行：

```python
import requests
print(requests.__file__)
```

这将返回 `requests` 包在当前 Python 环境中的路径

### 注意事项

如果使用虚拟环境（`venv` 或 `virtualenv`），建议在激活环境后再使用 `pip`，这样可以避免污染全局环境



## 虚拟环境

`virtualenv`：创建虚拟环境，用于隔离项目依赖的 Python 包（类似 `venv`，但功能更强大）。当需要为每个项目创建一个独立的环境，避免依赖冲突，每个虚拟环境有自己的 `site-packages` 目录，可以单独安装和管理包

```sh
pip install virtualenv

# 在当前目录下创建myenv的虚拟环境
virtualenv myenv

# 激活环境 (Linux/MacOS)
source myenv/bin/activate  
# 激活环境 (Windows)
myenv\Scripts\activate     
```

`virtualenvwrapper` 是一个 Python 包，用于扩展和增强 `virtualenv` 的功能，提供了一些命令来简化虚拟环境的创建、激活、删除等操作：

- 提供了一些易于记住的命令（例如 `mkvirtualenv`、`rmvirtualenv` 等），比直接使用 `virtualenv` 更加简洁
- 它将所有虚拟环境存储在一个特定的目录下，避免了将虚拟环境与项目目录混在一起。可以通过设置环境变量来指定存储虚拟环境的路径（默认是 `~/.virtualenvs`）
- 方便切换和激活虚拟环境：提供了一个 `workon` 命令，允许快速切换到已有的虚拟环境，而不必每次都去指定虚拟环境路径。

### `virtualenvwrapper` 主要命令

| 命令                      | 作用                                           |
| ------------------------- | ---------------------------------------------- |
| `mkvirtualenv <env_name>` | 创建新的虚拟环境，并激活它。                   |
| `rmvirtualenv <env_name>` | 删除指定的虚拟环境。                           |
| `workon <env_name>`       | 切换到并激活指定的虚拟环境。                   |
| `deactivate`              | 退出当前虚拟环境，回到全局环境或终端默认环境。 |
| `lsvirtualenv`            | 列出所有已创建的虚拟环境。                     |
| `cdvirtualenv`            | 进入指定虚拟环境的目录。                       |

### `virtualenvwrapper` 安装

```sh
# 首先，确保已经安装了 virtualenv
pip install virtualenv

# 安装 virtualenvwrapper
pip install virtualenvwrapper

# 在 shell 配置文件中（例如 .bashrc 或 .zshrc）添加以下配置
export WORKON_HOME=$HOME/.virtualenvs
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3  # 设置 Python 解释器路径（根据你的系统调整）
source $(which virtualenvwrapper.sh)
```



## 项目搭建

### 1. 创建项目目录

首先创建一个合适的项目目录结构：

```
my_python_project/
├── app/                    # 主程序目录（或者可以命名为 src/）
│   ├── __init__.py         # 初始化文件（标记该目录为包）
│   ├── main.py             # 启动脚本，入口文件
│   ├── models/             # 数据模型
│   │   ├── __init__.py
│   │   └── user.py         # 用户相关的模型
│   ├── routes/             # 路由/控制器
│   │   ├── __init__.py
│   │   └── auth.py         # 用户认证相关路由
│   └── services/           # 业务逻辑
│       ├── __init__.py
│       └── user_service.py
├── venv/                   # 虚拟环境（可选，但推荐）
├── config/                 # 配置文件目录
|   └── config.py           # 配置文件（可以是 YAML、JSON 或 Python 文件）
├── requirements.txt        # 项目依赖包清单
└── .gitignore
```

### 2. 创建虚拟环境

在 Python 中，使用虚拟环境可以将项目的依赖隔离开，避免与全局 Python 环境发生冲突。推荐使用 `venv` 或 `virtualenv` 来创建虚拟环境

```sh
# 推荐使用 venv 或 virtualenv 来创建虚拟环境
python -m venv venv

# 激活虚拟环境
source venv/bin/activate
```

### 3. 安装项目依赖

创建一个 `requirements.txt` 文件，记录项目依赖包的名称及版本号：

```
flask==2.0.1
requests==2.25.1
```

添加新的依赖并且更新 `requirements.txt` 文件

```sh
pip install <package_name>
# 将当前环境的所有依赖（包括新安装的）写入 requirements.txt 文件
pip freeze > requirements.txt
```

从`requirements.txt`安装依赖：

```sh
pip install -r requirements.txt
```

### 4. 创建启动文件

Python 项目通常有一个启动脚本，通常使用 `main.py` 文件作为项目的启动文件。例如在一个简单的 Flask 项目中，可以有如下 `main.py` 文件：

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello, Python!"

if __name__ == '__main__':
    app.run(debug=True)
```

### 5. 配置文件管理

Python 项目通常会有一些配置项，比如数据库连接、第三方 API 密钥、环境设置等。常见的做法是将这些配置放在单独的文件中

可以使用 `.env` 文件（配合 `python-dotenv` 库）或者将配置项写在 Python 文件中。例如，使用 `.env` 文件：

```makefile
FLASK_APP=my_app
FLASK_ENV=development
SECRET_KEY=your_secret_key
```

然后在 Python 项目中加载这个配置：

```python
from dotenv import load_dotenv
import os

load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY")
```

### 6. 日志管理

Python 项目通常使用 `logging` 模块来管理日志。可以根据需要配置日志级别和日志输出方式（例如输出到文件、控制台等）：

```python
import logging

logging.basicConfig(level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

logger.debug('Debug message')
logger.info('Info message')
```

### 7. 启动与运行

```sh
# 通常是在命令行中运行主程序
python app/main.py
```

### 8. 版本控制

```sh
git init

# 配置 .gitignore 来忽略一些不需要版本控制的文件
venv/
__pycache__/
*.pyc
*.pyo
```



## 特殊说明

### `__init__.py` 文件

`__init__.py` 是一个非常重要的 Python 文件，它主要用于标记一个目录为 Python 包，从而使得该目录中的模块可以被导入。在 Python 中，**包**是一个包含多个模块的目录，而 **模块** 是包含 Python 代码的单个文件。`__init__.py` 让 Python 识别目录为包，而不仅仅是一个普通的文件夹。

```
my_project/
    my_package/
        __init__.py
        module1.py
        module2.py
```

如果包没有初始化代码或其他特殊需求，`__init__.py` 文件也可以是一个空文件，只要存在，它就会让 Python 识别目录为包

`__init__.py` 文件不仅仅是一个空文件，它可以包含一些初始化代码。每当包被导入时，`__init__.py` 文件中的代码都会被执行。这允许在包导入时执行一些设置操作，例如：

- 初始化包级别的变量
- 设置日志记录
- 导入包中的子模块
- 包装子模块的功能

```python
# __init__.py
import logging

# 设置版本号
__version__ = '1.0.0'

# 初始化日志
logging.basicConfig(level=logging.DEBUG)

# 导入包中的子模块
# 包内相对导入,而不是使用绝对路径 from my_package.module1 import func1
from .module1 import func1
from .module2 import func2
```

这样，当导入 `my_package` 时，它不仅执行了初始化代码，还将 `module1` 和 `module2` 中的特定功能暴露给外部使用。

```python
# 这样，用户在导入 my_package 时，可以直接使用 func1 和 func2，而不需要单独导入每个模块
from my_package import func1, func2

# 引用包的版本号
import my_package
print(my_package.__version__)
```

从 Python 3.3 开始，`__init__.py` 文件不是强制要求的。没有 `__init__.py` 文件的目录仍然可以作为包被导入。然而，仍然强烈建议使用 `__init__.py` 文件，因为它能保证兼容性、清晰的包结构，并支持某些特性，如初始化代码、版本控制等

### python文件运行

在命令行中，使用 `python` 或 `python3` 命令来运行 Python 文件：

```sh
python your_file.py
```

这样会执行 `your_file.py` 中的所有代码，从文件的顶部开始依次执行。如果文件中有 `if __name__ == "__main__":` 语句，程序会从这个位置开始执行。

```python
import sys

def my_function():
    print("Hello from my_function!")

def another_function():
    print("Hello from another_function!")

# 只在直接运行文件时才会执行
if __name__ == "__main__":
    # 根据命令行传入的参数决定执行哪个函数
    if len(sys.argv) > 1:
        if sys.argv[1] == "my_function":
            my_function()
        elif sys.argv[1] == "another_function":
            another_function()
        else:
            print("Unknown function!")
    else:
        print("No function specified!")
```

此时，可以通过命令行传入一个参数来指定执行哪个方法

```
python your_file.py my_function
```

如果在 Python REPL（交互式环境）中运行，并且希望执行某个方法，可以通过导入并调用：

```python
>>> import your_file
>>> your_file.my_function()
```

### jupyter

Jupyter 是一个非常强大的工具，它提供了一个交互式的 Web 环境，可以运行 Python 代码并即时显示结果，包括图表、图像等。Jupyter Lab 是 Jupyter Notebook 的升级版本，提供了更多的功能和更好的界面。

```sh
# 安装 pip install notebook
pip install jupyterlab

# 运行 jupyter notebook
jupyter lab

# 运行命令后，Jupyter 会在你的浏览器中打开一个新页面，通常是 http://localhost:8888。这就是 Jupyter 的主界面
# 可以创建和执行代码单元格，或者使用 Markdown 单元格来写注释和说明
# 可以保存 .ipynb 文件并通过多种方式分享或导出
```

常用快捷键：

- 运行当前单元格并跳到下一个：`Shift + Enter`
- 运行当前单元格但不跳到下一个：`Ctrl + Enter`
- 插入新单元格：`Esc + A`（在当前单元格上方插入），`Esc + B`（在当前单元格下方插入）
- 切换到 Markdown 单元格：`Esc + M`
- 切换到代码单元格：`Esc + Y`
- 删除单元格：`Esc + D + D`（按两次 D）

**在 JupyterLab 中创建的 `.ipynb` 文件默认存储在启动 JupyterLab 时所在的工作目录下**，这个目录就是 JupyterLab 启动时的当前工作目录

使用 `--notebook-dir` 参数启动 JupyterLab，可以指定一个目录作为工作目录：

```sh
jupyter lab --notebook-dir=/path/to/your/folder
```

#### 关联虚拟环境

**在虚拟环境中**，安装 `ipykernel` 使其能够作为 Jupyter 内核使用（`ipykernel` 是 Jupyter 用来与 Python 环境交互的内核，安装它可以让 Jupyter 识别并使用你的虚拟环境作为一个内核）：

```sh
pip install ipykernel
```

要让虚拟环境在 Jupyter 中可用，在**虚拟环境中**运行以下命令，将该环境作为一个内核注册到 Jupyter 中：

```sh
python -m ipykernel install --user --name=myenv --display-name "Python (myenv)"

# --name=myenv：这是内核的名称，可以随意命名，但通常使用虚拟环境的名称
# --display-name "Python (myenv)"：这是你在 Jupyter 中看到的内核名称
```

注册成功后，你可以运行以下命令查看当前虚拟环境是否成功添加为 Jupyter 内核：

```sh
jupyter kernelspec list

# 删除指定内核
jupyter kernelspec remove myenv
```

在 JupyterLab 中：点击右上角的内核选择框，选择 **Python (myenv)**

在 Notebook 中运行以下代码来检查当前 Python 环境，确认是否为虚拟环境中的python：

```python
import sys
print(sys.executable)
```

这将显示 Python 解释器的路径。确保你使用的环境与安装包时的环境一致

当虚拟环境成功关联到 Jupyter，就可以在 Notebook 中使用该虚拟环境中安装的所有包。如果创建了多个虚拟环境并希望在 Jupyter 中切换，重复上面的步骤：在每个虚拟环境中安装 `ipykernel` 并将它注册为内核，然后在 Notebook 中选择相应的内核

#### Jupyter 中直接安装包

可以在 Notebook 的代码单元格中直接运行系统命令来安装包。前缀 `!` 表示执行系统命令：

```sh
# 使用 pip 安装包
!pip install package_name
```



