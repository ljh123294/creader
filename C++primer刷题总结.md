

<h1 align = "center">C++primer刷题总结</h1>

## 内置类型

### 转化

```C++
int i = stoi(s);
double d = stod(s);
```

### char

```C++
const char a[] = "aaa";
const char b[] = "bbb";
char *pca = new char[strlen("aaa""bbb") +1];//7 居然可以这么写 ！！！
std::strcat(pca, a);
std::strcat(pca, b);
```



## 类

### 成员初始化	

```c++
class A
{
    int i=0;//非static变量允许类内初始化
    static constexpr int sci=0;//static变量只有加上constexpr才能类内初始化
    static int si=0;//非法
    static int si;
    static void test();
}
int A::si=0;//类外定义时不能重复static关键字
void test(){}
```

### 函数对象类

```C++
#include <iostream>
class DebugDelete
{
public:
    DebugDelete(std::ostream &s = std::cerr) : os(s) {}
    void operator()(int *p) const
    {
        os << "deleting unique_ptr" << std::endl;
        delete p;
    }

private:
    std::ostream &os;
};

int main()
{
    DebugDelete d;
    int *p = new int;
    d(p);

    return 0;
}

```



### 简单的文件管理类

```C++
#ifndef FOLDER_MESSAGE_H
#define FOLDER_MESSAGE_H
#include <string>
#include <set>
class Folder;
class Message
{
    friend class Folder;
    friend void swap(Message &, Message &);

private:
    std::string contents;
    std::set<Folder *> folders;
    void add_to_Folders(const Message &);
    void remove_from_Folders();
    void addFolder(Folder *f) { folders.insert(f); }
    void removeFolder(Folder *f) { folders.erase(f); }
    void move_Folders(Message *);

public:
    explicit Message(const std::string &str = "") : contents(str) {}
    Message(const Message &);
    Message &operator=(const Message &);
    Message &operator=(Message &&);//移动赋值
    Message(Message &&m) : contents(std::move(m.contents))//移动构造
    {
        move_Folders(&m);
    }
    ~Message();
    void save(Folder &);
    void remove(Folder &);
};
void Message::move_Folders(Message *m)
{
    m->remove_from_Folders();
    this->folders = std::move(m->folders);
    this->add_to_Folders(*this);
    m->folders.clear();
}
void swap(Message &lhs, Message &rhs)
{
    using std::swap;
    lhs.remove_from_Folders();
    rhs.remove_from_Folders();
    swap(lhs.folders, rhs.folders);
    swap(lhs.contents, rhs.contents);
    lhs.add_to_Folders(lhs);
    rhs.add_to_Folders(rhs);
}

void Message::save(Folder &f)
{
    folders.insert(&f);
    f.addMsg(this);
}

void Message::remove(Folder &f)
{
    folders.erase(&f);
    f.removeMsg(this);
}

void Message::add_to_Folders(const Message &m)
{
    for (auto f : m.folders)
        f->addMsg(this);
}

void Message::remove_from_Folders()
{
    for (auto f : this->folders)
        f->removeMsg(this);
}

Message::Message(const Message &m) : contents(m.contents), folders(m.folders)
{
    add_to_Folders(m);
}

Message &Message::operator=(const Message &m)
{
    remove_from_Folders();
    contents = m.contents;
    folders = m.folders;
    add_to_Folders(m);
    return *this;
}

Message &Message::operator=(Message &&m)
{
    m.remove_from_Folders();
    this->contents = std::move(m.contents);
    this->move_Folders(&m);
    return *this;
}

Message::~Message()
{
    remove_from_Folders();
}

class Folder
{
    friend void swap(Folder &, Folder &);
    friend class Message;

public:
    Folder() = default;
    Folder(const Folder &);
    Folder &operator=(const Folder &);
    ~Folder();

private:
    std::set<Message *> msgs;
    void add_to_Message(const Folder &);
    void remove_from_Message();
    void addMsg(Message *m) { msgs.insert(m); }
    void removeMsg(Message *m) { msgs.erase(m); }
};

void swap(Folder &lhs, Folder &rhs)
{
    using std::swap;
    lhs.remove_from_Message();
    rhs.remove_from_Message();
    swap(lhs.msgs, rhs.msgs);
    lhs.add_to_Message(lhs);
    rhs.add_to_Message(rhs);
}

void Folder::add_to_Message(const Folder &f)
{
    for (auto m : f.msgs)
        m->addFolder(this);
}

void Folder::remove_from_Message()
{
    for (auto m : this->msgs)
        m->removeFolder(this);
}

Folder::Folder(const Folder &f) : msgs(f.msgs)
{
    add_to_Message(f);
}

Folder &Folder::operator=(const Folder &f)
{
    remove_from_Message();
    this->msgs = f.msgs;
    add_to_Message(f);
    return *this;
}

Folder::~Folder()
{
    remove_from_Message();
}
#endif
```

### 简单的智能指针

```C++
// 定义你自己的使用引用计数版本的 HasPtr。
#ifndef HASPTR_27_H_
#define HASPTR_27_H_

#include <string>

class HasPtr
{
private:
    std::string *ps;
    int i;
    std::size_t *use;

public:
    HasPtr(const std::string &s = std::string()) : ps(new std::string(s)), i(0), use(new std::size_t(1)) {}
    HasPtr(const HasPtr &hp) : ps(hp.ps), i(hp.i), use(hp.use) { ++*use; }
    HasPtr &operator=(const HasPtr &rhs)
    {
        ++*rhs.use;
        if (--*use == 0)
        {
            delete ps;
            delete use;
        }
        ps = rhs.ps;
        i = rhs.i;
        use = rhs.use;
        return *this;
    }
    ~HasPtr()
    {
        if (--*use == 0)
        {
            delete ps;
            delete use;
        }
    }
};
#endif
```

### 实现vec(初级)

