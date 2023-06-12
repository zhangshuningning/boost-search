# boost-search
@[TOC](目录)
## 宏观原理
![在这里插入图片描述](https://img-blog.csdnimg.cn/0c9d04a7704e4997883ce11dae761da0.png)
* 当我们使用搜索引擎搜索内容时，每条搜索结果一般都是由标题(title)，摘要(description)，网址(url)组成的，这也是本搜索引擎显示结果的模式。
* 项目实现的搜索引擎中的数据(data)来源于上图不同，不来源于全网搜集，而是通过在Boost官网下载网页数据得到。
## 技术栈和项目环境
* 技术栈：C/C++, C++11, STL, 转标准库Boost, Jsoncpp, cppjieba, cpp-httplib, html5 css, js, jQuery, Ajax
* 项目环境：Centos7云服务器, vim/gcc(g++)/Makefile, vscode
## 具体原理（正排&倒排索引）
**正排索引**：从文档ID找到文档内容
| 文档id | 文档内容             |
| ------ | -------------------- |
| 1      | 乔布斯买了两斤苹果   |
| 2      | 乔布斯发布了苹果手机 |

**目标文档分词**（目的：方便建立倒排索引和查找）
* 乔布斯/买/两斤/苹果/两斤苹果
* 乔布斯/发布/苹果/苹果手机
停止词：了，的，吗，a，the，一般在分词的时候不考虑

**倒排索引**：根据文档内容，分词整理出不重复的关键字，对应联系到文档ID的方案
| 关键字   | 文档ID |
| -------- | ------ |
| 乔布斯   | 1，2   |
| 买       | 1      |
| 两斤     | 1      |
| 苹果     | 1，2   |
| 两斤苹果 | 1      |
| 发布     | 2      |
| 苹果手机 | 2      |

模拟一次查找到过程：
用户输入：苹果->倒排索引中查找->提取出文档ID(1,2)->根据正排索引找到文档内容->title+content(description)+url文档结果进行摘要->构建响应结果
## 
boost官网链接: [https://www.boost.org/](https://www.boost.org/)
进入官网点击download下载最新版文档
![在这里插入图片描述](https://img-blog.csdnimg.cn/5add0007898647e5b67563b421743926.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/7417138666dc4c35bd80d61944aef874.png)
下载完成后进入项目目录，使用命令rz -E把文档传输到云服务器。
![在这里插入图片描述](https://img-blog.csdnimg.cn/e326b3dcd081462391a7359a6e75e1f1.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/eb1c10c94b5e4618be35ab01850ea80d.png)

上传完成后使用`tar xzf boost_1_75_0.tar.gz`命令解压文件![在这里插入图片描述](https://img-blog.csdnimg.cn/bd4ba8d67f724bbfbd9ba14b61b3cac2.png)
接着使用**mkdir -p data/input** 命令创建目录，使用**cp -rf boost_1_75_0/doc/html/* data/input/** 命令将html目录下的内容全部拷贝到input下。
##  编写数据去标签与数据清洗的模块parser
什么是标签？下面是笔者随意打开的一个html文件，其中被<>括住的内容就是标签。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/6c6dc145699846179a87a350ef8c885c.png)
他们对搜索没有价值，需要去掉这些标签，一般标签是成对出现的。现在返回data目录创建proccessd_html目录用来储存去完标签的数据，我们的目标是把每个文档都去标签，写入同一个文件中，并且每个文档只占一行。各文档之间用\3 进行区分

*[\3]: 控制字符不可显示，因此不会污染内容
### 框架
```cpp
// 此目录指向所有html网页
const std::string src_path = "/home/ssj/Boost/boost-search/data/input";
const std::string proc_path = "/home/ssj/Boost/boost-search/data/output/proc.txt";

typedef struct DocInfo
{
  std::string title; //文档的标题
  std::string content; //文档内容
  std::string url; //该文档在官网中的url
}DocInfo_t;

//const& :输入
//*: 输出
//&: 输入输出
bool EnumFile(const std::string &src_path, std::vector<std::string> *file_list);
bool ParseHtml(std::vector<std::string>& file_list, std::vector<DocInfo_t> *results);
bool SaveHtml(const std::vector<DocInfo_t> &results, const std::string& proc_path);

int main()
{
  std::vector<std::string> file_list;
  // 第一步: 递归式的把每个html文件名及路径保存到files_list中，方便后期进行文件读取
  if (!EnumFile(src_path, &file_list))
  {
    std:cerr << "enum file name error" << std::endl;
    return 1;
  }
  // 第二步: 按照file_list读取每个文件的内容，并进行解析
  std::vector<DocInfo_t> results;
  if (!ParseHtml(file_list, &results))
  {
    std::cerr << "parse html error" << std::endl;
    return 2;
  }
  //第三步：把解析完毕的各个文件内容写入proc_path中，按\3作为每个文档的分隔符
  if (!SaveHtml(results, proc_path))
  {
    std::cerr << "Save error" << std::endl;
    return 3;
  }
  return 0;
}
bool EnumFile(const std::string &src_path, std::vector<std::string> *file_list)
{
  return true;
}
bool ParseHtml(std::vector<std::string>& file_list, std::vector<DocInfo_t> *results)
{
  return true;
}
bool SaveHtml(const std::vector<DocInfo_t> &results, const std::string& proc_path)
{
  return true;
}
```
### 枚举文件名
#### 安装boost开发库
首先我们要安装boost库来方便管理文件系统，输入如下指令进行安装

```bash
sudo yum install -y boost-devel // boost开发库
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/4ba1cb7181c045f1b747c2bab237a951.png)
因为我已经下载过Boost开发库了，因此下面提示我已经安装过
![在这里插入图片描述](https://img-blog.csdnimg.cn/09d7a9eb53b94ecfb37a0d2f52f1d3c5.png)


值得注意的是，目前我们已经下载了两个Boost库，其中第一个是我们用来做站内查找的，第二个则是开发库，两者并不冲突。

#### 代码
**EnumFile**
```cpp
bool EnumFile(const std::string &src_path, std::vector<std::string> *files_list)
{
  namespace fs = boost::filesystem;
  fs::path root_path(src_path); // 创建根目录
  if (!fs::exists(root_path)) // 检查根目录是否存在
  {
    std::cerr << src_path << " not exists" << std::endl;
    return false;
  }
  // 定义一个空的迭代器，用来判断递归结束
  fs::recursive_directory_iterator end;
  // 进行遍历
  for(fs::recursive_directory_iterator iter(root_path); iter != end; iter++)
  {
    // 如果不是普通文件，就跳过
    if (!fs::is_regular_file(*iter))
    {
      continue;
    }
    // 如果文件后缀不是".html"，就跳过
    if (iter->path().extension() != ".html")
    {
      continue;
    }

    // debug代码
    std::cout << "debug: " << iter->path().string() << std::endl;


    // 当前路径一定是合法的，以.html为结尾的普通网页文件
    // string()可以把路径以字符串的形式返回
    files_list->push_back(iter->path().string());
  }

  return true;
}
```
**Makefile**
```
cc=g++
parcer:parcer.cc
	$(cc) -o $@ $^ -lboost_system -lboost_filesystem -std=c++11
.PHONY:clean
clean:
	rm -rf parcer
```

##### 测试结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/91a96de294e74e3a8e22abe05ab874a7.png)
一共有7千多个html文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/10ffdc7981b541898856e01182f21f90.png)
### 解析html结构
接着我们就开始第二步，还记得在代码框架中我们所需每个条目的三个要素吗？
```cpp
typedef struct DocInfo
{
  std::string title; //文档的标题
  std::string content; //文档内容
  std::string url; //该文档在官网中的url
}DocInfo_t;
```
通过遍历每个文件名，依次获得他们的html结构，
```cpp
bool ParseHtml(std::vector<std::string>& files_list, std::vector<DocInfo_t> *results)
{
  for (const std::string& file : files_list)
  {
    // 1. 读取文件Read()
    // 2. 提取title
    // 3. 提取content
    // 4. 解析指定的文件路径，构建url
  }
  return true;
}
```
接下来我们创建一个文件**util.hpp**专门存放工具，这一次我们编写函数来解析html结构：
![在这里插入图片描述](https://img-blog.csdnimg.cn/d92cfb608eb64ab78d1903e996a361c6.png)
随便打开一个html文件看看，其实提取内容在本质上也就是去标签。


#### 读取文件
```cpp
static bool ReadFile(const std::string &file_path, std::string *out)
    {
      std::ifstream in(file_path, std::ios::in);
      if (!in.is_open())
      {
        std::cerr << "open file " << file_path << std::endl;
        return false;
      }

      std:: string line;
      while (std::getline(in, line))
      {
        *out += line;
      }

      in.close();
      return true;
    }
```

#### 提取title
![在这里插入图片描述](https://img-blog.csdnimg.cn/01b9627331774116a37c7ce0a0624355.png)
上图就是html中的title，\<title>和\</title>标签框住，因此可以根据这个特性用C++的find()查找。

```cpp
static bool ParceTitle(const std::string &file, std::string *title)
{
  // 找到title
  std::size_t begin = file.find("<title>");
  if (begin ==std::string::npos)
  {
    return false;
  }
  // 找到结尾
  std::size_t end = file.find("</title>");
  if (end == std::string::npos)
  {
    return false;
  }
  // 开始位置跳过标签
  begin += std::string("<title>").size();
  // 确保begin不大于end
  if (begin > end)
  {
    return false;
  }
  // 提取title
  *title = file.substr(begin, end - begin);
  return true;
}
```

#### 提取内容
```cpp
static bool ParseContent(const std::string &file, std::string *content)
{
  // 去标签，基于一个简易的状态机
  enum status
  {
    LABLE,
    CONTENT
  };

  enum status s = LABLE;
  // 在遍历时，只要碰到了>,就意味着当前的标签处理完毕
  for (char c : file)
  {
    switch (s)
    {
      // 当状态是LABLE的时候什么都不需要处理
      case LABLE:
      if (c == '>')
      {
        s = CONTENT;
      }
      break;
      case CONTENT:
      // 只要碰到<就代表进入标签
      if (c == '<')
      {
        s = LABLE;        
      }
      else
      {
        // 我们不想保留原始文件中的\n，因为他将来会被作为html解析之后文本的分隔符
        if (c == '\n') c = ' ';
        content->push_back(c);
      }
      break;
      default:
      break;
    }
  }

  return true;
}
```

#### 解析URL
这是官网中的一个url ：*https://www.boost.org/doc/libs/1_75_0/doc/html/boost_asio/reference/basic_socket_acceptor/basic_socket_acceptor.html*
这是我放在input目录下的同一个网页的路径：*/home/ssj/Boost/boost-search/data/input/boost_asio/reference/basic_socket_acceptor*
![在这里插入图片描述](https://img-blog.csdnimg.cn/714f56b31c67478884fdebe59b78a458.png)

```cpp
static bool ParseUrl(const std::string &file_path, std::string *url)
{
  std::string url_head = "https://www.boost.org/doc/libs/1_75_0/doc/html";
  std::string url_tail = file_path.substr(src_path.size());

  *url = url_head + url_tail;
  return true;
}
```
**EnumFile**
```cpp
bool ParseHtml(std::vector<std::string>& files_list, std::vector<DocInfo_t> *results)
{
  for (const std::string& file : files_list)
  {
    // 1. 读取文件,Read()
    std::string result;
    // 如果无法找到文件，跳过
    if (!ns_util::FileUtil::ReadFile(file, &result))
    {
      continue;
    }
    DocInfo_t doc;
    // 2. 提取title
    if (!ParceTitle(result, &doc.title))
    {
      continue;
    }
    // 3. 提取content,本质是去标签
    if (!ParseContent(result, &doc.content))
    {
      continue;
    }

    // 4. 解析指定的文件路径，构建url
    if (!ParseUrl(file, &doc.url))
    {
      continue;
    }
    // 完成解析任务，当前文档的相关结果都保存在doc
    results->push_back(std::move(doc));

    // Debug
    //ShowDoc(doc);
  }
  return true;
}
```

**测试**
添加如下代码检查下结果是否正确
```cpp
void ShowDoc(const DocInfo_t &doc)
{
  std::cout << "title: " << doc.title << std::endl;
  std::cout << "content: " << doc.content << std::endl;
  std::cout << "url: " << doc.url << std::endl;
}

bool ParseHtml(std::vector<std::string>& files_list, std::vector<DocInfo_t> *results)
{
  for (const std::string& file : files_list)
  {
    // 1. 读取文件,Read()
    std::string result;
    // 如果无法找到文件，跳过
    if (!ns_util::FileUtil::ReadFile(file, &result))
    {
      continue;
    }
    DocInfo_t doc;
    // 2. 提取title
    if (!ParceTitle(result, &doc.title))
    {
      continue;
    }
    // 3. 提取content,本质是去标签
    if (!ParseContent(result, &doc.content))
    {
      continue;
    }

    // 4. 解析指定的文件路径，构建url
    if (!ParseUrl(file, &doc.url))
    {
      continue;
    }
    // 完成解析任务，当前文档的相关结果都保存在doc
        results->push_back(std::move(doc));
    // 添加此处 Debug
    ShowDoc(doc);
  }
  return true;
}
```
编译代码，这里会得到非常多的数据直接`Ctrl + c`即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/0c3616cfcedb4790ab785855a97eb804.png)
这是其中一条结果，可以看到content中已经没有标签了，而且用url也可以正常登录网页
![在这里插入图片描述](https://img-blog.csdnimg.cn/d85b20018b3d4af1b25f60101e3ed7a2.png)
打开网页可以看到标题跟处理结果也是能对上的

#### 写入文件
下面开始parcer.cc中的最后一步：SaveHtml()
在写入文件时需要考虑写入方便，也要考虑读取方便，在本项目中采用的是这种写入形式：
`title1\3content1\3url1\ntitle2\3content2\3url2\n`
这样我们可以通过getline(ifstream, line)每次读取一个网页的全部信息，并通过\3区分不同的元素。
```cpp
bool SaveHtml(const std::vector<DocInfo_t> &results, const std::string& proc_path)
{
  // 按照二进制方式写入
  std::ofstream out(proc_path, std::ios::out | std::ios::binary);
  if (!out.is_open())
  {
    std::cerr << "open " << proc_path << " failed" << std::endl;
    return false;
  }

  // 就可以进行文件内容写入
  for (auto &item : results)
  {
#define SEP '\3'
    std::string out_string;
    out_string = item.title;
    out_string += SEP;
    out_string += item.content;
    out_string += SEP;
    out_string += item.url;
    out_string += SEP;
    out_string += '\n';
    out.write(out_string.c_str(), out_string.size());
  }


  out.close();
  return true;
}
```

### 索引
再创建一个文件:index.hpp，用来编写索引功能的代码。

#### 框架
```cpp
namespace ns_index
{
    // 存储文档信息
    struct DocInfo
    {
        std::string title;
        std::string content;
        std::string url;
        int doc_id; // 文档的id
    };

	// 倒排索引元素
    struct InvertedElem
    {
        uint64_t doc_id;
        std::string word; //关键字
        int weight; //权重
    };

    // 倒排拉链
    typedef std::vector<InvertedElem> InvertedList;
    // 索引类
    class Index
    {
    private:
        // 正排索引的数据结构用数组，数组的下标是文档的id
        std::vector<DocInfo> forward_index; // 正排索引
        // 倒排索引一定是一个关键字和一组InvetedElem对应[关键字和倒排拉链的映射关系]
        std::unordered_map<std::string, InvertedList> inverted_index;

    public:
        forward_Index(){}
        Inverted_Index(){}
    public:
        // 根据doc_id找到文档内容
        DocInfo *GetForwardIndex(const uint64_t &doc_id)
        {
            if (doc_id >= forward_index.size())
            {
                std::cerr << "doc_id out range" << std::endl;
                return nullptr;
            }
            return forward_index[doc_id];
        }
        // 根据关键字string,获得倒排拉链
        InvertedList *GetInvertedList(const std::string &word)
        {
            auto iter = invered_index.find(word);
            if (iter == inverted.end())
            {
                std::cerr << word << " has no InvertedList" << std::endl;
                return nullptr;
            }
            return &iter->second;
        }

        // 根据文档去标签，格式化之后的文档，构建正派和倒排索引
        bool BuildIndex(const std::string &input) // parse处理完毕的数据
        {
            std::ifstream in(input, std::ios::in | std::ios::binary);
            if (!in.isopen())
            {
                std::cerr << "sorry, " << input << " open error" << std::endl;
                return false;
            }

            std::string line;
            while (std::getline(in, line))
            {
                DocInfo *doc = BuildForwordIndex(line);
                if (nullptr == doc)
                {
                    // debug
                    std::cerr << "build " << line << "error" << std::endl;
                    continue;
                }
                BuildInvertedIndex(*doc);
            }
            return true;
        }
    private:
   		 // 建立正排索引
        DocInfo *BuildForwardIndex(const std::string &line)
        {}
		// 建立倒排索引
        bool BuildInvertedIndex(const DocInfo &doc)
        {}
    };
}
```

#### 构建正排索引
```cpp
        DocInfo *BuildForwardIndex(const std::string &line)
        {
            //1. 解析line，字符串切分
            std::vector<std::string> results;
            std::string sep = "\3"; //行内分隔符
            // 此代码在util.hpp中
            ns_util::StringUtil::CutString(line, &results, sep);
            if (results.size() != 3)
            {
                return nullptr;
            }
            //2. 字符串填充到DocInfo
            DocInfo doc;
            doc.title = results[0];
            doc.content = results[1];
            doc.url = results[2];
            doc.doc_id = forward_index.size();//doc在vector中的下标
            //3. 插入到正排索引的vector
            forward_index.push_back(std::move(doc));
			
			return &forward_index.back();
        }
```
**util.hpp**中的内容
```cpp
class StringUtil
{
  public:
  	// 切分字符串
    static void CutString(const std::string &target, std::vector<std::string> *out, std::string sep)
    {
		// 使用boost库的split()
		// 需要包含头文件<boost/algorithm/string.hpp>
        boost::split(*out, target, boost::is_any_of(sep), boost::token_compress_on);
	}
};

```

#### 构建倒排索引
```cpp
// 原理
// 根据每个文档的内容，形成一个或多个InvertedElem。
struct InvertedElem
{
	uint64_t doc_id;
	std::string word;
	int weight;
};
// 倒排拉链
typedef std::vector<InvertedElem> InvertedList;
//倒排索引一定是一个关键字和一组（个）InvertedElem对应[关键字和倒排拉链的对应关系]
```
1.需要对title和content分词
2.词频统计
3.自定义相关性
```cpp
struct InvertedElem elem;
elem.doc_id = 123;
elem.word = word.first;
elem.weight = 10 * word.second。title_cnt + word.second.content_cnt;// 根据词频统计得出的相关性
inverted_index[word.first].push_back(elem);
```
##### 下载并使用Jieba
为了分词我们需要下载Jiaba，Github链接: [https://github.com/yanyiwu/cppjieba](https://github.com/yanyiwu/cppjieba)，复制仓库http地址，clone到Linux的工作目录即可。
接着为了引头文件更加方便，我们可以把重要的文件建立软链接，放到项目路径中，
```shell
在项目路径下
ln -s ~/cppjieba/include/cppjieba.hpp cppjieba （保存了库文件，包含Jieba.hpp）
ln -s ~/cppjieba/dict dict (词典，保存词条)
```
现在直接开始使用jieba还会出现问题，还得做这一步
```shell
cd cppjieba
cp -rf deps/limonp include/cppjieba
```

---------------------------------------------------------------------------
#### 编写倒排索引代码
```cpp
bool BuildInvertedIndex(const DocInfo &doc)
{
    // DocInfo{title, content, url, doc_id}
    //word->倒排拉链
    struct word_cnt
    {
    	/*储存标题，内容关键词出现用于计算权重*/
        int title_cnt;
        int content_cnt;
        word_cnt():title_cnt(0), content_cnt(0){}
    };
    std::unordered_map<std::string, word_cnt> word_map; // 用来暂存词频的映射表
    // 拆分标题
    std::vector<std::string> title_words;
    ns_util::JiebaUtil::CutString(doc.title, &title_words);
    // 对标题进行词频统计
    for (std::string s : title_words)
    {
        boost::to_lower(s); // 将分词统一转化为小写
        word_map[s].title_cnt++;
    }
    // 拆分内容
    std::vector<std::string> content_words;
    ns_util::JiebaUtil::CutString(doc.content, &content_words);
    // 统计内容
    for (std::string s : content_words)
    {
        boost::to_lower(s); // 将分词统一转化为小写
        word_map[s].content_cnt++;
    }
#define X 10
#define Y 1
    for (auto &word_pair : word_map)
    {
    	/*InvertedElem{doc_id,word,weight}*/
        InvertedElem item;
        item.doc_id = doc.doc_id;
        item.word = word_pair.first;
        item.weight =X * word_pair.second.title_cnt + Y * word_pair.second.content_cnt; // 相关性
        InvertedList &inverted_list = inverted_index[word_pair.first];
        inverted_list.push_back(std::move(item));
    }
    return true;
}
```
**util.hpp**中的内容
```cpp
const char* const DICT_PATH = "./dict/jieba.dict.utf8";                                                                                                                                            
const char* const HMM_PATH = "./dict/hmm_model.utf8";
const char* const USER_DICT_PATH = "./dict/user.dict.utf8";
const char* const IDF_PATH = "./dict/idf.utf8";
const char* const STOP_WORD_PATH = "./dict/stop_words.utf8";

//将该对象设置为单例
class JiebaUtil
  {
  private:
    // static cppjieba::Jieba jieba;
    cppjieba::Jieba jieba;
    std::unordered_map<std::string, bool> stop_words;
    JiebaUtil():jieba(DICT_PATH, HMM_PATH, USER_DICT_PATH, IDF_PATH, STOP_WORD_PATH)
    {}
    JiebaUtil(const JiebaUtil&) = delete;

    static JiebaUtil *instance;
    public:
    static JiebaUtil* get_instance()
    {
      static std::mutex mtx;
      if (nullptr == instance)
      {
        mtx.lock();
        if (nullptr == instance)
        {
          instance = new JiebaUtil();
          instance->InitJiebaUtil();
        }
        mtx.unlock();
      }
      return instance;
    }
    
    void InitJiebaUtil()
    {
      std::ifstream in(STOP_WORD_PATH);
      if (!in.is_open())
      {
        LOG(FATAL, "load stop words file error");
        return;
      }
      std::string line;
      while (std::getline(in, line))
      {
        stop_words.insert({line, true});
      }
    }
    void CutStringHelper(const std::string &src, std::vector<std::string> *out)
    {
      jieba.CutForSearch(src, *out);
      for (auto iter = out->begin(); iter != out->end(); )
      {
        auto it = stop_words.find(*iter);
        if (it != stop_words.end())
        {
          //说明当前的string是暂停词，需要去掉
          iter = out->erase(iter);
        }
        else
        {
          iter++;
        }
      }
    }

  public:
    static void CutString(const std::string &src, std::vector<std::string> *out)
    {
      //jieba.CutForSearch(src, *out);
      ns_util::JiebaUtil::get_instance()->CutStringHelper(src, out);
    }

  };
  //在类外初始化
  JiebaUtil *JiebaUtil::instance = nullptr;
  // cppjieba::Jieba JiebaUtil::jieba(DICT_PATH, HMM_PATH, USER_DICT_PATH, IDF_PATH, STOP_WORD_PATH);
```

搜索不区分大小写

---

### 编写搜索引擎模块searcher
#### 代码框架
```cpp
void Search(const std::string &query, std::string *json_string)
{
    // 1. 分词：对query进行searcher的要求进行
    // 2. 触发：根据分词的各个“词”，进行index查找，建立index是忽略大小写的，所以搜索时也需要忽略大小写
        // 先查倒排，获得倒排拉链
        // 把所有的拉链保存在一起
        // 多个词可能跟一个文档相关，因此文档可能会重复
    // 3. 合并排序：根据汇总查找结果，按照相关性（weight）降序排序
    // 4. 构建：根据查找结果，构建json串 —— 通过jsoncpp完成序列化和反序列化
}
```
为了进行序列化反序列化，需要安装jsoncpp
```shell
sudo yum install -y jsoncpp-devel
```

json的使用
```cpp
#include <iostream>
#include <string>
#include <vector>
#include <jsoncpp/json/json.h>

// Value Reader Writer
int main()
{
	Json::Value root;
	Json::Value item1;
	item1["key1"] = "value1";
	item2["key2"] = "value2";
	Json::Value item2;

	Json::Value item2;
	item2["key1"] = "value1";
	item2["key2"] = "value2";
	
	root.append(item1);
	root.append(item2);

	Json::StyledWriter writer;
	// Json::FastWriter writer;
	std::string s = writer.write(root);
	std::cout << s << std::endl;
	
	return 0;
}
```
#### 分词
```cpp
// 1. 分词：对query进行searcher的要求进行
std::vector<std::string> words;
ns_util::JiebaUtil::CutString(query, &words);
```
#### 触发
```cpp
// 2. 触发：根据分词的各个“词”，进行index查找，建立index是忽略大小写的，所以搜索时也需要忽略大小写
std::unordered_map<uint64_t, InvertedElemPrint> tokens_map;
std::vector<InvertedElemPrint> inverted_list_all;
/*InvertedElemPrint{doc_id,weight,vector<std::string> words}*/
for (std::string word : words)
{
    boost::to_lower(word);
    // 先查倒排，获得倒排拉链
    ns_index::InvertedList *inverted_list = index->GetInvertedList(word);
    if (nullptr == inverted_list)
    {
        continue;
    }
    // 把所有的拉链保存在一起
    // 多个词可能跟一个文档相关，因此文档可能会重复
    
    for (const auto &elem : *inverted_list)
    {
        auto &item = tokens_map[elem.doc_id];
        // item一定是doc_id相同的print节点
        item.doc_id = elem.doc_id;
        item.weight += elem.weight;
        item.words.push_back(elem.word);
    }
}
for (const auto &item : tokens_map)
{
    inverted_list_all.push_back(std::move(item.second));
}
```
#### 合并排序
```cpp
// 3. 合并排序：根据汇总查找结果，按照相关性（weight）降序排序
 std::sort(inverted_list_all.begin(), inverted_list_all.end(),\
  [](const InvertedElemPrint &e1,\
 const InvertedElemPrint &e2){return e1.weight > e2.weight;});
```
#### 构建
```cpp
// 4. 构建：根据查找结果，构建json串 —— 通过jsoncpp完成序列化和反序列化
Json::Value root;
for (auto &item : inverted_list_all)
{
    ns_index::DocInfo *doc = index->GetForwardIndex(item.doc_id);
    if (nullptr == doc)
    {
        continue;
    }
    Json::Value elem;
    elem["title"] = doc->title;
    
    elem["desc"] = GetDesc(doc->content, item.words[0]); // 是文档去标签的结果，但不是摘要
    elem["url"] = doc->url;
    
    root.append(elem);
}
//Json::StyledWriter writer;
Json::FastWriter writer;
*json_string = writer.write(root);
}
```

### 文本摘要
这一步的目的是获得搜索结果中的摘要。获取摘要的逻辑是：找到word在html_content中首次出现，然后往前找50字节(如果没有，从begin开始),往后找100字节(如果没有，到end就可以的)
```cpp
std::string GetDesc(const std::string &html_content, const std::string &word)
{
    // 找到word在html_content中首次出现，然后往前找50字节(如果没有，从begin开始),往后找100字节(如果没有，到end就可以的)
    // 截出这部分内容
    const int prev_step = 50;
    const int next_step = 100;
    // 1. 找到首次出现
    auto iter = std::search(html_content.begin(), html_content.end(), word.begin(), word.end(), [](int x, int y){
        return (std::tolower(x) == std::tolower(y));
    });
    if (iter == html_content.end())
    {
        return "None1";
    }
    int pos = std::distance(html_content.begin(), iter);

    // 2. 获取start, end, std::size_t 无符号整数
    int start = 0;
    int end = html_content.size() - 1;
    // 如果之前有50+字符，就更新开始位置
    if (pos > start + prev_step) start = pos - prev_step;
    if (pos < end - next_step) end = pos + next_step;

    // 3. 截取字串，return
    if (start >= end) return "None2";
    std::string desc = html_content.substr(start, end - start);
    desc += "...";
    return desc;
}
```

###  综合测试
**debug.cc**
```cpp
#include "searcher.hpp"
#include <iostream>
#include <string>
#include <cstdio>
// #include <unistd.h>

const std::string input = "/home/ssj/Boost/boost-search/data/output/raw.txt";

int main()
{
    // for test
    ns_searcher::Searcher *search = new ns_searcher::Searcher();
    //std::cout << "search over--------------" << std::endl;
    //sleep(5);
    search->InitSearcher(input);

   // std::cout << "init over--------------" << std::endl;
   // sleep(5);

    std::string query;
    std::string json_string;
    char buffer[1024];
    while (true)
    {
        std::cout << "Please Enter Your Search Query# ";
        fgets(buffer, sizeof(buffer) - 1, stdin);
        buffer[strlen(buffer) - 1] = 0;
        query = buffer;
        // std::cin >> query;
        search->Search(query, &json_string);
        std::cout << json_string << std::endl;
    }
    return 0;
}
```

### 编写http_server模块
cpp-httplib在使用时需要使用较新版本的gcc，centos 7下默认gcc 4.8.5
![在这里插入图片描述](https://img-blog.csdnimg.cn/f34e8d19a916420983f5d027c6963d87.png)
用老的编译器要么编译不通过，要么运行时出错
#### 升级编译器
搜索centos7 scl gcc devsettool升级gcc
安装scl源
```shell
sudo yum install centos-release-scl scl-utils-build
```
// 安装新版本gcc
```shell
sudo yum install -y devtoolset-7-gcc devtoolset-7-gcc-c++
```
查看工具集
```shell
ls /opt/rh
```
启动
```shell
scl enable devtoolset-7 bash
```
查看gcc版本
```shell
gcc -v
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/0c1ca9417785441cbd1adbbaa8cca135.png)
每次启动只能在本会话有效，如果不想这么麻烦可以每次启动默认启动
```shell
vim ~/.bash_profile
```
```shell
# 在文件底部添加如下命令
# 每次启动的时候都会执行这个scl命令
scl enable devtoolset-7 bash
```
#### 安装cpp-httplib

最新的cpp-httplib在使用的时候，如果gcc不是特别新的话有可能会有运行时出错的问题，建议选择v0.7.15，下载链接: [https://github.com/yhirose/cpp-httplib/tree/v0.7.15](https://github.com/yhirose/cpp-httplib/tree/v0.7.15)
下载好后将zip文件上传至服务器中
```shell
unzip cpp-httplib-0.7.15.zip
rm cpp-httplib-0.7.15.zip
```

#### http_server.cc
```cpp
#include "searcher.hpp"
#include "cpp-httplib/httplib.h"

// const std::string proc_path = "/home/ssj/Boost/boost-search/data/output/raw.txt";
//根目录
const std::string root_path = "./wwwroot";

int main()
{
  ns_searcher::Searcher search;
  //初始化搜索类
  search.InitSearcher(proc_path);

  httplib::Server svr;
  // 设置主页（根目录）
  svr.set_base_dir(root_path.c_str());
  // '/s'访问目录
  svr.Get("/s", [&search](const httplib::Request &req, httplib::Response &rsp)
  {
    if (!req.has_param("word"))
    {
      rsp.set_content("need key words:", "text/plain; charset=utf-8");
      return;
    }
    std::string word = req.get_param_value("word");
    // std::cout << "用户在搜索：" << word << std::endl;
    LOG(NORMAL, "用户在搜索: " + word);
    std::string json_string;
    search.Search(word, &json_string);
    //响应内容
    rsp.set_content(json_string, "application/json");
    // rsp.set_content("hello world!", "text/plain; charset=utf-8");
  });
  LOG(NORMAL, "服务器启动成功...");
  svr.listen("0.0.0.0", 8081);
  return 0;
}
```

### 编写前端模块
html是网页的骨骼: 网页的结构
css是网页的皮肉: 网页的美观
js是网页的灵魂 动态效果: 前后端交互
#### 前端代码
```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="http://code.jquery.com/jquery-2.1.1.min.js"></script>

  <title>Boost 搜索引擎</title>
  <style>
    /* 去掉网页中所有的默认内外边距，html的盒子模型 */
    * {
      /* 设置外边距 */
      margin:0;
      /* 设置内边距 */
      padding:0;
    }
    /* 将body内的内容100%和html的呈现吻合 */
    html,
    body{
      height:100%;
    }
    /* 类选择器 */
    .container {
      /* 设置div的宽度 */
      width: 800px;
      /* 通过设置外边距达到居中对齐的目的 */
      margin: 0px auto;
      /* 设置外边距的上边距，保持元素和网页的上部距离 */
      margin-top:15px;
    }
    /* 复合选择器，选择container下的search */
    .container .search{
      /* 宽度与父标签保持一致 */
      width: 100%;
      /* 高度设置为52px */
      height: 52px;
    }
    /* 先选中input标签，直接设置标签的属性，先要选中，input：标签选择器 */
    .container .search input{
      /* 设置left浮动 */
      float: left;
      width: 600px;
      height: 50px;
      /* 设置边框属性：宽度，样式，颜色 */
      border: 1px solid black;
      /* 去掉input输入框的右边框 */
      border-right: none;
      /* 设置内边距，默认文字不要和左侧边距紧挨着 */
      padding-left: 10px;
      /* 设置文字样式 */
      color: #CCC;
      font-size: 15px;
    }
    /* 先选中button标签，直接设置标签的属性，先要选中，button：标签选择器 */
    .container .search button {
      /* 设置left浮动 */
      float: left;
      width: 120px;
      height: 50px;
      border: 1px #4e6ef2;
      /* 设置button颜色 */
      background-color: #4e6ef2;
      /* 设置button中的字体颜色 */
      color: #FFF;
      font-size: 17px;
      font-family:'Courier New', Courier, monospace;
    }
    .container .results {
      width: 100%;
    }
    .container .results .item {
      margin-top: 15px;
    }
    .container .results .item a {
      /* 设置为块级元素，单独占一行 */
      display: block;
      text-decoration: none;
      /* 设置a标签中的文字字体大小 */
      font-size: 20px;
      /* 设置字体的颜色 */
      color: #0726a1;
    }
    .container .results .item a:hover {
      /* 设置鼠标放在a之上的动态效果 */
      text-decoration: underline;
    }
    .container .results .item p {
      margin-top:5px;
      font-size: 16px;
    }
    .container .results .item i {
      /* 设置为块级元素，单独占一行 */
      display: block;
      /* 取消斜体风格 */
      /* font-style: normal; */
      color: rgb(80, 108, 80);
    }

  </style>
</head>

<body>
  
  <div class="container">
    <div class="search">
      <input type="text" value="输入关键字">
      <button onclick="Search()">搜索</button>
    </div>
    <div class="results">
    </div>
  </div>
  <script>
    function Search()
    {
      // 浏览器的弹出框
      // alert("hello js!");
      // 1.提取数据，$可以看作是JQuery的别称
      // JQuery cdn
      let query = $(".container .search input").val();
      if (query == '' || query == null)
      {
        return;
      }
      console.log("query = " + query); //console是浏览器的对话框，可以查看js的数据

      //2.发起http请求,ajax:属于一个和后端进行数据交互的函数，JQuery中的
      $.ajax({
        type: "GET",
        url: "/s?word="+query,
        success:function(data)
        {
          console.log(data);
          BuildHtml(data);
        }
      });
    }
    function BuildHtml(data)
    {
      if (data == '' || data == null)
      {
        document.write("没有搜索的内容");
        return;
      }
      // 获取html中results标签
      let results_lable = $(".container .results");
      // 清空历史搜索结果
      results_lable.empty();

      for (let elem of data)
      {
        // console.log(elem.title);
        // console.log(elem.url);
        let a_lable = $("<a>", {
          text: elem.title,
          href: elem.url,
          // 跳转到新的页面
          target: "_blank"
        });
        let p_lable = $("<p>", {
          text: elem.desc
        });
        let i_lable = $("<i>", {
          text: elem.url
        });

        let div_lable = $("<div>", {
          class: "item"
        });
        a_lable.appendTo(div_lable);
        p_lable.appendTo(div_lable);
        i_lable.appendTo(div_lable);
        div_lable.appendTo(results_lable);
      }
    }
  </script>
</body>

</html>
```

### 日志
```cpp
#pragma once

#include <iostream>
#include <string>
#include <ctime>

#define NORMAL  1
#define WARNING 2
#define DEBUG   3
#define FATAL   4

#define LOG(LEVEL, MESSAGE) log(#LEVEL, MESSAGE, __FILE__, __LINE__)

void log(std::string level, std::string message, std::string file, int line)
{
    std::cout << "[" << level << "]" << "[" << time(nullptr) << "]" << "[" << message << "]" << "[" << file << " : " << line << "]" << std::endl;
}
```

### 部署服务到linux上
```shell
nohup ./http_server > log/log.txt 2>&1 &
```

## 访问结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/2d8f19982e4e4581b55e3c87aedf1ede.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/0d2e84bf4cea4a469f3128321b27901a.png)
