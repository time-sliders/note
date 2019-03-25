# YAML 基础
YAML是专门用来写配置文件的语言，一种通用的数据串行化格式

YAML语法规则：

* 大小写敏感
* 使用**缩进**表示层级关系
* 缩进时不允许使用Tal键，只允许使用空格
* 缩进的空格数目不重要，只要相同层级的元素**左侧对齐**即可
* \# 表示注释，从这个字符一直到行尾，都会被解析器忽略


## Maps

    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: kubernetes-site
      labels:
        app: web
        
--- 为可选的分隔符，当需要在一个文件中定义多个结构的时候需要使用。  
上述内容表示有两个键apiVersion和kind，分别对应的值为v1和Pod。  
maps的value既能够对应字符串也能够对应一个Maps

YAML处理器根据行缩进来知道内容之间的关联。  
上述例子中，使用**两个空格作为缩进，但空格的数据量并不重要，只是至少要求一个空格并且所有缩进保持一致的空格数** 

例如，name和labels是相同缩进级别，因此YAML处理器知道他们属于同一map；它知道 app 是labels 的值因为 app 的缩进更大

# List

    args
     -beijing
     -shanghai
     -shenzhen
     -guangzhou
     
每个项的定义以破折号（-）开头，并且与父元素之间存在缩进

Lists的子项也可以是Maps，Maps的子项也可以是List