```C++
// 编写自己的vector<string>
#ifndef STRVEC_H_
#define STRVEC_H_

#include <string>
#include <memory>
// #include <utility>
class Vec
{
public:
    Vec() : elements(nullptr), first_free(nullptr), cap(nullptr) {}
    Vec(const Vec &);
    Vec &operator=(const Vec &);
    Vec(std::initializer_list<std::string>);
    Vec(Vec &&s) noexcept : alloc(std::move(s.alloc)), elements(std::move(s.elements)), first_free(std::move(s.first_free)), cap(std::move(cap)) { s.elements = s.first_free = s.cap = nullptr; }
    ~Vec();
    void push_back(const std::string &);
    size_t size() const
    {
        return first_free - elements;
    }
    size_t capacity() const
    {
        return cap - elements;
    }
    std::string *begin() const
    {
        return elements;
    }
    std::string *end() const
    {
        return first_free;
    }
    void reserve(size_t n);
    void resize(size_t n);
    void resize(size_t n, const std::string &s);

private:
    std::string *elements;
    std::string *first_free;
    std::string *cap;
    std::allocator<std::string> alloc;
    std::pair<std::string *, std::string *> alloc_n_copy(const std::string *, const std::string *);
    void free();
    void reallocate();
    void my_alloc(size_t);
    void chk_n_alloc()
    {
        if (size() == capacity())
            reallocate();
    }
};

void Vec::free()
{
    if (elements)
    {
        for (auto p = first_free; p != elements;)
            alloc.destroy(--p);
        alloc.deallocate(elements, cap - elements);
    }
}

void Vec::my_alloc(size_t n)//处理需要自赋值的情况
{
    auto new_data = alloc.allocate(n);
    auto dest = new_data;
    auto elem = elements;
    for (size_t i = 0; i != size(); ++i)
    {
        alloc.construct(dest++, std::move(*elem++));
    }
    free();
    elements = new_data;
    first_free = dest;
    cap = elements + n;
}

void Vec::reallocate()
{
    auto new_capacity = size() ? 2 * size() : 1;
    my_alloc(new_capacity);
}

void Vec::push_back(const std::string &s)
{
    chk_n_alloc();
    alloc.construct(first_free++, s);
}

std::pair<std::string *, std::string *> Vec::alloc_n_copy(const std::string *b, const std::string *e)
{
    auto data = alloc.allocate(e - b);
    return {data, uninitialized_copy(b, e, data)};//非自赋值，unitiallized_copy
}

Vec::Vec(const Vec &sv)
{
    auto data = alloc_n_copy(sv.begin(), sv.end());
    elements = data.first;//first
    first_free = cap = data.second;//second
}

Vec &Vec::operator=(const Vec &rhs)
{
    auto data = alloc_n_copy(rhs.begin(), rhs.end());
    free();
    elements = data.first;
    first_free = cap = data.second;
    return *this;
}

Vec::Vec(std::initializer_list<std::string> ilst)
{
    auto data = alloc_n_copy(ilst.begin(), ilst.end());
    elements = data.first;
    first_free = cap = data.second;
}

Vec::~Vec()
{
    free();
}

void Vec::reserve(size_t n)
{
    if (n <= capacity())
        return;
    my_alloc(n);
}

void Vec::resize(size_t n)
{
    resize(n, std::string());
}

void Vec::resize(size_t n, const std::string &s)
{
    if (n < size())
    {
        while (n < size())
            alloc.destroy(--first_free);
    }
    else if (n > size())
    {
        while (n > size())
            push_back(s);
    }
}
#endif
```

### 实现vec(高级)

```C++
#ifndef STRVEC_H_
#define STRVEC_H_

#include <string>
#include <utility>
#include <memory>
#include <algorithm>

class StrVec
{
    friend bool operator==(StrVec &lhs, StrVec &rhs);
    friend bool operator!=(StrVec &lhs, StrVec &rhs);
    friend bool operator<(StrVec &lhs, StrVec &rhs);
    friend bool operator>(StrVec &lhs, StrVec &rhs);
    friend bool operator<=(StrVec &lhs, StrVec &rhs);
    friend bool operator>=(StrVec &lhs, StrVec &rhs);

public:
    StrVec() : elements(nullptr), first_free(nullptr), cap(nullptr) {}
    StrVec(std::initializer_list<std::string>);
    StrVec(const StrVec &);
    StrVec(StrVec &&s) noexcept : alloc(std::move(s.alloc)), elements(std::move(s.elements)), first_free(std::move(s.first_free)), cap(std::move(s.cap)) { s.elements = s.first_free = s.cap = nullptr; }
    template <typename... Args>
    void emplace_back(Args &&...args);
    StrVec &operator=(const StrVec &);
    StrVec &operator=(StrVec &&) noexcept;
    std::string &operator[](std::size_t n) { return elements[n]; }
    const std::string &operator[](std::size_t n) const { return elements[n]; }
    ~StrVec();
    void push_back(const std::string &);
    size_t size() const { return first_free - elements; }
    size_t capacity() const { return cap - elements; }
    std::string *begin() const { return elements; }
    std::string *end() const { return first_free; }
    void reserve(size_t n);
    void resize(size_t n);
    void resize(size_t n, const std::string &s);

private:
    std::allocator<std::string> alloc;
    void chk_n_alloc()
    {
        if (size() == capacity())
            reallocate();
    }
    std::pair<std::string *, std::string *> alloc_n_copy(const std::string *, const std::string *);
    void free();
    void reallocate();
    std::string *elements;
    std::string *first_free;
    std::string *cap;
};

StrVec::StrVec(std::initializer_list<std::string> il)
{
    auto newdata = alloc_n_copy(il.begin(), il.end());
    elements = newdata.first;
    first_free = cap = newdata.second;
}

template <typename... Args>
inline void StrVec::emplace_back(Args &&...args)
{
    chk_n_alloc();
    alloc.construct(first_free++, std::forward<Args>(args)...);
}

void StrVec::push_back(const std::string &s)
{
    chk_n_alloc();
    alloc.construct(first_free++, s);
}

std::pair<std::string *, std::string *> StrVec::alloc_n_copy(const std::string *b, const std::string *e)
{
    auto data = alloc.allocate(e - b);
    return {data, uninitialized_copy(b, e, data)};
}

void StrVec::free()
{
    if (elements)
    {
        std::for_each(elements, first_free, [this](std::string &p)
                      { alloc.destroy(&p); });
        // for(auto p = first_free; p != elements; )
        //  alloc.destroy(--p);
        alloc.deallocate(elements, cap - elements);
    }
}

StrVec::StrVec(const StrVec &s)
{
    auto newdata = alloc_n_copy(s.begin(), s.end());
    elements = newdata.first;
    first_free = cap = newdata.second;
}

StrVec::~StrVec()
{
    free();
}

void StrVec::reserve(size_t n)
{
    if (n > capacity())
        return;
    auto newdata = alloc.allocate(n);
    auto dest = newdata;
    auto elem = elements;
    for (size_t i = 0; i != size(); ++i)
        alloc.construct(dest++, std::move(*elem++));
    free();
    elements = newdata;
    first_free = dest;
    cap = elements + n;
}

void StrVec::resize(size_t n)
{
    resize(n, std::string());
}

void StrVec::resize(size_t n, const std::string &s)
{
    if (n < size())
    {
        while (n < size())
            alloc.destroy(--first_free);
    }
    else if (n > size())
    {
        while (n > size())
            push_back(s);
        // alloc.construct(first_free, s);
    }
}

StrVec &StrVec::operator=(const StrVec &rhs)
{
    auto data = alloc_n_copy(rhs.begin(), rhs.end());
    free();
    elements = data.first;
    first_free = cap = data.second;
    return *this;
}

StrVec &StrVec::operator=(StrVec &&rhs) noexcept
{
    if (this != &rhs)
    {
        free();
        alloc = std::move(rhs.alloc);
        elements = std::move(rhs.elements);
        first_free = std::move(rhs.first_free);
        cap = std::move(rhs.cap);
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}

void StrVec::reallocate()
{
    auto newcapacity = size() ? 2 * size() : 1;
    auto newdata = alloc.allocate(newcapacity);
    auto dest = newdata;
    auto elem = elements;
    for (size_t i = 0; i != size(); ++i)
        alloc.construct(dest++, std::move(*elem++));
    free();
    elements = newdata;
    first_free = dest;
    cap = elements + newcapacity;
}

bool operator==(StrVec &lhs, StrVec &rhs)
{
    return lhs.size() == rhs.size() && std::equal(lhs.begin(), lhs.end(), rhs.begin());
}

bool operator!=(StrVec &lhs, StrVec &rhs)
{
    return !(lhs == rhs);
}

bool operator<(StrVec &lhs, StrVec &rhs)
{
    return std::lexicographical_compare(lhs.begin(), lhs.end(), rhs.begin(), rhs.end());
}

bool operator>(StrVec &lhs, StrVec &rhs)
{
    return rhs < lhs;
}

bool operator<=(StrVec &lhs, StrVec &rhs)
{
    return !(rhs < lhs);
}

bool operator>=(StrVec &lhs, StrVec &rhs)
{
    return !(lhs < rhs);
}

#endif

```



### 继承

#### 入门级

##### quote

```C++
#ifndef QUOTE_H_
#define QUOTE_H_

#include <string>
#include <iostream>

class Quote
{
public:
	Quote() = default;
	Quote(const std::string &book, double sales_price) : bookNo(book), price(sales_price) {}
	Quote(const Quote &);
	Quote(Quote &&) noexcept;
	Quote &operator=(const Quote &);
	Quote &operator=(Quote &&) noexcept;
	std::string isbn() const { return bookNo; }
	virtual double net_price(std::size_t n) const { return n * price; }
	virtual void debug() const;
	virtual ~Quote() = default;
	virtual Quote *clone() const & { return new Quote(*this); }//左值调用
	virtual Quote *clone() && { return new Quote(std::move(*this)); }//右值调用

private:
	std::string bookNo;

protected:
	double price = 0;
};

void Quote::debug() const
{
	std::cout << "bookNo: " << bookNo << "; price: " << price << std::endl;
}

double print_total(std::ostream &os, const Quote &item, std::size_t n)
{

	double ret = item.net_price(n);
	os << "ISBN: " << item.isbn()
	   << " # sold: " << n << " total due: " << ret << std::endl;
	return ret;
}

#endif

```

##### disc_quote

```C++
#ifndef DISC_QUOTE_
#define DISC_QUOTE_

#include "Quote.h"
#include <string>

class Disc_quote : public Quote
{
public:
    Disc_quote() = default;
    Disc_quote(const std::string &book, double price, std::size_t qty, double disc) : Quote(book, price), quantity(qty), discount(disc) {}
    Disc_quote(const Disc_quote &rhs) : quantity(rhs.quantity), discount(rhs.discount) {}
    Disc_quote(Disc_quote &&rhs) : quantity(std::move(rhs.quantity)), discount(rhs.discount) {}
    Disc_quote &operator=(const Disc_quote &);
    Disc_quote &operator=(Disc_quote &&);
    double net_price(std::size_t) const = 0;

protected:
    std::size_t quantity = 0;
    double discount = 0.0;
};
Disc_quote &Disc_quote::operator=(const Disc_quote &rhs)
{
    quantity = rhs.quantity;
    discount = rhs.discount;
    return *this;
}
Disc_quote &Disc_quote::operator=(Disc_quote &&rhs)
{
    quantity = std::move(rhs.quantity);
    discount = std::move(rhs.discount);
    return *this;
}
#endif
```



##### bulk_quote

```C++
#ifndef BULK_QUOTE_H_
#define BULK_QUOTE_H_

#include "Disc_quote.h"

class Bulk_quote : public Disc_quote
{
public:
    Bulk_quote() = default;
    using Disc_quote::Disc_quote;// 默认 拷贝 移动构造函数不会被继承
    double net_price(std::size_t) const override;
    void debug() const override;
    Bulk_quote *clone() const & override { return new Bulk_quote(*this); }
    Bulk_quote *clone() && override { return new Bulk_quote(std::move(*this)); }
};

void Bulk_quote::debug() const
{
    Quote::debug();
    std::cout << "; quantity: " << quantity
              << "; discount: " << discount << std::endl;
}

double Bulk_quote::net_price(size_t cnt) const
{
    if (cnt >= quantity)
        return cnt * price * (1 - discount);
    else
        return cnt * price;
}
#endif
```

#### 劝退级

##### textquery && queryresult

```C++
#ifndef TEXTQUERY_H_
#define TEXTQUERY_H_
#include <string>
#include <map>
#include <set>
#include <vector>
#include <iostream>
#include <sstream>
#include <memory>
#include <fstream>
#include <make_plural.h>//自己写的
using namespace std;
using line_no = vector<string>::size_type;

struct QueryResult;

struct TextQuery
{
private:
    shared_ptr<vector<string>> text;
    map<string, std::shared_ptr<set<line_no>>> m;

public:
    TextQuery(ifstream &);
    QueryResult query(string &) const;
};

TextQuery::TextQuery(ifstream &ifs) : text(new vector<string>)
{
    string line;
    while (getline(ifs, line))
    {
        text->push_back(line);
        int r = text->size() - 1;
        istringstream iss(line);
        string word;
        while (iss >> word)
        {
            auto &lines = m[word];
            if (!lines)
                lines.reset(new set<line_no>);
            lines->insert(r);
        }
    }
}

QueryResult TextQuery::query(string &s) const
{
    static shared_ptr<set<line_no>> nodata(new set<line_no>);
    auto loc = m.find(s);
    if (loc == m.end())
        return QueryResult(s, nodata, text);
    else
        return QueryResult(s, loc->second, text);
}

struct QueryResult
{
    friend ostream &print(ostream &, const QueryResult &);

private:
    string word;
    shared_ptr<set<line_no>> lines;
    shared_ptr<vector<string>> text;

public:
    QueryResult(string s, shared_ptr<set<line_no>> lines_, shared_ptr<vector<string>> text_) : word(s), lines(lines_), text(text_) {}
};

ostream &print(ostream &os, const QueryResult &qr)
{
    os << qr.word << " occurs " << qr.lines->size() << " " << make_plural(qr.lines->size(), "time", "s") << endl;
    for (const auto &num : *qr.lines)
        os << "\t(line " << num + 1 << ") " << (*qr.text)[num] << endl;
}

void runQueries(ifstream &infile)
{
    TextQuery tq(infile);
    while (true)
    {
        cout << "enter word to look for, or q to quit: ";
        string s;
        if (!(cin >> s) || s == "q")
            break;
        print(cout, tq.query(s)) << endl;
    }
}

int main()
{
    ifstream ifs("txt");
}
#endif
```

##### query && query_base && base派生出的四个子类

看书，打不动了、、、

query存在的意义是将base子类返回的指向base类的指针封装起来，便于操作

## 流

### cin

```c++
int a,b;
std::cin>>a>>b;//此时输入“1 2[回车]”之后，cin处于回车这个字符
std::cin.ignore();//消耗掉回车，或者 std::cin.ignore(100,'/n');
std::cin>>a>>b;//假设没有上一行，正常输入两个整数也不会出错，cin会掠过回车，直到下一个int

while(getline(cin,buff))//结尾的回车被getline消耗掉（不在buff中），cin处于回车的下一个字符

string s1;//输入“test1    test2”
cin >> s1;//cin处于test1之后的第一个空格
cout << s1 << endl;
getline(cin, s1);
cout << s1 << endl;//输出“    test2”
```

### cout

```C++
vector<int> v1{1, 2, 3};
vector<int> v2 = {1, 3};
cout << boolalpha << (v1 == v2) << endl;//boolalpha直接输出true or false
```



### ifstream

```C++
ifstream ifs("文件名");
std::string buff;
while(getline(ifs,line))
```

### istringstream

```C++
istringitream iss("asd zxc");//构造
ifs.str("ddd");//赋值
while(iss>>buff)
```

### ostringstream

```C++
ostringstream oss;
oss<<"asdf";//oss中现在有了四个字符
string s=oss.str();
```

### clear

```c++
while (cin>>num1)
    cout << num1 << endl;
cin.clear();//需要将cin复原
while(getline(cin, num1))//不需要
```

### 常用的读取文件操作

```c++
string line, word;
istringstream record;
ifstream ifs("personinfo");
while (getline(ifs, line))
{
    record.str(line);
    PersonInfo info;
    record >> info.name;
    while (record >> word)
        info.phones.push_back(word);
    record.clear();
}
```

### 流迭代器

```C++
#include <iostream>
#include <iterator>
#include <fstream>
#include <vector>
#include <algorithm>

using namespace std;

int main()
{
    // ifstream ifs("../ch08_The_IO_Library/data");
    // istream_iterator<string> str_istream_iter(ifs), eof;
    istream_iterator<string> str_istream_iter(cin), eof;

    vector<string> v1(str_istream_iter, eof);
    cout << endl;
    //cout_iter将string写到cout中去，每个string后面跟一个“ ”
    ostream_iterator<string> cout_iter(cout, " ");
    copy(v1.begin(), v1.end(), cout_iter);
    cout << endl;
    
    //ofs_iter将string写到文件中去，每个string后面跟一个“\n”
    ofstream ofs("输出到的文件名");
    ostream_iterator<string> ofs_iter(ofs_odd, "\n");
    copy(v1.begin(), v1.end(), ofs_iter);
    
/*
dfafd afssdfadf dasfdaerqrew
dafasf fdasgaafd        
dfafd afssdfadf dasfdaerqrew dafasf fdasgaafd 
前两行是输入，第一行回车之后cin还处于开启的状态，等待后续输入，需要输入Ctrl+D结束cin
*/
    return 0;
}
```



## 顺序容器

### 初始化

```C++
vector<int> vi{1,2,3,4,5,6};//列表初始化 用()不行
vector<int> vi(10, 1);//十个一
list<int> ilst(5, 4);
vector<double> dvc(ilst.begin(), ilst.end());// from list<int> to vector<double>

int ia[] = {0, 1, 1, 2, 3, 5, 8, 13, 21, 55, 89};
vector<int> v1(begin(ia), end(ia));//数组用begin(ia),end(ia)
```

### insert

```C++
vector<int> vi{1,2,3,4,5};
auto iter=vi.begin();
iter=vi.insert(iter,10);//括号中的迭代器及其后面的迭代器都会失效，10插在1之前（iter指向刚添加的元素）
++iter;
++iter;//现在指向的是2
```

### erase

```C++
vector<int> vi{1,2,3,4,5};
auto iter=vi.begin();
iter=vi.erase(iter);//括号中的迭代器及其后面的迭代器都会失效，返回的是iter的下一个位置
```



### forward_list

​	单向链表比较特殊，一般需要两个迭代器才能完成操作

#### erase_after

```C++
//删除偶数
forward_list<int> flst = {0, 1, 3, 4, 5, 6, 7, 8, 9};
auto prev = flst.before_begin();
auto curr = flst.begin();
while (curr != flst.end())
{
    if (*curr % 2)
        curr = flst.erase_after(prev);//删除prev之后的元素，返回prev的下两个位置（curr指向被删掉的元素的下一个元素）
    else
    {
        prev = curr;
        ++curr;
    }
}
```

#### insert_after

```C++
//在偶数前插入一个9
forward_list<int> flst = {0, 1, 3, 4, 5, 6, 7, 8, 9};
auto prev = flst.before_begin();
auto curr = flst.begin();
while (curr != flst.end())
{
    if (*curr % 2 == 0)
    {
        curr = flst.insert_after(prev, 9); //在prev之后添加元素，返回prev的下一个位置（curr指向刚添加的元素）
        ++curr;
    }
    prev = curr;
    ++curr;
}
```

### string

#### insert && append

```C++
string name("tx");
cout << add_pre_post01(name, "Mr.", "Jr.") << endl;
cout << add_pre_post02("TX", "Mr.", "Jr.") << endl;

string add_pre_post01(const string &name, const string &pre, const string &post)
{
    string res = name;
    res.insert(res.begin(), pre.begin(), pre.end());
    // res.insert(res.end(), post.begin(), post.end());
    // return res;
    return res.append(post);
}

string add_pre_post02(const string &name, const string &pre, const string &post)
{
    string res = name;
    res.insert(0, pre);
    return res.insert(res.size(), post);
}
```

#### find_first_of

```c++
string numbers{"123456789"};
string alphabet{"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"};
string str{"ab2c3d7R4E6"};
//str.find_first_of(numbers, pos) 从pos处开始找第一个在numbers中出现的字符，返回其位置，没有返回string::npos
for (string::size_type pos = 0; (pos = str.find_first_of(numbers, pos)) != string::npos; ++pos)
{
    cout << "found number at index: " << pos
        << " element is " << str[pos] << endl;
}
for (string::size_type pos = 0; (pos = str.find_first_of(alphabet, pos)) != string::npos; ++pos)
{
    cout << "found alphabet at index: " << pos
        << " element is " << str[pos] << endl;
}

for (string::size_type pos = 0; (pos = str.find_first_not_of(alphabet, pos)) != string::npos; ++pos)
{
    cout << "found number at index: " << pos
        << " element is " << str[pos] << endl;
}
for (string::size_type pos = 0; (pos = str.find_first_not_of(numbers, pos)) != string::npos; ++pos)
{
    cout << "found alphabet at index: " << pos
        << " element is " << str[pos] << endl;
}
```

#### find && substr



## 关联容器

### 初始化

```C++
bool compareIsbn(const Sales_data &sales_data1, const Sales_data &sales_data2)
{
    return sales_data1.isbn() < sales_data2.isbn();
}

int main()
{
    using COMPAREISBN = bool (*)(const Sales_data &, const Sales_data &);
    std::multiset<Sales_data, COMPAREISBN> bookstore(compareIsbn);//通过自定义的比较器来初始化
    // std::multiset<Sales_data, decltype(compareIsbn)*> bookstore(compareIsbn);
 
    map<int, string> m = {{1, "a"}, {2, "b"}};

    return 0;
}
```

### emplace_back  && emplace

```C++
vector<pair<string, int>> vp;
int i;
string s;
vp.emplace_back(s, i);
//对于关联容器，没有emplace_back，只有emplace
```

### insert

```C++
map<string, size_t> word_count;
string word;
while (cin >> word)
{
    //ret是一个pair，first是迭代器，second是bool
    auto ret = word_count.insert({word, 1});
    if (!ret.second)
        ++ret.first->second;
}
```

### find

```C++
#include <map>
#include <vector>
#include <string>
#include <iostream>

using namespace std;

int main()
{
    map<string, vector<int>> m{{"a", {1, 2, 3}}};
    auto iter = m.find("a");
}
```

### erase

```C++
#include <map>
#include <string>
#include <iostream>

int main()
{
    std::multimap<std::string, std::string> m1 = {{"a", "abc"}, {"c", "bcd"}, {"b", "cde"}};
    auto iter = m1.find("a");
    if (iter != m1.end())
        m1.erase(iter);
    for (const auto &p : m1)
        std::cout << p.first << " " << p.second << std::endl;

    return 0;
}

```





## lambda表达式

### 捕获列表

```C++
int a = 1;
int b = 2;

// Default capture by value
[=]() { return a + b; }; // OK; a and b are captured by value

// Default capture by reference
[&]() { return a + b; }; // OK; a and b are captured by reference

[=a]() { return a; }; // 不能写=a，要只写一个a，就是值捕获
[&a]() { return a; }; // 可以写&a或者&

[=, &b]() {
    a = 2; // Illegal; 'a' is capture by value, and lambda is not 'mutable'
    b = 2; // OK; 'b' is captured by reference
};
```



## bind

```C++
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <functional>

using namespace std;

bool judge_size(string &s, string::size_type sz)
{
    return s.size() >= sz;
}

int main()
{
    vector<string> vs = {"d", "c", "b", "a", "a", "c", "e", "bb", "aa", "aaa", "aaaaa"};
    //函数名后面的是依次传递给函数的实参
    cout << count_if(vs.begin(), vs.end(), bind(judge_size, placeholders::_1, 6)) << endl;

    return 0;
}

```



## 泛型算法

### count && accumulate

```C++
#include <vector>
#include <iostream>
#include <algorithm>//count
#include <numeric>//accumulate

using namespace std;

int main()
{
    vector<int> v1 = {1, 2, 3, 1, 1};
    cout << count(v1.cbegin(), v1.cend(), 1) << endl;
    cout << accumulate(v1.cbegin(), v1.cend(), 0) << endl;
    return 0;
}
```

### count_if

```C++
vector<string> vs = {"d", "c", "b", "a", "a", "c", "e", "bb", "aa", "aaa", "aaaaa"};
string::size_type sz = 5;
cout << count_if(vs.begin(), vs.end(), [sz](const string &s)
                 { return s.size() >= sz; })
    << endl;
```

### inserter && copy

```C++
#include <iostream>
#include <vector>
#include <list>
#include <algorithm>

using namespace std;

int main()
{
    vector<int> v1 = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    list<int> l1, l2, l3;

    copy(v1.begin(), v1.end(), back_inserter(l1));//1 2 3 4 5 6 7 8 9 
    copy(v1.begin(), v1.end(), front_inserter(l2));//9 8 7 6 5 4 3 2 1 
    copy(v1.begin(), v1.end(), inserter(l3, l3.begin()));//1 2 3 4 5 6 7 8 9 

    return 0;
}
```

### fill_n

```C++
//编写程序，使用 fill_n 将一个序列中的 int 值都设置为 0。

vector<int> v1(10, 1);
fill_n(v1.begin(), v1.size(), 0);

vector<int> v2;
fill_n(v2.end(), 10, 0);//不行
//首先end()返回的迭代器是不支持解引用的，要想修改最后一个元素使用v.end()-1，强行对end()解引用会报异常。

std::back_inserter(v2)) = 10;// 是在末尾新增一个元素。    
// 标准库算法不会改变他们所操作的容器的大小。为什么使用back_inserter不会使这一断言失效
// 标准库算法根本不知道有“容器”这个东西。它们只接受迭代器参数，运行于这些迭代器之上，通过这些迭代器来访问元素。 back_inserter 的迭代器能调用对应的插入算法

fill_n(back_inserter(v2), 10, 0);// 在末尾新增十个元素。
```

### sort && stable_sort && unique

```C++
//去掉容器中重复的元素，然后按照长度排序
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

vector<string> &elimDups(vector<string> &words)
{
    sort(words.begin(), words.end());//unique只有在重复元素相邻时才有效，所以得先排序
    auto end_unique = unique(words.begin(), words.end());
    words.erase(end_unique, words.end());
    return words;
}

bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
int main()
{
    vector<string> vs = {"d", "c", "b", "a", "a", "c", "e", "bb", "aa", "aaa"};
    for (const auto s : elimDups(vs))
        cout << s << " ";
    cout << endl;
    stable_sort(vs.begin(), vs.end(), isShorter);//如果相等保持原有顺序不变，可以编写自己的排序规则，也可以不写
    for (const auto s : vs)
        cout << s << " ";
    cout << endl;
    return 0;
}
```

### partition

```C++
//使得容器中只剩下长度大于等于5的string
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

bool greater_than_5(const string &s)
{
    return s.size() >= 5;
}

int main()
{
    vector<string> vs = {"d", "c", "b", "a", "a", "c", "e", "bb", "aa", "aaa", "aaaaa"};
    auto iter = partition(vs.begin(), vs.end(), greater_than_5);//满足条件的放到前面
    vs.erase(iter, vs.end());
    for (const auto s : vs)
        cout << s << " ";
    cout << endl;

    return 0;
}
```

### find

```C++
vector<int> a;
find(a.begin(),a.end(),1);
//这句话就表示从a的头开始一直到尾，找到第一个值为1的元素，返回的是一个指向该元素的迭代器。
```



### find_if && for_each

```C++
//使得容器中只剩下长度大于等于5的string
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

void biggies(vector<string> &words, vector<string>::size_type sz)
{
    stable_sort(words.begin(), words.end(), [](const string &a, const string &b)
                { return a.size() < b.size(); });
    auto wc = find_if(words.begin(), words.end(), [sz](const string &a)
                      { return a.size() >= sz; });//find_if
    auto cnt = words.end() - wc;
    cout << cnt << endl;
    for_each(wc, words.end(), [](const string &a)
             { cout << a << " "; });//for_each
    cout << endl;
}

int main()
{
    vector<string> vs = {"d", "c", "b", "a", "a", "c", "e", "bb", "aa", "aaa", "aaaaa"};
    biggies(vs, 5);
    return 0;
}
```

### unique_copy

```c++
#include <vector>
#include <list>
#include <algorithm>

using namespace std;

int main()
{
    vector<int> v1 = {1, 1, 1, 2, 3, 4, 5};
    list<int> l1;
    unique_copy(v1.begin(), v1.end(), back_inserter(l1));

    return 0;
}

```

### reverse_iterator

#### 逆序输出

```C++
#include <iostream>
#include <iterator>
#include <vector>

using namespace std;

int main()
{
    vector<int> v1 = {1, 2, 3, 4, 5, 6, 7, 8};

    // for (auto iter = v1.end() - 1; iter != v1.begin() - 1; --iter)
    for (auto r_iter = v1.crbegin(); r_iter != v1.crend(); ++r_iter)
        cout << *r_iter << endl;
    cout << endl;

    return 0;
}
```

#### 逆序查找+删除

```C++
#include <iostream>
#include <iterator>
#include <list>
#include <algorithm>

using namespace std;

int main()
{
    list<int> l1 = {1, 2, 3, 4, 5, 6, 7, 8, 0};

    auto r_iter = find(l1.crbegin(), l1.crend(), 6);//r_iter指向6，r_iter.base()指向7
	v.erase(--r_iter.base()); // 尝试删除ri.base()前面的元素，一般来说编译不通过，因为函数返回的是右值
	v.erase((++ri).base()); // 正确删除ri指向的元素
    cout << distance(r_iter, l1.crend()) << endl;
    // cout << l1.end() - l1.begin() << endl;

    return 0;
}

```

### remove

```C++
//对于vector string等
string s = "abc";
s.erase(remove(s.begin(), s.end(), 'a')), s.end();//remove只是将要删掉的元素放在了最后，没有删掉

//对于list
list<int> l{1,2,3};
l.remove(2);//真正将元素删掉
```



### list和forward_list

​	一般用自己的算法 P370

```C++
#include <iostream>
#include <string>
#include <list>
#include <algorithm>

using namespace std;

list<string> &elimDups(list<string> &words)
{
    words.sort();
    words.unique();
    return words;
}

int main()
{
    list<string> vs = {"d", "c", "b", "a", "a", "c", "e"};

    for (const auto s : elimDups(vs))
        cout << s << " ";
    cout << endl;

    return 0;
}
```



## 内存管理

### shared_ptr

#### 初始化

```C++
std::shared_ptr<int> sp(new int(9));

int *p=new double();
std::shared_ptr<double> sp(p);

connection c;
std::shared_ptr<connection> sp(&c, [](connection *p){ disconnect(*p); });//添加删除器
std::shared_ptr<connection> sp1(sp, [](connection *p){ disconnect1(*p); });//换删除器

std::shared_ptr<int> sp1 = std::make_shared<int>(1);//new int(1)
std::shared_ptr<string> str = std::make_shared<std::string>(10, 'c');//new string(10.'c')
```

#### use_count && set

### allocator

```C++
#include <iostream>
#include <string>
#include <memory>

int main()
{
    std::allocator<std::string> alloc;//std::allocator<T> alloc
	//分配
    auto const p = alloc.allocate(5);
    auto q = p;
    alloc.construct(q++);
    alloc.construct(q++, 10, 'c');
    alloc.construct(q++, "hi");
    //释放
    while (q != p)
    {
        std::cout << *(--q) << std::endl;
        alloc.destroy(q);
    }
    alloc.deallocate(p, 5);
    return 0;
}
```

### 封装vector<\string>成类(初级)

​	只是套了一层壳，内部都没有变，用的还都是vector的函数

```C++
#ifndef STRBLOB_H_
#define STRBLOB_H_

#include <string>
#include <initializer_list>
#include <memory>
#include <vector>
#include <stdexcept>

class ConstStrBlobPtr;

class StrBlob//vector<string>
{
public:
    friend class ConstStrBlobPtr;
    typedef std::vector<std::string>::size_type size_type;
    StrBlob();
    StrBlob(std::initializer_list<std::string> il);//initializer_list
    size_type size() const { return data->size(); }
    bool empty() const { return data->empty(); }
    void push_back(const std::string &t) { data->push_back(t); }
    void pop_back();
    std::string &front();
    std::string &back();
    const std::string &front() const;
    const std::string &back() const;
    ConstStrBlobPtr begin();//将迭代器封装成一个类
    ConstStrBlobPtr end();

private:
    std::shared_ptr<std::vector<std::string>> data;//指向vector<string>的智能指针
    void check(size_type i, const std::string &msg) const;
};

StrBlob::StrBlob() : data(std::make_shared<std::vector<std::string>>()) {}//make_shared
StrBlob::StrBlob(std::initializer_list<std::string> il) : data(std::make_shared<std::vector<std::string>>(il)) {}

void StrBlob::check(size_type i, const std::string &msg) const
{
    if (i >= data->size())
        throw std::out_of_range(msg);
}

std::string &StrBlob::front()
{
    check(0, "front on empty StrBlob");
    return data->front();
}

std::string &StrBlob::back()
{
    check(0, "back on empty StrBlob");
    return data->back();
}

const std::string &StrBlob::front() const
{
    check(0, "front on empty StrBlob");
    return data->front();
}

const std::string &StrBlob::back() const
{
    check(0, "back on empty StrBlob");
    return data->back();
}

void StrBlob::pop_back()
{
    check(0, "pop_back on empty StrBlob");
    data->pop_back();
}

ConstStrBlobPtr StrBlob::begin() { return ConstStrBlobPtr(*this); }

ConstStrBlobPtr StrBlob::end()
{
    auto ret = ConstStrBlobPtr(*this, data->size());
    return ret;
}

//vector<string>::iterator
class ConstStrBlobPtr
{
public:
    ConstStrBlobPtr() : curr(0){};
    ConstStrBlobPtr(const StrBlob &a, size_t sz = 0) : wptr(a.data), curr(sz) {}
    std::string &deref() const;//迭代器解引用
    ConstStrBlobPtr &incr();

private:
    std::shared_ptr<std::vector<std::string>> check(std::size_t, const std::string &) const;
    std::weak_ptr<std::vector<std::string>> wptr;//与StrBlob指向相同的空间
    std::size_t curr;
};

std::shared_ptr<std::vector<std::string>> ConstStrBlobPtr::check(std::size_t i, const std::string &msg) const
{
    auto ret = wptr.lock();//lock，返回sp
    if (!ret)
        throw std::runtime_error("unbound ConstStrBlobPtr");
    if (i >= ret->size())
        throw std::out_of_range(msg);
    return ret;
}

std::string &ConstStrBlobPtr::deref() const
{
    auto p = check(curr, "dereference past end");
    return (*p)[curr];
}

ConstStrBlobPtr &ConstStrBlobPtr::incr()
{
    check(curr, "increment past end of ConstStrBlobPtr");
    ++curr;
    return *this;
}
#endif
```



## 模板

### 数组

```C++
#include <iostream>
#include <string>

template <typename T, unsigned N>
T *begin(T (&arr)[N])//也可以写(T &arr)
{
    return arr;
}

template <typename T, unsigned N>
T *end(T (&arr)[N])
{
    return arr + N;
}

int main()
{
    char ac[] = "aabbccdd";

    std::cout << *(begin(ac)) << std::endl;
    std::cout << *(end(ac) - 1) << std::endl;

    return 0;
}

```

### 完美转发

```C++
#include <iostream>
#include <memory>

void func_lvalue(std::string &lhs, std::string &rhs)
{
    lhs = "Wang\n";
    rhs = "Alan\n";
}

void func_rvalue(int &&lhs, int &&rhs)
{
    std::allocator<int> alloc;
    int *data(alloc.allocate(3));

    alloc.construct(data, lhs);
    alloc.construct(data + 1, 0);
    alloc.construct(data + 2, rhs);

    for (auto p = data; p != data + 3; ++p)
        std::cout << *p << "\n";

    for (auto p = data + 3; p != data;)
        alloc.destroy(--p);
    alloc.deallocate(data, 3);
}

template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)//完美转发使得值左右值属性保留不变
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));//解决右值的问题
}

int main()
{
    std::string s1, s2;
    flip(func_lvalue, s1, s2);
    std::cout << s1 << s2;
    flip(func_rvalue, 99, 88);
    int b = 0;
    int &&a = (int &&)b;
    int &c = (int &)b;
    return 0;
}

```

### 参数包

#### sizeof...

```C++
#include <iostream>
template <typename T, typename... Args>
void foo(const T &t, const Args &...rest)
{
    std::cout << sizeof...(Args) << std::endl;
    std::cout << sizeof...(rest) << std::endl;
}
int main()
{
    int i = 0;
    double d = 3.14;
    std::string s = "how now brown cow";

    foo(i, s, 42, d);
    foo(s, 42, "hi");
    foo(d, s);
    foo("hi");
    return 0;
}
```

#### 递归

```C++
#include <iostream>
#include <string>

template <typename T>
void print(std::ostream &os, const T &t)//参数包为空
{
    os << t << ",";
}

template <typename T, typename... Args>
void print(std::ostream &os, const T &t, Args... rest)//参数包不为空
{
    os << t << ",";
    print(os, rest...);
}

int main()
{
    int i = 1, *p = &i;
    double d = 0.1;
    std::string s = "abc";

    print(std::cout, i);
    std::cout << std::endl;
    print(std::cout, i, d);
    std::cout << std::endl;
    print(std::cout, i, d, s, p, "ccc");
    std::cout << std::endl;

    return 0;
}
```

#### 对包中对象进行函数调用

```C++
#include <iostream>
#include <memory>
#include <sstream>

template <typename T>
std::string debug_rep(const T &t);
template <typename T>
std::string debug_rep(T *p);

std::string debug_rep(const std::string &s);
std::string debug_rep(char *p);
std::string debug_rep(const char *p);

template <typename T>
std::string debug_rep(const T &t)
{
    std::ostringstream ret;
    ret << t;
    return ret.str();
}

template <typename T>
std::string debug_rep(T *p)
{
    std::ostringstream ret;
    ret << "pointer: " << p;

    if (p)
        ret << " " << debug_rep(*p);
    else
        ret << " null pointer";
    return ret.str();
}

std::string debug_rep(const std::string &s)
{
    return '"' + s + '"';
}

std::string debug_rep(char *p)
{
    return debug_rep(std::string(p));
}

std::string debug_rep(const char *p)
{
    std::cout << "debug_rep(const char *p)" << std::endl;
    return debug_rep(std::string(p));
}

template <typename T>
std::ostream &print(std::ostream &os, const T &t)
{
    os << t;
    return os;
}

template <typename T, typename... Args>
std::ostream &print(std::ostream &os, const T &t, const Args &...rest)
{
    os << t << ", ";
    return print(os, rest...);
}

template <typename... Args>
std::ostream &errorMsg(std::ostream &os, const Args &...rest)
{
    return print(os, debug_rep(rest)...);//对包中每一个对象进行函数调用
}

int main()
{
    std::string s = "bb";
    int *b = new int(8);

    errorMsg(std::cout, 1, b, "a", s) << std::endl;

    return 0;
}

```

### 模板特化

```C++
template <typename T>
size_t get_number(T t, std::vector<T> &vt)
{
    auto iter = vt.begin();
    size_t num = 0;
    while (iter != vt.end())
    {
        iter = std::find(iter, vt.end(), t);
        if (iter != vt.end())
        {
            ++num;
            ++iter;
        }
    }
    return num;
}

template <>
size_t get_number(const char *t, std::vector<const char *> &vt)
{
    auto iter = vt.begin();
    size_t num = 0;
    while (iter != vt.end())
    {
        iter = std::find(iter, vt.end(), t);
        if (iter != vt.end())
        {
            ++num;
            ++iter;
        }
    }
    return num;
}
```

### 实现vec<\T>(骨灰级)

```C++
// 编写自己的vec<T>
#ifndef VEC_H_
#define VEC_H_

#include <string>
#include <memory>
// #include <utility>

template <typename T>
class Vec;

template <typename T>
bool operator!=(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator==(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator<=(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator>=(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator<(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator>(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator!=(Vec<T> &lhs, Vec<T> &rhs);
template <typename T>
bool operator!=(Vec<T> &lhs, Vec<T> &rhs);

template <typename T>
class Vec
{
    friend bool operator!=<T>(Vec &lhs, Vec &rhs);
    friend bool operator==<T>(Vec &lhs, Vec &rhs);
    friend bool operator<=<T>(Vec &lhs, Vec &rhs);
    friend bool operator>=<T>(Vec &lhs, Vec &rhs);
    friend bool operator<< T>(Vec &lhs, Vec &rhs);
    friend bool operator><T>(Vec &lhs, Vec &rhs);

public:
    Vec() : elements(nullptr), first_free(nullptr), cap(nullptr) {}
    Vec(const Vec &);
    Vec &operator=(const Vec &);
    Vec(std::initializer_list<T>);
    Vec(Vec &&s) noexcept : alloc(std::move(s.alloc)), elements(std::move(s.elements)), first_free(std::move(s.first_free)), cap(std::move(cap)) { s.elements = s.first_free = s.cap = nullptr; }
    ~Vec();
    void push_back(const T &);
    size_t size() const
    {
        return first_free - elements;
    }
    size_t capacity() const
    {
        return cap - elements;
    }
    std::string *begin() const
    {
        return elements;
    }
    std::string *end() const
    {
        return first_free;
    }
    void reserve(size_t n);
    void resize(size_t n);
    void resize(size_t n, const std::string &s);

private:
    T *elements;
    T *first_free;
    T *cap;
    std::allocator<T> alloc;
    std::pair<T *, T *> alloc_n_copy(const T *, const T *);
    void free();
    void reallocate();
    void my_alloc(size_t);
    void chk_n_alloc()
    {
        if (size() == capacity())
            reallocate();
    }
};

template <typename T>
void Vec<T>::free()
{
    if (elements)
    {
        for (auto p = first_free; p != elements;)
            alloc.destroy(--p);
        alloc.deallocate(elements, cap - elements);
    }
}

template <typename T>
void Vec<T>::my_alloc(size_t n)
{
    auto new_data = alloc.allocate(n);
    auto dest = new_data;
    auto elem = elements;
    for (size_t i = 0; i != size(); ++i)
    {
        alloc.construct(dest++, std::move(*elem++));
    }
    free();
    elements = new_data;
    first_free = dest;
    cap = elements + n;
}

template <typename T>
void Vec<T>::reallocate()
{
    auto new_capacity = size() ? 2 * size() : 1;
    my_alloc(new_capacity);
}

template <typename T>
void Vec<T>::push_back(const T &s)
{
    chk_n_alloc();
    alloc.construct(first_free++, s);
}

template <typename T>
std::pair<T *, T *> Vec<T>::alloc_n_copy(const T *b, const T *e)
{
    auto data = alloc.allocate(e - b);
    return {data, uninitialized_copy(b, e, data)};
}

template <typename T>
Vec<T>::Vec(const Vec &sv)
{
    auto data = alloc_n_copy(sv.begin(), sv.end());
    elements = data.first;
    first_free = cap = data.second;
}

template <typename T>
Vec<T> &Vec<T>::operator=(const Vec &rhs)
{
    auto data = alloc_n_copy(rhs.begin(), rhs.end());
    free();
    elements = data.first;
    first_free = cap = data.second;
    return *this;
}

template <typename T>
Vec<T>::Vec(std::initializer_list<T> ilst)
{
    auto data = alloc_n_copy(ilst.begin(), ilst.end());
    elements = data.first;
    first_free = cap = data.second;
}

template <typename T>
Vec<T>::~Vec()
{
    free();
}

template <typename T>
void Vec<T>::reserve(size_t n)
{
    if (n <= capacity())
        return;
    my_alloc(n);
}

template <typename T>
void Vec<T>::resize(size_t n)
{
    resize(n, std::string());
}

template <typename T>
void Vec<T>::resize(size_t n, const std::string &s)
{
    if (n < size())
    {
        while (n < size())
            alloc.destroy(--first_free);
    }
    else if (n > size())
    {
        while (n > size())
            push_back(s);
    }
}

template <typename T>
bool operator==(Vec<T> &lhs, Vec<T> &rhs)
{
    return lhs.size() == rhs.size() && std::equal(lhs.begin(), lhs.end(), rhs.begin());
}

template <typename T>
bool operator!=(Vec<T> &lhs, Vec<T> &rhs)
{
    return !(lhs == rhs);
}

template <typename T>
bool operator<(Vec<T> &lhs, Vec<T> &rhs)
{
    return std::lexicographical_compare(lhs.begin(), lhs.end(), rhs.begin(), rhs.end());
}

template <typename T>
bool operator>(Vec<T> &lhs, Vec<T> &rhs)
{
    return rhs < lhs;
}

template <typename T>
bool operator<=(Vec<T> &lhs, Vec<T> &rhs)
{
    return !(rhs > lhs);
}

template <typename T>
bool operator>=(Vec<T> &lhs, Vec<T> &rhs)
{
    return !(rhs < lhs);
}
#endif
```

### 封装vector<\string>成类(高级)

```C++
#ifndef STRBLOB_H_
#define STRBLOB_H_

#include <string>
#include <initializer_list>
#include <memory>
#include <vector>
#include <stdexcept>

template <typename T>
class ConstStrBlobPtr;

template <typename T>
class StrBlob
{
public:
    friend class ConstStrBlobPtr<T>;
    typedef typename std::vector<T>::size_type size_type;
    StrBlob();
    StrBlob(std::initializer_list<T> il);
    size_type size() const { return data->size(); }
    bool empty() const { return data->empty(); }
    void push_back(const T &t) { data->push_back(t); }
    void pop_back();
    T &front();
    T &back();
    const T &front() const;
    const T &back() const;
    ConstStrBlobPtr<T> begin();
    ConstStrBlobPtr<T> end();

private:
    std::shared_ptr<std::vector<T>> data;
    void check(size_type i, const T &msg) const;
};

template <typename T>
class ConstStrBlobPtr
{
public:
    ConstStrBlobPtr<T>() : curr(0){};
    ConstStrBlobPtr<T>(const StrBlob<T> &a, std::size_t sz = 0) : wptr(a.data), curr(sz) {}
    T &deref() const;
    ConstStrBlobPtr<T> &incr();

private:
    std::shared_ptr<std::vector<T>> check(std::size_t, const T &) const;
    std::weak_ptr<std::vector<T>> wptr;
    std::size_t curr;
};

template <typename T>
std::shared_ptr<std::vector<T>> ConstStrBlobPtr<T>::check(std::size_t i, const T &msg) const
{
    auto ret = wptr.lock();
    if (!ret)
        throw std::runtime_error("unbound ConstStrBlobPtr<T>");
    if (i >= ret->size())
        throw std::out_of_range(msg);
    return ret;
}

template <typename T>
T &ConstStrBlobPtr<T>::deref() const
{
    auto p = check(curr, "dereference past end");
    return (*p)[curr];
}

template <typename T>
ConstStrBlobPtr<T> &ConstStrBlobPtr<T>::incr()
{
    check(curr, "increment past end of ConstStrBlobPtr<T>");
    ++curr;
    return *this;
}

template <typename T>
StrBlob<T>::StrBlob() : data(std::make_shared<std::vector<T>>()) {}

template <typename T>
StrBlob<T>::StrBlob(std::initializer_list<T> il) : data(std::make_shared<std::vector<T>>(il)) {}

template <typename T>
void StrBlob<T>::check(size_type i, const T &msg) const
{
    if (i >= data->size())
        throw std::out_of_range(msg);
}

template <typename T>
T &StrBlob<T>::front()
{
    check(0, "front on empty StrBlob");
    return data->front();
}

template <typename T>
T &StrBlob<T>::back()
{
    check(0, "back on empty StrBlob");
    return data->back();
}

template <typename T>
const T &StrBlob<T>::front() const
{
    check(0, "front on empty StrBlob");
    return data->front();
}

template <typename T>
const T &StrBlob<T>::back() const
{
    check(0, "back on empty StrBlob");
    return data->back();
}

template <typename T>
void StrBlob<T>::pop_back()
{
    check(0, "pop_back on empty StrBlob");
    data->pop_back();
}

template <typename T>
ConstStrBlobPtr<T> StrBlob<T>::begin() { return ConstStrBlobPtr<T>(*this); }

template <typename T>
ConstStrBlobPtr<T> StrBlob<T>::end()
{
    auto ret = ConstStrBlobPtr<T>(*this, data->size());
    return ret;
}

#endif
```

