# 剑指Offer

## 1. 赋值运算符函数
> 如下为类型CMyString的声明，请为该类型添加赋值运算符函数。
```cpp
class CMyString {
public:
CMyString(char* pData = nullptr);
CMyString(const CMyString& str);
~CMyString(void);
private:
char* m_pData;
};
```

注意以下几点：
* 返回值的类型为该类型的引用，才能连续赋值。
* 传入参数类型为常量引用，避免实参到形参的复制构造，并且不改变传入实例的状态。
* 分配新内存之前释放自身内存
* 判断传入参数和当前实例是否为同一个实例

**经典的解法：**
```cpp
CMyString& CMyString::operator = (const CMyString& str)
{
	if (this == &str)
		return *this;

	delete[] m_pData;
	m_pData = nullptr;

	m_pData = new char[strlen(str.m_pData) + 1];
	strcpy(m_pData, str.m_pData);

	return *this;
}
```

**考虑异常安全性的解法：**

在经典解法中，分配内存之前先用delete释放了m_pData的内存，如果此时内存不足导致new抛出异常，则m_pData将是一个空指针，这样很容易导致程序崩溃。

在赋值运算符函数中实现异常安全性，有两种方法。
1. 调整语句顺序。先用new分配新内存，再用delete释放已有的内存。这样只在分配内存成功后再释放原来的内存，也就是说分配内存失败时能确保CMyString的实例不会被修改。
2. copy and swap手法。先创建一个临时实例，再交换临时实例和原来的实例。
```cpp
CMyString& CMyString::operator = (const CMyString& str)
{
	if(this != &str)
	{
		CMyString strTemp(str);
		char* pTemp = strTemp.m_pData;
		strTemp.m_pData = m_pData;
		m_pData = pTemp;
	}
	return *this;
}
```
在新的代码中，临时实例strTemp是一个局部变量，在离开作用域后会自动调用自身的析构函数，释放strTemp.m_pData的内存。由于交换，此时strTemp.m_pData指向的内存就是原来实例的m_pData的内存，这就相当于自动调用析构函数来释放实例的内存。

如果由于内存不足，在CMyString的构造函数里用new分配内存失败而抛出异常，此时还没有修改原来实例的状态，这就保证了异常安全性。

## 2. 实现单例模式

> 设计一个类，我们只能生成该类的一个实例。

单例模式保证一个类仅有一个实例，并提供一个访问它的全局访问点，该实例被所有程序模块共享。

定义一个单例类：
1. 私有化它的构造函数，以防止外界创建单例类的对象；
2. 使用类的私有静态指针变量指向类的唯一实例；
3. 使用一个公有的静态方法获取该实例。

**懒汉式单例：**

单例实例在第一次被使用时才进行初始化，这叫做延迟初始化。

懒汉式单例在多线程环境下是线程不安全的，使用双重校验锁来解决实例可能被多次初始化的问题。加入double check lock后，其实依然是有问题的。

singleton = new Singleton1(); 这段代码其实是分为三步执行：
1. 为 Singleton1实例 分配内存空间
2. 调用构造函数初始化 Singleton1实例
3. 将 singleton 指向分配的内存地址

由于编译器的优化以及运行时优化等等原因，执行顺序有可能变成 1>3>2。指令重排在单线程环境下不会出现问题，但是在多线程环境下，一个线程在某种情况下可能会出现new返回了地址赋值给singleton变量而Singleton1此时还没有构造完全，当另一个线程随后运行到if判断时将不会进入if从而返回了不完全的实例对象给用户使用，造成了严重的错误。

在`C++11`没有出来的时候，只能靠插入两个memory barrier（内存屏障）来解决这个错误，但是`C++11`引进了memory model，提供了Atomic实现内存的同步访问，即不同线程总是获取对象修改前或修改后的值，无法在对象修改期间获得该对象。
```cpp
//懒汉 double check lock，会有re-order的问题，考虑C++11的memory model
class Singleton1 {
private:
	static atomic<Singleton1*> singleton;
	static mutex m_mutex;
private:
	Singleton1() {}
	Singleton1(const Singleton1& other) {}
	Singleton1& operator=(const Singleton1& other) {}
	~Singleton1() {}
public:
	static Singleton1* getInstance() {
		if (singleton == nullptr) {
			lock_guard<mutex> lock(m_mutex);  // 自解锁
			if (singleton == nullptr) {
				singleton = new Singleton1();
			}
		}
		return singleton;
	}
};
atomic<Singleton1*> Singleton1::singleton = nullptr;
mutex Singleton1::m_mutex;
```

**饿汉式单例：**

单例实例在程序运行时被立即执行初始化。`C++`规定，non-local static 对象在main函数之前初始化，所以没有线程安全的问题。

```cpp
//饿汉
class Singleton2 {
private:
	static Singleton2* singleton;
private:
	Singleton2() {}
	Singleton2(const Singleton2& other) {}
	Singleton2& operator=(const Singleton2& other) {}
	~Singleton2() {}
public:
	static Singleton2* getInstance() {
		return singleton;
	}
};
Singleton2* Singleton2::singleton = new Singleton2();
```

**Meyers' Singleton:**

`C++11`规定了local static在多线程条件下的初始化行为，要求编译器保证了内部静态变量的线程安全性。在`C++11`标准下，《Effective C++》提出了一种更优雅的单例模式实现，使用函数内的 local static 对象。这样，只有当第一次访问getInstance()方法时才创建实例。这种方法也被称为Meyers' Singleton。`C++0x`之后该实现是线程安全的，`C++0x`之前仍需加锁。
```cpp
class Singleton3 {
private:
	Singleton3() {}
	Singleton3(const Singleton3& other) {}
	Singleton3& operator=(const Singleton3& other) {}
	~Singleton3() {}
public:
	static Singleton3* getInstance() {
		static Singleton3 singleton;
		return &singleton;
	}
};
```

## 3. 数组中重复的数字

> 在一个长度为n的数组里的所有数字都在0 ~ n-1的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。 例如，如果输入长度为7的数组{2,3,1,0,2,5,3}，那么对应的输出是重复的数字2或者3。

[LeetCode 442. Find All Duplicates in an Array](https://leetcode.com/problems/find-all-duplicates-in-an-array/)

注意到数组中的数字都在0 ~ n-1的范围内，考虑将数字i调整到下标为i的位置，遍历数组，当扫描到下标为i的数字时，首先比较这个数字m是不是等于i。如果等于，则说明出现在正确的位置，接着扫描下一个数字。如果不等于，则比较m和下标为m的位置上的数t，若m = t，说明数字重复，若m != t，将m交换到正确的位置，即m和t交换。重复这个比较交换的过程，直到发现重复的数字。

```cpp
bool duplicate(int numbers[], int length, int* duplication)
{
	if (numbers == nullptr || length <= 0)
		return false;
	for (int i = 0; i < length; ++i)
	{
		if (numbers[i] < 0 || numbers[i] > length - 1)
			return false;
	}
	for (int i = 0; i < length; ++i)
	{
		while (numbers[i] != i)
		{
			if (numbers[i] == numbers[numbers[i]])
			{
				*duplication = numbers[i];
				return true;
			}
			swap(numbers[i], numbers[numbers[i]]);
		}
	}
	return false;
}
```

LeetCode讨论区给出了另外一种解法，核心思想是当发现数字i时，将i位置上的数翻转为负数，作为该位置上正确的数已经出现过的标记。遍历的时候如果发现i位置上的数已经为负数，说明数字i重复了。

```cpp
 bool duplicate(int numbers[], int length, int* duplication) {
        if(numbers == nullptr || length <= 0)
            return false;
        for(int i = 0; i < length; i++)
            if(numbers[i] < 0 || numbers[i] > length - 1)
                return false;
        for(int i = 0; i < length; i++){
            int index = abs(numbers[i]);
            if(numbers[index] < 0){
                *duplication = index;
                return true;
            }
            numbers[index] = -numbers[index];
        }
        return false;
    }
```

> 在一个长度为n+1的数组里的所有数字都在1 ~ n的范围内，所以数组中至少有一个数字是重复的。请找出数组中任意一个重复的数字，但不能修改输入的数组。例如，如果输入长度为8的数组{2, 3, 5, 4, 3, 2, 6, 7}，那么对应的输出是重复的数字2或者3。

[LeetCode 287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/)

**二分查找：**

把从1 ~ n的数字从中间的数字mid分为两部分，前面一半为1 ~ mid，后面一半为mid+1 ~ n，如果1 ~ mid的数字个数小于等于mid，说明重复的数字在另一半区间mid+1 ~ n内，如果大于mid，说明重复的数字在当前区间1 ~ mid内。
```cpp
int findDuplicate(vector<int>& nums) {
        if(nums.size() < 2)
            return -1;
        for(auto a : nums)
            if(a < 1 || a > nums.size() - 1)
                return -1;
        int start = 1, end = nums.size() - 1;
        while(start < end){
            int count = 0;
            int mid = start + (end - start) / 2;
            for(auto num : nums)
                if(num <= mid)
                    count++;
            if(count <= mid)
                start = mid + 1;
            else
                end = mid;
        }
        return start;
    }
```

**快慢指针：**

这个问题可以等同看做求有环链表的入口节点。
用两个指针 first，second 分别从起点开始走，first 每次走一步，second 每次走两步。
如果过程中 second 走到null，则说明不存在环。否则当 first 和 second 相遇后，让 first 返回起点，second 待在原地不动，然后两个指针每次分别走一步，当相遇时，相遇点就是环的入口。
![image](https://wx3.sinaimg.cn/large/b0c02cdely1fxmmy0paoij20dh06oweg.jpg)

证明：如上图所示，a 是起点，b 是环的入口，c 是两个指针的第一次相遇点，ab 之间的距离是 x，bc 之间的距离是 y。

用 z 表示从 c 点顺时针走到 b 的距离。则第一次相遇时 second 所走的距离是 x+(y+z)∗n+y，n 表示圈数，同时 second 走过的距离是 first 的两倍，也就是 2(x+y)，所以有 x+(y+z)∗n+y=2(x+y)，所以 x=(n−1)*(y+z)+z。那么让 second 从 c 点开始走，走 x 步，会恰好走到 b 点；让 first 从 a 点开始走，走 x 步，也会走到 b 点。

```cpp
int findDuplicate(vector<int>& nums) {
        if(nums.size() < 2)
            return -1;
        for(auto a : nums)
            if(a < 1 || a > nums.size() - 1)
                return -1;
        int fast = nums[nums[0]], slow = nums[0];
        while(fast != slow){
            fast = nums[nums[fast]];
            slow = nums[slow];
        }
        slow = 0;
        while(fast != slow){
            fast = nums[fast];
            slow = nums[slow];
        }
        return slow;
    }
```

## 4. 二维数组中的查找

> 在一个二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

[LeetCode 240. Search a 2D Matrix II](https://leetcode.com/problems/search-a-2d-matrix-ii/)

首先选取数组中右上角的数字。如果该数字等于要查找的数字，则找到目标数字，查找过程结束。如果该数字大于要查找的数字，则剔除这个数字所在的列。如果该数字小于要查找的数字，则剔除这个数字所在的行。每一步缩小查找的范围，直到找到要查找的数字，或者查找范围为空。

```cpp
bool Find(int target, vector<vector<int>> array) {
    if(array.size() == 0 || array[0].size() == 0)
        return false;
    int rows = array.size();
    int columns = array[0].size();
    int row = 0;
    int column = columns - 1;
    while(row < rows && column >= 0){
        if(array[row][column] == target)
            return true;
        else if(array[row][column] > target)
            column--;
        else
            row++;
    }
    return false;
}
```

## 5. 替换空格

> 请实现一个函数，把字符串中的每个空格替换成"%20"。例如输入“We are happy.”，则输出“We%20are%20happy.”。

先遍历一次字符串，统计出字符串中空格的总数，然后计算出替换之后的字符串的总长度。从字符串的尾部开始复制和替换，采用两个指针，分别指向替换之后字符串的末尾和原始字符串的末尾，向前移动进行复制替换。

```cpp
void replaceSpace(char *str,int length) {
    if(str == nullptr || length <= 0)
        return;
    int originLength = 0;
    int count = 0;
    char *p = str;
    while(*p != '\0'){
        if(*p == ' ')
            count++;
        originLength++;
        p++;
    }
    int newLength = originLength + 2 * count;
    if(newLength >= length)
        return;
    int originIndex = originLength;
    int newIndex = newLength;
    while(originIndex >=0 && newIndex > originIndex){
        if(str[originIndex] == ' '){
            str[newIndex--] = '0';
            str[newIndex--] = '2';
            str[newIndex--] = '%';
        }
        else
            str[newIndex--] = str[originIndex];
        originIndex--;
    }
}
```

## 6. 从尾到头打印链表

> 输入一个链表的头结点，从尾到头反过来打印出每个结点的值。
链表节点定义如下:
```cpp
struct ListNode{
    int m_nKey;
    ListNode* m_pNext;
};
```

[LeetCode 206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

用栈
```cpp
void PrintListReversingly_Iteratively(ListNode* pHead)
{
    stack<ListNode*> nodes;
    ListNode* pNode = pHead;
    while(pNode != nullptr)
    {
        nodes.push(pNode);
        pNode = pNode -> m_pNext;
    }
    while(!nodes.empty())
    {
        pNode = nodes.top();
        cout << pNode -> m_nKey << '\t';
        nodes.pop();
    }
}
```

用递归
```cpp
void PrintListReversingly_Recursively(ListNode* pHead)
{
    if(pHead != nullptr)
    {
        if (pHead->m_pNext != nullptr)
        {
            PrintListReversingly_Recursively(pHead -> m_pNext);
        }
        cout << pHead -> m_nKey << '\t';
    }
}
```

## 7. 重建二叉树

> 输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1, 2, 4, 7, 3, 5, 6, 8}和中序遍历序列{4, 7, 2, 1, 5, 3, 8, 6}，则重建出如图所示的二叉树并输出它的头结点。

```cpp
			  1
           /     \
          2       3
         /       / \
        4       5   6
         \         /
          7       8
```
二叉树节点的定义如下：
```cpp
struct TreeNode {
     int val;
     TreeNode *left;
     TreeNode *right;
     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 };
```

[LeetCode 105. Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

先根据前序遍历的第一个值创建根结点，然后在中序遍历中找到根结点的位置，这样就能确定左右子树结点的数量，递归构建左右子树即可。

```cpp
class Solution {
public:
	TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
		if (preorder.empty() || inorder.empty())
			return nullptr;
		return dfs(preorder, 0, preorder.size() - 1, inorder, 0, inorder.size() - 1);
	}

private:
	TreeNode* dfs(vector<int>& preorder, int preStart, int preEnd, vector<int>& inorder, int inStart, int inEnd) {
		if (preStart > preEnd || inStart > inEnd)
			return nullptr;
		TreeNode* root = new TreeNode(preorder[preStart]);
		int rootIndex = 0;
		for (int i = inStart; i <= inEnd; i++)
			if (inorder[i] == preorder[preStart]) {
				rootIndex = i;
				break;
			}
		int k = rootIndex - inStart;
		root->left = dfs(preorder, preStart + 1, preStart + k, inorder, inStart, inStart + k - 1);
		root->right = dfs(preorder, preStart + k + 1, preEnd, inorder, inStart + k + 1, inEnd);
		return root;
	}
};
```

## 8. 二叉树的下一个节点

> 给定一棵二叉树和其中的一个结点，如何找出中序遍历顺序的下一个结点？树中的结点除了有两个分别指向左右子结点的指针以外，还有一个指向父结点的指针。

如果当前节点存在右子树，则下一个节点为右子树的最左节点。

![image](https://ws4.sinaimg.cn/large/b0c02cdely1fxmqi2ymu8j20az0az0ss.jpg)

如果当前节点不存在右子树，则向上查找第一个当前节点为其父节点左孩子的节点，其父节点即为要求的下一个节点。

![image](https://ws2.sinaimg.cn/large/b0c02cdely1fxmqihqc23j20am0bj0ss.jpg)

```cpp
TreeLinkNode* GetNext(TreeLinkNode* pNode)
{
    if(pNode == nullptr)
            return nullptr;
    if(pNode -> right){
        TreeLinkNode* cur = pNode -> right;
        while(cur -> left)
            cur = cur -> left;
        return cur;
    }
    while(pNode -> next){
        TreeLinkNode* parent = pNode -> next;
        if(pNode == parent -> left)
            return parent;
        pNode = parent;
    }
    return nullptr;
}
```

## 9. 用两个栈实现队列

> 用两个栈实现一个队列。队列的声明如下，请实现它的两个函数appendTail和deleteHead，分别完成在队列尾部插入结点和在队列头部删除结点的功能。

[LeetCode 232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/)

stack1 栈用来处理入栈（push）操作，stack2 栈用来处理出栈（pop）操作。一个元素进入 stack1 栈之后，出栈的顺序被反转。当元素要出栈时，需要先进入 stack2 栈，此时元素出栈顺序再一次被反转，因此出栈顺序就和最开始入栈顺序是相同的，先进入的元素先退出，这就是队列的顺序。

```cpp
class Solution
{
public:
	void push(int node) {
		stack1.push(node);
	}

	int pop() {
		if (stack2.empty()) {
			while (!stack1.empty()) {
				stack2.push(stack1.top());
				stack1.pop();
			}
		}
		if (stack2.empty())
			throw new exception("队列为空，出栈失败");
		int t = stack2.top();
		stack2.pop();
		return t;
	}

private:
	stack<int> stack1;
	stack<int> stack2;
};
```

> 用两个队列实现一个栈

[LeetCode 225. Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues/)

Push时，一个空队列A用来增加元素，然后将另外一个队列B的所有元素依次出队然后添加进A队尾。此时A就变为非空，B就变为空。重复这个push操作，就能使元素后进先出。

Pop时，直接将非空队列的队首元素出队即可。

```cpp
class MyStack {
public:
    /** Initialize your data structure here. */
    MyStack() {
        
    }
    
    /** Push element x onto stack. */
    void push(int x) {
        if(q1.empty()){
            q1.push(x);
            while(!q2.empty()){
                q1.push(q2.front());
                q2.pop();
            }
        }else {
            q2.push(x);
            while(!q1.empty()){
                q2.push(q1.front());
                q1.pop();
            }
        }
    }
    
    /** Removes the element on top of the stack and returns that element. */
    int pop() {
        if(!q1.empty()){
            int t = q1.front();
            q1.pop();
            return t;
        }else{
            int t = q2.front();
            q2.pop();
            return t;
        }
        
    }
    
    /** Get the top element. */
    int top() {
        if(!q1.empty())
            return q1.front();
        else
            return q2.front();
    }
    
    /** Returns whether the stack is empty. */
    bool empty() {
        return q1.empty() && q2.empty();
    }
    
private:
    queue<int> q1;
    queue<int> q2;
};
```

## 10. 斐波那契数列

> 写一个函数，输入n，求斐波那契（Fibonacci）数列的第n项。

**迭代解法：**

使用迭代的方式，保存中间项，避免重复计算。

```cpp
int Fibonacci(int n) {
	int result[2] = { 0, 1 };
	if (n < 2)
		return result[n];
	int fib1 = 0;
	int fib2 = 1;
	int fibn = 0;
	for (int i = 2; i <= n; i++) {
		fibn = fib1 + fib2;
		fib1 = fib2;
		fib2 = fibn;
	}
	return fibn;
}
```

**矩阵乘方解法：**

先给出一个数学公式，可以用数学归纳法来证明。

$$
\left[
\begin{array}{cc}
  f(n)&f(n-1)\\
  f(n-1)&f(n-2)
\end{array}
\right]=
\left[
\begin{array}{cc}
  1&1\\
  1&0
\end{array}
\right]
^{n-1}
$$

只需要求得矩阵
$
\left[
\begin{array}{cc}
  1&1\\
  1&0
\end{array}
\right]
^{n-1}
$即可得到f(n)，考虑乘方的如下性质：
$$
a^n =
\begin{cases}
a^{n/2} \cdot a^{n/2},  & \text{n为偶数} \\
a^{(n-1)/2} \cdot a^{(n-1)/2} \cdot a, & \text{n为奇数}
\end{cases}
$$
求n次方可以用递归分治的思路实现。

```cpp
struct Matrix2By2
{
	int m_00;
	int m_01;
	int m_10;
	int m_11;
};

Matrix2By2 MatrixMultiply(const Matrix2By2& matrix1, const Matrix2By2& matrix2)
{
	Matrix2By2 res;
	res.m_00 = matrix1.m_00 * matrix2.m_00 + matrix1.m_01 * matrix2.m_10;
	res.m_01 = matrix1.m_00 * matrix2.m_01 + matrix1.m_01 * matrix2.m_11;
	res.m_10 = matrix1.m_10 * matrix2.m_00 + matrix1.m_11 * matrix2.m_10;
	res.m_11 = matrix1.m_10 * matrix2.m_01 + matrix1.m_11 * matrix2.m_11;
	return res;
}

Matrix2By2 MatrixPower(int n)
{
	Matrix2By2 matrix;
	Matrix2By2 res;
	matrix.m_00 = 1;
	matrix.m_01 = 1;
	matrix.m_10 = 1;
	matrix.m_11 = 0;
	if (n == 1)
		return matrix;
	else if (n % 2 == 0)
	{
		Matrix2By2 temp = MatrixPower(n / 2);
		res = MatrixMultiply(temp, temp);
	}
	else if (n % 2 == 1)
	{
		Matrix2By2 temp = MatrixPower((n - 1) / 2);
		res = MatrixMultiply(temp, temp);
		res = MatrixMultiply(res, matrix);
	}
	return res;
}

int Fibonacci2(int n)
{
	int result[2] = { 0, 1 };
	if (n < 2)
		return result[n];
	Matrix2By2 res = MatrixPower(n - 1);
	return res.m_00;
}
```

> 青蛙跳台阶问题。一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶。求该青蛙跳上一个n级的台阶总共有多少种跳法。

[LeetCode 70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/)

等同于求菲波那切数列，第一步先跳1级，后面有f(n-1)种跳法；第一步先跳2级，后面有f(n-2)种跳法。f(n) = f(n-1) + f(n-2).

```cpp
class Solution {
public:
    int climbStairs(int n) {
        if(n <= 0)
            return 0;
        if(n == 1)
            return 1;
        if(n == 2)
            return 2;
        int f1 = 1;
        int f2 = 2;
        int fn;
        for(int i = 3; i <= n; i++){
            fn = f1 + f2;
            f1 = f2;
            f2 = fn;
        }
        return fn;
    }
};
```

> 变态跳台阶。一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶......它也可以跳上n级。求该青蛙跳上一个n级的台阶总共有多少种跳法。

f(n) = f(n-1) + f(n-2) + f(n-3)... + f(0)

f(n-1) = f(n-2) + f(n-3) + ... + f(0)

则有f(n) = f(n-1) + f(n-1) = 2 * f(n-1)，f(1) = 1，所以f(n) = $2^{n-1}$
```cpp
class Solution {
public:
    int jumpFloorII(int number) {
        if(number <= 0)
            return 0;
        int res = 1;
        for(int i = 0; i < number - 1; i++)
            res *= 2;
        return res;
    }
};
```

> 矩形覆盖问题。我们可以用2 * 1的小矩形横着或者竖着去覆盖更大的矩形。请问用n个2 * 1的小矩形无重叠地覆盖一个2 * n的大矩形，总共有多少种方法？

记f(n)为2 * n的大矩形的覆盖方法数。用第一个小矩形去覆盖大矩形的最左边时有两种选择：竖着放和横着放。竖着放的时候，剩余的覆盖方法为f(n-1)。横着放的时候，剩余的覆盖方法为f(n-2)。所以f(n) = f(n-1) + f(n-2)。

```cpp
class Solution {
public:
    int rectCover(int number) {
        if(number <= 0)
            return 0;
        vector<int> f(number + 1);
        f[0] = 0;
        f[1] = 1;
        f[2] = 2;
        for(int i = 3; i <= number; i++)
            f[i] = f[i - 1] + f[i - 2];
        return f[number];
    }
};
```

## 11. 旋转数组的最小数字

> 把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个递增排序的数组的一个旋转，输出旋转数组的最小元素。例如数组{3, 4, 5, 1, 2}为{1, 2, 3, 4, 5}的一个旋转，该数组的最小值为1。

[LeetCode 153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)

[LeetCode 154. Find Minimum in Rotated Sorted Array II](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array-ii/)

数组无重复元素的情况下，使用二分查找：

[start...end]表示最小数字所在的区间。

nums[mid] < nums[end]时，end = mid；否则，start = mid + 1。

```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int start = 0, end = nums.size() - 1;
        while(start < end){
            int mid = start + (end - start) / 2;
            if(nums[mid] <= nums[end])
                end = mid;
            else
                start = mid + 1;
        }
        return nums[start];
    }
};
```

数组有重复元素的情况下，就会出现一个特殊的情况：nums[start] == nums[mid] == nums[end]，那么此时无法确定解在哪个区间，需要切换到顺序查找。例如对于数组 {1,1,1,0,1}，start、mid 和 end 指向的数都为 1，此时无法知道最小数字 0 在哪个区间。

```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int start = 0, end = nums.size() - 1;
        while(start < end){
            int mid = start + (end - start) / 2;
            if(nums[mid] == nums[start] && nums[mid] == nums[end])
                return findMinInOrder(nums, start, end);
            if(nums[mid] <= nums[end])
                end = mid;
            else
                start = mid + 1;
        }
        return nums[start];
    }
    
private:
    int findMinInOrder(vector<int>& nums, int start, int end){
        int res = nums[start];
        for(int i = start; i <= end; i++)
            if(nums[i] < res)
                res = nums[i];
        return res;
    }
};
```

LeetCode讨论区给出了另外一种思考方式，当nums[mid] == nums[end]时，不能确定最小值在mid的左边还是右边，此时将右边界end-1后再搜索。

```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int start = 0, end = nums.size() - 1;
        while(start < end){
            int mid = start + (end - start) / 2;
            if(nums[mid] == nums[end])
                end--;
            else if(nums[mid] < nums[end])
                end = mid;
            else
                start = mid + 1;
        }
        return nums[start];
    }
};
```

## 12. 矩阵中的路径

> 请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中任意一格开始，每一步可以在矩阵中向左、右、上、下移动一格。如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。例如在下面的3×4的矩阵中包含一条字符串“bfce”的路径（路径中的字母用下划线标出）。但矩阵中不包含字符串“abfb”的路径，因为字符串的第一个字符b占据了矩阵中的第一行第二个格子之后，路径不能再次进入这个格子。

```cpp
a b t g
c f c s
j d e h
```

采用回溯法。首先，在矩阵中任选一个格子作为路径的起点，假设矩阵中某个格子的字符为ch，并且这个格子将对应于路径上的第i个字符。如果路径上的第i个字符不是ch，那么这个格子不可能处在路径上的第i个位置。如果路径上的第i个字符刚好是ch，那么到相邻的格子寻找路径上的第i + 1个字符。重复这个过程。

当在矩阵中定位了路径中前n个字符的位置之后，在与第n个字符对应的格子的周围都没有找到第n + 1个字符，就回到第n - 1个字符，重新定位第n个字符。

由于路径不能重复进入矩阵的格子，所以还需要定义一个布尔矩阵来标识某个格子是否被进入。

```cpp
class Solution {
public:
    bool hasPath(char* matrix, int rows, int cols, char* str)
    {
        if(matrix == nullptr || rows <= 0 || cols <= 0 || str == nullptr)
            return false;
        vector<vector<bool>> visited(rows, vector<bool>(cols, false));
        for(int row = 0; row < rows; row++)
            for(int col = 0; col < cols; col++)
                if(hasPathCore(matrix, rows, cols, row, col, str, 0, visited))
                    return true;
        return false;
    }
    
private:
    bool hasPathCore(char*& matrix, int rows, int cols, int row, int col, char*& str, int pathLength, vector<vector<bool>>& visited){
        if(str[pathLength] == '\0')
            return true;
        bool flag = false;
        if(row >= 0 && row < rows && col >= 0 && col < cols && matrix[row * cols + col] == str[pathLength] && !visited[row][col]){
            pathLength++;
            visited[row][col] = true;
            flag = hasPathCore(matrix, rows, cols, row, col - 1, str, pathLength, visited) 
                || hasPathCore(matrix, rows, cols, row - 1, col, str, pathLength, visited) 
                || hasPathCore(matrix, rows, cols, row, col + 1, str, pathLength, visited) 
                || hasPathCore(matrix, rows, cols, row + 1, col, str, pathLength, visited);
            if(!flag){
                pathLength--;
                visited[row][col] = false;
            }
        }
        return flag;
    }
};
```

## 13. 机器人的运动范围

> 地上有一个m行n列的方格。一个机器人从坐标(0,0)的格子开始移动，它每一次可以向左、右、上、下移动一格，但不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格(35,37)，因为3+5+3+7=18。但它不能进入方格(35,38)，因为3+5+3+8=19。请问该机器人能够到达多少个格子？

采用回溯法。从（0，0）开始移动，当它准备进入坐标为（i，j）的格子时，通过检查坐标的数位和来判断是否能够进入。

```cpp
class Solution {
public:
	int movingCount(int threshold, int rows, int cols)
	{
		if (threshold < 0 || rows <= 0 || cols <= 0)
			return 0;
		vector<vector<bool>> visited(rows, vector<bool>(cols, false));
		int count = movingCountCore(threshold, rows, cols, 0, 0, visited);
		return count;
	}

private:
	int movingCountCore(int threshold, int rows, int cols, int row, int col, vector<vector<bool>>& visited) {
		int count = 0;
		if (row >= 0 && row < rows && col >= 0 && col < cols 
			&& getDigitSum(row) + getDigitSum(col) <= threshold && !visited[row][col]) {
			visited[row][col] = true;
			count = 1 + movingCountCore(threshold, rows, cols, row - 1, col, visited)
				+ movingCountCore(threshold, rows, cols, row, col - 1, visited)
				+ movingCountCore(threshold, rows, cols, row + 1, col, visited)
				+ movingCountCore(threshold, rows, cols, row, col + 1, visited);
		}
		return count;
	}

	int getDigitSum(int number) {
		int sum = 0;
		while (number) {
			sum += number % 10;
			number /= 10;
		}
		return sum;
	}
};
```

## 14. 剪绳子

> 给你一根长度为n的绳子，请把绳子剪成m段（m、n都是整数，n>1并且m≥1）。每段的绳子的长度记为k[0]、k[1]、……、k[m]。k[0] * k[1] * … * k[m]可能的最大乘积是多少？例如当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到最大的乘积18。

[LeetCode 343. Integer Break](https://leetcode.com/problems/integer-break/)

**动态规划**

将i拆成两部分 j 和 i - j， 则f[i] = max(j, f[j]) * max(i - j, dp[i - j])
```cpp
class Solution {
public:
    int integerBreak(int n) {
        if(n < 2)
            return 0;
        vector<int> dp(n + 1, 0);
        dp[1] = 1;
        for(int i = 2; i <= n; i++)
            for(int j = 1; j <= i / 2; j++)
                dp[i] = max(dp[i], max(j, dp[j]) * max(i - j, dp[i - j]));
        return dp[n];
    }
};
```

**贪心法**

首先把一个正整数 N 拆分成若干正整数只有有限种拆法，所以存在最大乘积。假设 N=n1+n2+…+nk，并且 n1×n2×…×nk 是最大乘积。

* 显然1不会出现在其中；
* 如果对于某 i 有 ni ≥ 5，那么把 ni 拆分成 3+(ni−3)，我们有 3(ni−3)=3ni−9>ni；
* 如果 ni=4，拆成 2+2 乘积不变，所以不妨假设没有4；
* 如果有三个以上的2，那么 3×3>2×2×2，所以替换成3乘积更大；

综上，选用尽量多的3，直到剩下2或者4时，用2。

```cpp
class Solution {
public:
    int integerBreak(int n) {
        if(n <= 3)
            return 1 * (n - 1);
        int res = 1;
        while(n > 4){
            res *= 3;
            n -= 3;
        }
        res *= n;
        return res;
    }
};
```

## 15. 二进制中1的个数

> 请实现一个函数，输入一个整数，输出该数二进制表示中1的个数。例如把9表示成二进制是1001，有2位是1。因此如果输入9，该函数输出2。

[LeetCode 191. Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/)

把一个整数减去1，再和原整数做与运算，会把该整数最右边的1变成0.

```cpp
class Solution {
public:
     int NumberOf1(int n) {
         int count = 0;
         while(n){
             n = n & (n - 1);
             count ++;
         }
         return count;
     }
};
```

> 用一条语句判断一个整数是不是2的整数次方。

[LeetCode 231. Power of Two](https://leetcode.com/problems/power-of-two/)

一个整数如果是2的整数次方，那么它的二进制表示中有且只有一位是1，而其他所有位都是0。

```cpp
class Solution {
public:
    bool isPowerOfTwo(int n) {
        if(n <= 0)
            return false;
        return !(n & (n - 1));
    }
};
```

> 输入两个整数m和n，计算需要改变m的二进制表示中的多少位才能得到n。比如10的二进制表示为1010，13的二进制表示为1101，需要改变1010中的3位才能得到1101。

[LeetCode 461. Hamming Distance](https://leetcode.com/problems/hamming-distance/)

先求这两个数的异或，再统计异或结果中1的位数。

```cpp
class Solution {
public:
    int hammingDistance(int x, int y) {
        int n = x ^ y;
        int count = 0;
        while(n){
            n &= n - 1;
            count++;
        }
        return count;
    }
};
```

## 16. 数值的整数次方

> 实现函数double Power(double base, intexponent)，求base的exponent次方。不得使用库函数，同时不需要考虑大数问题。

[LeetCode 50. Pow(x, n)](https://leetcode.com/problems/powx-n/)

参考之前用矩阵求斐波那契数列的方法，利用快速幂，分治求得幂次方。

```cpp
class Solution {
public:
    double myPow(double x, int n) {
        if(n == 0)
            return 1;
        if(n == 1)
            return x;
        bool isNegative = false;
        if(n < 0){
            n = -n;
            isNegative = true;
        }
        double res = myPow(x * x, (unsigned)n >> 1);
        if(n & 0x1 == 1)
            res *= x;
        return isNegative ? 1.0 / res : res;
    }
};
```

## 17. 打印从1到最大的n位数

> 输入数字n，按顺序打印出从1最大的n位十进制数。比如输入3，则打印出1、2、3一直到最大的3位数即999。

当输入的n很大的时候，需要考虑大数问题。用字符串来存储数字，把问题转换成数字排列的解法。如果在数字前面补0，就会发现n位所有十进制数其实就是n个从0到9的全排列。也就是说，把数字的每一位都从0到9排列一遍，就得到了所有的十进制数。只是在打印的时候，排在前面的0不打印出来而已。

```cpp
class Solution {
public:
	void Print1ToMaxOfNDigits(int n)
	{
		if (n <= 0)
			return;
		string s(n, '0');
		dfs(s, 0);
	}

private:
	void dfs(string& s, int index) {
		if (index == s.size()) {
			print(s);
			return;
		}
		for (int i = 0; i < 10; i++) {
			s[index] = i + '0';
			dfs(s, index + 1);
		}
	}

	void print(string s)
	{
		int i = 0;
		while (i < s.size() && s[i] == '0')
			i++;
		if (i == s.size())
			return;
		cout << s.substr(i) << endl;
	}
};
```

> 定义一个函数，在该函数中可以实现任意两个非负整数的加法。

```cpp
class Solution {
public:
    string addStrings(string num1, string num2) {
        string res;
        int i = num1.size() - 1;
        int j = num2.size() - 1;
        int carry = 0;
        while(i >= 0 || j >= 0 || carry){
            int sum = (i >= 0 ? num1[i] - '0' : 0) + (j >= 0 ? num2[j] - '0' : 0) + carry;
            res += (sum % 10 + '0');
            carry = sum / 10;
            --i;
            --j;
        }
        reverse(res.begin(), res.end());
        return res;
    }
};
```

## 18. 删除链表的节点

> 在O(1)时间内删除链表节点。给定单向链表的头指针和一个节点指针，定义一个函数在O(1)时间内删除该节点。链表节点与函数的定义如下：

```cpp
struct ListNode {
	int val;
    ListNode *next;
    ListNode(int x) : val(x), next(NULL) {}
 };
```

[LeetCode 237. Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list/)

要删除节点i，先把i的下一个节点j的内容复制到i，然后把i的指针指向节点j的下一个节点，此时再删除节点j，其效果刚好是把节点i删除了。

如果要删除的节点位于链表的尾部，那么它就没有下一个节点。从链表的头节点开始，顺序遍历得到该节点的前序节点，完成删除操作。

如果链表中只有一个节点，在删除节点之后还需要把链表的头节点置空。

```cpp
//在O(1)时间内删除链表节点
void DeleteNode(ListNode* head, ListNode* node)
{
	if (head == nullptr || node == nullptr)
		return;
	// 要删除的结点不是尾结点
	if (node -> next != nullptr)
	{
		ListNode* pNext = node -> next;
		node -> val = pNext -> val;
		node -> next = pNext -> next;
		delete pNext;
		pNext = nullptr;
	}
	// 链表只有一个结点，删除头结点（也是尾结点）
	else if (head == node)
	{
		delete node;
		node = nullptr;
		head = nullptr;
	}
	// 链表中有多个结点，删除尾结点
	else
	{
		ListNode* p = head;
		while (p -> next != node)
			p = p -> next;
		p -> next = nullptr;
		delete node;
		node = nullptr;
	}
}
```

> 删除链表中重复的节点。

[LeetCode 82. Remove Duplicates from Sorted List II](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/)

[LeetCode 83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list/)

如果将重复的节点全部删除，使用两个指针，一个用来遍历链表，一个用来维护删除重复节点后的链表。

```cpp
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        if(head == nullptr)
            return head;
        ListNode* dummy = new ListNode(-1);
        dummy -> next = head;
        ListNode* pre = dummy;
        ListNode* cur = head;
        while (cur != nullptr) {
            while (cur -> next != nullptr && cur -> val == cur -> next -> val)
                cur = cur -> next;
            if (pre->next == cur)
                pre = pre->next;
            else
                pre->next = cur->next;
            cur = cur->next;
        }
        return dummy->next;
    }
};
```

如果将重复的链表保留至一个,则只需要调整当前节点的next指针即可。

```cpp
class Solution {
public:
    ListNode* deleteDuplicates(ListNode* head) {
        if(head == nullptr || head -> next == nullptr)
            return head;
        ListNode* cur = head;
        while(cur != nullptr){
            while(cur -> next != nullptr && cur -> val == cur -> next -> val){
                ListNode* temp = cur -> next;
                cur -> next = temp -> next;
                delete temp;
                temp = nullptr;
            }
            cur = cur -> next;
        }
        return head;
    }
};
```

## 19. 正则表达式匹配

> 请实现一个函数用来匹配包含'.'和'*'的正则表达式。模式中的字符'.'表示任意一个字符，而'*'表示它前面的字符可以出现任意次（含0次）。在本题中，匹配是指字符串的所有字符匹配整个模式。例如，字符串"aaa"与模式"a.a"和"ab*ac*a"匹配，但与"aa.a"及"ab*a"均不匹配。

[LeetCode 10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching/)

**递归**

如果模式串中的第二个字符是 '*'，考虑两种情况，重复0次和重复至少一次。

```cpp
 //x* matches empty string or at least one character: x* -> xx*
bool isMatch(string s, string p) {
		if (p.empty())
			return s.empty();
		if (p.size() == 1)
			return s.size() == 1 && (s[0] == p[0] || p[0] == '.');
		if (p[1] == '*')
			return isMatch(s, p.substr(2)) || (!s.empty() && (s[0] == p[0] || p[0] == '.') && isMatch(s.substr(1), p));
		else
			return !s.empty() && (s[0] == p[0] || p[0] == '.') && isMatch(s.substr(1), p.substr(1));
	}
```

**动态规划**

```cpp
        /* f[i][j]: if s[0..i-1] matches p[0..j-1]
         * if p[j - 1] != '*'
         *      f[i][j] = f[i - 1][j - 1] && s[i - 1] == p[j - 1]
         * if p[j - 1] == '*', denote p[j - 2] with x
         *      f[i][j] is true iff any of the following is true
         *      1) "x*" repeats 0 time and matches empty: f[i][j - 2]
         *      2) "x*" repeats >= 1 times and matches "x*x": s[i - 1] == x && f[i - 1][j]
         * '.' matches any single character
         */
bool isMatch2(string s, string p) {
		int m = s.size(), n = p.size();
		vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
		dp[0][0] = true;
		for (int i = 1; i <= n; i++)
			dp[0][i] = i > 1 && p[i - 1] == '*' && dp[0][i - 2];
		for (int i = 1; i <= m; i++)
			for (int j = 1; j <= n; j++)
				if (p[j - 1] == '*')
					dp[i][j] = dp[i][j - 2] || ((s[i - 1] == p[j - 2] || p[j - 2] == '.') && dp[i - 1][j]);
				else
					dp[i][j] = (s[i - 1] == p[j - 1] || p[j - 1] == '.') && dp[i - 1][j - 1];
		return dp[m][n];
	}
```

## 20. 表示数值的字符串

> 请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串“+100”、“5e2”、“-123”、“3.1416”及“-1E-16”都表示数值，但“12e”、“1a3.14”、“1.2.3”、“+-5”及“12e+5.4”都不是。

[LeetCode 65. Valid Number](https://leetcode.com/problems/valid-number/)

**正则表达式**

利用c++11的regex库。
```cpp
/*
[]  ： 字符集合
()  ： 分组
?   ： 重复 0 ~ 1
+   ： 重复 1 ~ n
*   ： 重复 0 ~ n
.   ： 任意字符
\\. ： 转义后的 .
\\d ： 数字
*/

class Solution {
public:
    bool isNumber(string s) {
        s.erase(0, s.find_first_not_of(" "));
        s.erase(s.find_last_not_of(" ") + 1);
        if(s.empty())
            return false;
        regex pattern("[+-]?((\\d+(\\.\\d*)?)|(\\.\\d+))([Ee][+-]?\\d+)?");
        return regex_match(s, pattern); 
    }
};
```

**DFA**

![image](https://wx3.sinaimg.cn/large/b0c02cdely1fyay13vm3kj20o9058t9c.jpg)

```cpp
class Solution {
public:
    bool isNumber(string s) {
        vector<unordered_map<string, int>> state = {{},
                                                    {{"blank", 1}, {"sign", 2}, {"digit", 3}, {".", 4}},
                                                    {{"digit", 3}, {".", 4}},
                                                    {{"blank", 9}, {"digit", 3}, {"exp", 6}, {".", 5}},
                                                    {{"digit", 5}},
                                                    {{"digit", 5}, {"exp", 6}, {"blank", 9}},
                                                    {{"sign", 7}, {"digit", 8}},
                                                    {{"digit", 8}},
                                                    {{"digit", 8}, {"blank", 9}},
                                                    {{"blank", 9}}
                                                   };
        int curState = 1;
        for(auto c : s){
            string t = "";
            if(c == ' ')
                t = "blank";
            if(c == '+' || c == '-')
                t = "sign";
            if(c >= '0' && c <= '9')
                t = "digit";
            if(c == '.')
                t = ".";
            if(c == 'e' || c == 'E')
                t = "exp";
            if(state[curState].find(t) == state[curState].end())
                return false;
            curState = state[curState][t];
        }
        if(curState == 3 || curState == 5 || curState == 8 || curState == 9)
            return true;
        return false;
    }
};
```

## 21. 调整数组顺序使奇数位于偶数前面

> 输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位于数组的后半部分。

[LeetCode 905. Sort Array By Parity](https://leetcode.com/problems/sort-array-by-parity/)

双指针解法即可，注意同类型的题，可以将判断逻辑抽离出来，实现代码的松耦合。

```cpp
class Solution {
public:
    vector<int> sortArrayByParity(vector<int>& A) {
        int i = 0, j = A.size() - 1;
        while(i < j){
            while(i < j && A[i] % 2 == 0)
                i++;
            while(i < j && A[j] % 2 == 1)
                j--;
            swap(A[i++], A[j--]);
        }
        return A;
    }
};
```

## 22. 链表中倒数第k个节点

> 输入一个链表，输出该链表中倒数第k个结点。为了符合大多数人的习惯，本题从1开始计数，即链表的尾结点是倒数第1个结点。例如一个链表有6个结点，从头结点开始它们的值依次是1、2、3、4、5、6。这个链表的倒数第3个结点是值为4的结点。

[LeetCode 19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)

快慢指针解法，初始化两个指针指向头节点，让快指针先走k - 1步，此时快指针指向正数第k个节点，慢指针指向头节点。然后两个指针同时往后移，当快指针指向最后一个节点时，慢指针指向倒数第k个节点。

```cpp
class Solution {
public:
    ListNode* FindKthToTail(ListNode* pListHead, unsigned int k) {
        if(pListHead == nullptr || k == 0)
            return nullptr;
        ListNode* fast = pListHead;
        ListNode* slow = pListHead;
        for(int i = 0; i < k - 1; i++){
            fast = fast -> next;
            if(fast == nullptr)
                return nullptr;
        }
        while(fast -> next != nullptr){
            fast = fast -> next;
            slow = slow -> next;
        }
        return slow;
    }
};
```

> 求链表的中间节点。如果链表中的节点总数为奇数，则返回中间节点，如果节点总数为偶数，则返回中间两个节点的第二个节点。

[LeetCode 876. Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list/)

定义两个指针，同时从链表的头节点出发，一个指针一次走一步，另一个指针一次走两步。当走得快的指针走到链表的末尾时，走得慢的指针正好在链表的中间。

```cpp
class Solution {
public:
    ListNode* middleNode(ListNode* head) {
        if(head == nullptr || head -> next == nullptr)
            return head;
        ListNode* fast = head;
        ListNode* slow = head;
        while(fast != nullptr && fast -> next != nullptr){
            fast = fast -> next -> next;
            slow = slow -> next;
        }
        return slow;
    }
};
```

## 23. 链表中环的入口节点

> 一个链表中包含环，如何找出环的入口结点？

[LeetCode 141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)

[LeetCode 142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/)

用两个指针 fast，slow 分别从起点开始走，slow 每次走一步，fast 每次走两步。
如果过程中 fast 走到null，则说明不存在环。否则当 fast 和 slow 相遇后，让 slow 返回起点，fast待在原地不动，然后两个指针每次分别走一步，当相遇时，相遇点就是环的入口。
![image](https://wx3.sinaimg.cn/large/b0c02cdely1fxmmy0paoij20dh06oweg.jpg)

证明：如上图所示，a 是起点，b 是环的入口，c 是两个指针的第一次相遇点，ab 之间的距离是 x，bc 之间的距离是 y。

用 z 表示从 c 点顺时针走到 b 的距离。则第一次相遇时 fast 所走的距离是 x+(y+z)∗n+y，n 表示圈数，同时 fast 走过的距离是 slow 的两倍，也就是 2(x+y)，所以有 x+(y+z)∗n+y=2(x+y)，所以 x=(n−1)*(y+z)+z。那么让 fast 从 c 点开始走，走 x 步，会恰好走到 b 点；让 slow 从 a 点开始走，走 x 步，也会走到 b 点。

```cpp
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        if(head == nullptr)
            return nullptr;
        ListNode* fast = head;
        ListNode* slow = head;
        while(fast && fast -> next){
            slow = slow -> next;
            fast = fast -> next -> next;
            if(fast == slow){
                slow = head;
                while(slow != fast){
                    slow = slow -> next;
                    fast = fast -> next;
                }
                return slow;
            }
        }
        return nullptr;
    }
};
```

## 24. 反转链表

> 定义一个函数，输入一个链表的头结点，反转该链表并输出反转后链表的头结点。

[LeetCode 206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

定义三个指针。cur用来保存当前节点，pre用来保存当前节点的前一个节点，pNext用来保存当前节点的下一个节点，防止链表断链。

```cpp
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* pre = nullptr;
        ListNode* cur = head;
        while(cur){
            ListNode* pNext = cur -> next;
            cur -> next = pre;
            pre = cur;
            cur = pNext;
        }
        return pre;
    }
    
    //递归
    ListNode* reverseList2(ListNode* head) {
        if(head == nullptr || head -> next == nullptr)
            return head;
        ListNode* pNext = head -> next;
        ListNode* newHead = reverseList(pNext);
        pNext -> next = head;
        head -> next = nullptr;
        return newHead;
    }
};
```

## 25. 合并两个排序的链表

> 输入两个递增排序的链表，合并这两个链表并使新链表中的结点仍然是按照递增排序的。

[LeetCode 21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/)

```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
        ListNode head(-1);
        ListNode* cur = &head;
        if(l1 == nullptr) 
            return l2;
        if(l2 == nullptr) 
            return l1;
        while(l1 && l2){
            if(l1 -> val <= l2 -> val){
                cur -> next = l1;
                l1 = l1 -> next;
            }
            else{
                cur -> next = l2;
                l2 = l2 -> next;
            }
            cur = cur -> next;
        }
        cur -> next = l1 ? l1 : l2;
        return head.next;
    }
    
    //递归
    ListNode* mergeTwoLists2(ListNode* l1, ListNode* l2) {
        if(l1 == nullptr) 
            return l2;
        if(l2 == nullptr) 
            return l1;
        if(l1 -> val <= l2 -> val){
            l1 -> next = mergeTwoLists(l1 -> next, l2);
            return l1;
        }
        else{
            l2 -> next = mergeTwoLists(l1, l2 -> next);
            return l2;
        }
    }
    
};
```

## 26. 树的子结构

> 输入两棵二叉树A和B，判断B是不是A的子结构。

[LeetCode 572. Subtree of Another Tree](https://leetcode.com/problems/subtree-of-another-tree/)

第一步在树A中查找与树B根结点的值一样的节点R，第二步判断树A中以R为根结点的子树是不是和树B具有相同的结构。

另外考虑子结构的定义，剑指Offer上是只要包含B的结构就算；LeetCode上更严格一点，如图，虽然包含了B的结构，但是后面还有其他节点，此种情况不算子结构。

![image](https://ws3.sinaimg.cn/large/b0c02cdely1fyczri89l0j20kx099a9y.jpg)

```cpp
class Solution {
public:
    bool isSubtree(TreeNode* s, TreeNode* t) {
        if(s == nullptr || t == nullptr)
            return false;
        return isSame(s, t) || isSubtree(s -> left, t) || isSubtree(s -> right, t);
    }

private:
    bool isSame(TreeNode* s, TreeNode* t){
        /*//按照剑指Offer的子结构定义
          if(t == nullptr)
            return true;
          if(s == nullptr)
            return false;
        */
        //按照LeetCode的子结构定义
        if(s == nullptr && t == nullptr)
            return true;
        if(s == nullptr || t == nullptr)
            return false;
        if(s -> val != t -> val)
            return false;
        return isSame(s -> left, t -> left) && isSame(s -> right, t -> right);
    }
};
```

## 27. 二叉树的镜像

> 请完成一个函数，输入一棵二叉树，该函数输出它的镜像。

[LeetCode 226. Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/)

先前序遍历树的每个节点，如果遍历到的节点有子节点，就交换它的两个子节点。当交换完所有非叶节点的左右子节点之后，就得到了树的镜像。

```cpp
class Solution {
public:
    void Mirror(TreeNode *pRoot) {
        if(pRoot == nullptr)
            return;
        swap(pRoot -> left, pRoot -> right);
        Mirror(pRoot -> left);
        Mirror(pRoot -> right);
    }
};
```

## 28. 对称的二叉树

> 请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

[LeetCode 101. Symmetric Tree](https://leetcode.com/problems/symmetric-tree/)

针对前序遍历定义一种对称的遍历算法，即先遍历父节点，再遍历它的右子节点，最后遍历它的左子节点。通过比较二叉树的前序遍历序列和对称前序遍历序列来判断二叉树是不是对称的。

```cpp
class Solution {
public:
    bool isSymmetric(TreeNode* root) {
        return isSymmetricCore(root, root);
    }
    
private:
    bool isSymmetricCore(TreeNode* root1, TreeNode* root2){
        if(root1 == nullptr && root2 == nullptr)
            return true;
        if(root1 == nullptr || root2 == nullptr)
            return false;
        if(root1 -> val != root2 -> val)
            return false;
        return isSymmetricCore(root1 -> left, root2 -> right) && isSymmetricCore(root1 -> right, root2 -> left);
    }
};
```

## 29. 顺时针打印矩阵

> 输入一个矩阵，按照从外向里以顺时针的顺序依次打印出每一个数字。

[LeetCode 54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix/)

定义行的上下边界rowBegin，rowEnd，列的左右边界colBegin，colEnd。向右打印一行后rowBegin++，向下打印一列后colEnd--，向左打印一行后rowEnd--，向上打印一列后colBegin++，注意边界判断即可。

```cpp
class Solution {
public:
    vector<int> spiralOrder(vector<vector<int>>& matrix) {
        vector<int> res;
        if(matrix.empty() || matrix[0].empty())
            return res;
        int rowBegin = 0;
        int rowEnd = matrix.size() - 1;
        int colBegin = 0;
        int colEnd = matrix[0].size() - 1;
        while(rowBegin <= rowEnd && colBegin <= colEnd){
            //向右打印一行
            for(int i = colBegin; i <= colEnd; i++)
                res.push_back(matrix[rowBegin][i]);
            rowBegin++;
            //向下打印一列
            for(int i = rowBegin; i <= rowEnd; i++)
                res.push_back(matrix[i][colEnd]);
            colEnd--;
            //向左打印一行
            if(rowBegin <= rowEnd){
                for(int i = colEnd; i >= colBegin; i--)
                    res.push_back(matrix[rowEnd][i]);
            }
            rowEnd--;
            //向上打印一列
            if(colBegin <= colEnd){
                for(int i = rowEnd; i >= rowBegin; i--)
                    res.push_back(matrix[i][colBegin]);
            }
            colBegin++;
        }
        return res;
    }
};
```

## 30. 包含min函数的栈

> 定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的min函数。在该栈中，调用min，push及pop的时间复杂度都是O(1)。

[LeetCode 155. Min Stack](https://leetcode.com/problems/min-stack/)

定义两个栈，dataStack用来保存数据，minStack用来保存每次压入一个元素后的最小值。

```cpp
class MinStack {
public:
    /** initialize your data structure here. */
    MinStack() {
        
    }
    
    void push(int x) {
        dataStack.push(x);
        if(minStack.empty() || x < minStack.top())
            minStack.push(x);
        else
            minStack.push(minStack.top());
    }
    
    void pop() {
        dataStack.pop();
        minStack.pop();
    }
    
    int top() {
        return dataStack.top();
    }
    
    int getMin() {
        return minStack.top();
    }
    
private:
    stack<int> dataStack;
    stack<int> minStack;
};

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack obj = new MinStack();
 * obj.push(x);
 * obj.pop();
 * int param_3 = obj.top();
 * int param_4 = obj.getMin();
 */
```

## 31. 栈的压入、弹出序列

> 输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的的所有数字均不相等。例如，序列{1,2,3,4,5}是某栈的压栈序列，序列{4,5,3,2,1}是该压栈序列对应的一个弹出序列，但{4,5,3,1,2}就不可能是该压栈序列的弹出序列。

[LeetCode 946. Validate Stack Sequences](https://leetcode.com/problems/validate-stack-sequences/)

用一个栈sta来模拟压入弹出的过程，遍历压入序列，每次push一个元素到模拟栈sta中，判断栈顶元素是不是下一个弹出的数字，如果是，直接弹出，否则继续压栈。循环完毕后判断模拟栈是否为空即可。

```cpp
class Solution {
public:
    bool validateStackSequences(vector<int>& pushed, vector<int>& popped) {
        stack<int> sta;
        int i = 0;
        for(int x : pushed){
            sta.push(x);
            while(!sta.empty() && sta.top() == popped[i]){
                sta.pop();
                ++i;
            }
        }
        return sta.empty();
    }
};
```

## 32. 从上到下打印二叉树

> 不分行从上到下打印二叉树。从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

实质是考二叉树的层次遍历。需要用到队列，首先把根节点放入队列，接下来每次从队列的头部取出一个节点，遍历这个节点之后把它的子节点都依次放入队列。重复这个遍历过程，知道队列中的节点全部被遍历为止。

```cpp
class Solution {
public:
    vector<int> PrintFromTopToBottom(TreeNode* root) {
        vector<int> res;
        if(root == nullptr)
            return res;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty()){
            TreeNode* t = q.front();
            q.pop();
            res.push_back(t -> val);
            if(t -> left)
                q.push(t -> left);
            if(t -> right)
                q.push(t -> right);
        }
        return res;
    }
};
```

> 分行从上到下打印二叉树。从上到下打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

[LeetCode 102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/)

在开始遍历一层的节点时，当前队列中的节点数就是当前层的节点数。

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> res;
        if(root == nullptr)
            return res;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty()){
            int n = q.size();
            vector<int> level;
            for(int i = 0; i < n; i++){
                TreeNode* t = q.front();
                q.pop();
                level.push_back(t -> val);
                if(t ->left)
                    q.push(t -> left);
                if(t -> right)
                    q.push(t -> right);
            }
            res.push_back(level);
        }
        return res;
    }
};
```

> 之字形打印二叉树。请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。

[LeetCode 103. Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)

增加一个变量leftToRight来判断当前层是从左到右还是从右到左，当前层从右到左时从后往前增添元素。

```cpp
class Solution {
public:
    vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
        vector<vector<int>> res;
        if(root == nullptr)
            return res;
        queue<TreeNode*> q;
        q.push(root);
        bool leftToRight = true;
        while(!q.empty()){
            int n = q.size();
            vector<int> level(n);
            for(int i = 0; i < n; i++){
                TreeNode* t = q.front();
                q.pop();
                if(leftToRight)
                    level[i] = t -> val;
                else
                    level[n - 1 - i] = t -> val;
                if(t -> left)
                    q.push(t -> left);
                if(t -> right)
                    q.push(t -> right);
            }
            res.push_back(level);
            leftToRight = !leftToRight;
        }
        return res;
    }
};
```

## 33. 二叉搜索树的后序遍历序列

> 输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回true，否则返回false。假设输入的数组的任意两个数字互不相同。

在后序遍历得到的序列中，最后一个数字是树的根结点的值。数组中前面的数字可以分为两部分，第一部分是左子树节点的值，它们都比根结点的值小，第二部分是右子树节点的值，它们都比根结点的值大。递归重复这个判断过程，即可验证。

```cpp
class Solution {
public:
    bool VerifySquenceOfBST(vector<int> sequence) {
        if(sequence.empty())
            return false;
        return dfs(sequence, 0, sequence.size() - 1);
    }

private:
    bool dfs(vector<int>& sequence, int left, int right){
        if(right - left <= 1)
            return true;
        int root = sequence[right];
        int index = left;
        while(index < right && sequence[index] < root)
            index++;
        for(int i = index; i < right; i++)
            if(sequence[i] < root)
                return false;
        return dfs(sequence, left, index - 1) && dfs(sequence, index, right - 1);
    }
};
```

## 34. 二叉树中和为某一值的路径

> 输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。

[LeetCode 112. Path Sum](https://leetcode.com/problems/path-sum/)

[LeetCode 113. Path Sum II](https://leetcode.com/problems/path-sum-ii/)

递归回溯处理，当到达叶子节点的时候累加和刚好等于给定值则路径满足。

```cpp
class Solution {
public:
    vector<vector<int>> pathSum(TreeNode* root, int sum) {
        vector<vector<int>> res;
        vector<int> path;
        dfs(root, sum, res, path);
        return res;
    }
    
private:
    void dfs(TreeNode* root, int sum, vector<vector<int>>& res, vector<int>& path){
        if(root == nullptr)
            return;
        path.push_back(root -> val);
        if(root -> left == nullptr && root -> right == nullptr && sum == root -> val)
            res.push_back(path);
        dfs(root -> left, sum - root -> val, res, path);
        dfs(root -> right, sum - root -> val, res, path);
        path.pop_back();
    }
};
```

## 35. 复杂链表的复制

> 请实现函数ComplexListNode* Clone(ComplexListNode* pHead)，复制一个复杂链表。在复杂链表中，每个节点除了有一个m_pNext指针指向下一个节点，还有一个m_pSibling指针指向链表中的任意节点或者nullptr。

[LeetCode 138. Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/)

第一步，在每个节点的后面插入复制的节点。
![image](https://ws2.sinaimg.cn/large/b0c02cdely1fyjdjcw3vtj20ki04c0sl.jpg)

第二步，对复制节点的 random 链接进行赋值。
![image](https://wx3.sinaimg.cn/large/b0c02cdely1fyjdjvrnt8j20ko04z0sm.jpg)

第三步，拆分。
![image](https://ws2.sinaimg.cn/large/b0c02cdely1fyjdkby73oj20o605j0sm.jpg)

```cpp
/**
 * Definition for singly-linked list with a random pointer.
 * struct RandomListNode {
 *     int label;
 *     RandomListNode *next, *random;
 *     RandomListNode(int x) : label(x), next(NULL), random(NULL) {}
 * };
 */
class Solution {
public:
    RandomListNode *copyRandomList(RandomListNode *head) {
        if(head == nullptr)
            return nullptr;
        //复制节点
        RandomListNode* cur = head;
        while(cur != nullptr){
            RandomListNode* pNext = cur -> next;
            RandomListNode* copy = new RandomListNode(cur -> label);
            cur -> next = copy;
            copy -> next = pNext;
            cur = pNext;
        }
        //复制随机指针
        cur = head;
        while(cur != nullptr){
            if(cur -> random != nullptr)
                cur -> next -> random = cur -> random -> next;
            cur = cur -> next -> next;
        }
        //拆分
        cur = head;
        RandomListNode* dummy = new RandomListNode(-1);
        RandomListNode* p = dummy;
        while(cur != nullptr){
            RandomListNode* copy = cur -> next;
            cur -> next = copy -> next;
            cur = cur -> next;
            p -> next = copy;
            p = p -> next;
        }
        return dummy -> next;
    }
};
```

## 36. 二叉搜索树与双向链表

> 输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的节点，只能调整树中节点指针的指向。

![image](https://ws2.sinaimg.cn/large/b0c02cdely1fyjfd6vyvoj20eu05f0sm.jpg)

维护一个pre指针来记录当前已转换链表的最后一个节点，递归中序遍历二叉搜索树，当遍历转换到根节点时，它的左子树已经转换成一个排序的链表，并且处在链表中的最后一个节点是当前值最大的节点。将根节点和这个节点相互链接，接着遍历转换右子树。

```cpp
/*
struct TreeNode {
	int val;
	struct TreeNode *left;
	struct TreeNode *right;
	TreeNode(int x) :
			val(x), left(NULL), right(NULL) {
	}
};*/
class Solution {
public:
    TreeNode* Convert(TreeNode* pRootOfTree)
    {
        if(pRootOfTree == nullptr)
            return nullptr;
        TreeNode* pre = nullptr;
        convertCore(pRootOfTree, pre);
        TreeNode* head = pRootOfTree;
        while(head -> left)
            head = head -> left;
        return head;
    }

private:
    void convertCore(TreeNode* pNode, TreeNode*& pre){
        if(pNode == nullptr)
            return;
        convertCore(pNode -> left, pre);
        pNode -> left = pre;
        if(pre != nullptr)
            pre -> right = pNode;
        pre = pNode;
        convertCore(pNode -> right, pre);
    }
};
```

## 37. 序列化二叉树

> 请实现两个函数，分别用来序列化和反序列化二叉树。

[LeetCode 297. Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/)

前序遍历序列化二叉树，遇到空节点转化为字符$。反序列化时同样采用前序遍历,递归处理。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Codec {
public:

    // Encodes a tree to a single string.
    string serialize(TreeNode* root) {
        string res="";
        serializeCore(root, res);
        return res;
    }

    // Decodes your encoded data to tree.
    TreeNode* deserialize(string data) {
        if(data.empty())
            return nullptr;
        queue<string> node;
        string s = "";
        for(int i = 0; i < data.size(); i++){
            if(data[i] == ','){
                node.push(s);
                s.clear();
            }
            else
                s += data[i];
        }
        TreeNode* root = deserializeCore(node);
        return root;
    }
    
private:
    void serializeCore(TreeNode* root, string& res){
        if(root == nullptr){
            res += "$,";
            return;
        }
        res += to_string(root -> val) + ',';
        serializeCore(root -> left, res);
        serializeCore(root -> right, res);
    }
    
    TreeNode* deserializeCore(queue<string>& node){
        if(node.empty())
            return nullptr;
        string s = node.front();
        node.pop();
        if(s == "$")
            return nullptr;
        TreeNode* root = new TreeNode(stoi(s));
        root -> left = deserializeCore(node);
        root -> right = deserializeCore(node);
        return root;
    }
};

// Your Codec object will be instantiated and called as such:
// Codec codec;
// codec.deserialize(codec.serialize(root));
```

## 38. 字符串的排列

> 输入一个字符串，打印出该字符串中字符的所有排列。例如，输入字符串abc，则打印出由字符a，b，c所能排列出来的所有字符串abc，acb，bac，bca，cab和cba。

无重复元素的情况：

[LeetCode 46. Permutations](https://leetcode.com/problems/permutations/)

把字符串分为两部分，一部分是字符串的第一个字符，另一部分是字符串第一个字符以后的所有字符。将第一个字符和它后面的字符逐一替换，求剩下的全排列。递归这个过程即可得到全排列。

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> res;
        helper(res, nums, 0);
        return res;
    }

private:
    void helper(vector<vector<int>>& res, vector<int>& nums, int index){
        if(index == nums.size()){
            res.push_back(nums);
            return;
        }   
        for(int i = index; i < nums.size(); i++){
            swap(nums[index], nums[i]);
            helper(res, nums, index + 1);
            swap(nums[index], nums[i]);
        }
    }
};
```

有重复元素的情况：
[LeetCode 47. Permutations II](https://leetcode.com/problems/permutations-ii/)

去重的全排列就是从第一个数字起每个数分别与它后面非重复出现的数字交换。用编程的话描述就是第i个数与第j个数交换时，要求[i,j)中没有与第j个数相等的数。

```cpp
class Solution {
public:
    vector<vector<int>> permuteUnique(vector<int>& nums) {
        vector<vector<int>> res;
        helper(res, nums, 0);
        return res;
    }
    
private:
    void helper(vector<vector<int>>& res, vector<int>& nums, int index){
        if(index == nums.size()){
            res.push_back(nums);
            return;
        }
        unordered_set<int> s;
        for(int i = index; i < nums.size(); i++){
            if(s.find(nums[i]) != s.end())
                continue;
            s.insert(nums[i]);
            swap(nums[index], nums[i]);
            helper(res, nums, index + 1);
            swap(nums[index], nums[i]);
        }
            
    }
};
```

> 在1...n中选取k个数的组合

[LeetCode 77. Combinations](https://leetcode.com/problems/combinations/)

回溯法，选取第一个数，从后面的数再选k-1个数；不选第一个数，从后面的数再选k个数。

```cpp
class Solution {
public:
    vector<vector<int>> combine(int n, int k) {
        vector<vector<int>> res;
        vector<int> subset;
        helper(res, subset, 1, n, k);
        return res;
    }
    
private:
    void helper(vector<vector<int>>& res, vector<int>& subset, int start, int n, int k){
        if(subset.size() == k){
            res.push_back(subset);
            return;
        }
        for(int i = start; i <= n; i++){
            subset.push_back(i);
            helper(res, subset, i + 1, n, k);
            subset.pop_back();
        }
    }
};
```

> 无重复元素情况下，集合的所有子集

[LeetCode 78. Subsets](https://leetcode.com/problems/subsets/)

回溯法，同上面的思路。

```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<vector<int>> res;
        vector<int> subset;
        helper(res, subset, nums, 0);
        return res;
    }
    
private:
    void helper(vector<vector<int>>& res, vector<int>& subset, vector<int>& nums, int start){
        res.push_back(subset);
        for(int i = start; i < nums.size(); i++){
            subset.push_back(nums[i]);
            helper(res, subset, nums, i + 1);
            subset.pop_back();
        }
    }
};
```

> 有重复元素情况下，集合的所有子集

[LeetCode 90. Subsets II](https://leetcode.com/problems/subsets-ii/)

```cpp
class Solution {
public:
    vector<vector<int>> subsetsWithDup(vector<int>& nums) {
        vector<vector<int>> res;
        vector<int> subset;
        sort(nums.begin(), nums.end());
        helper(res, subset, nums, 0);
        return res;
    }
    
private:
    void helper(vector<vector<int>>& res, vector<int>& subset, vector<int>& nums, int start){
        res.push_back(subset);
        for(int i = start; i < nums.size(); i++){
            if(i > start && nums[i] == nums[i - 1])
                continue;
            subset.push_back(nums[i]);
            helper(res, subset, nums, i + 1);
            subset.pop_back();
        }
    }
};
```

> 在 8 * 8的国际象棋上摆放8个皇后，使其不能相互攻击，即任意两个皇后不得处在同一行，同一列或者同一对角线上。请问总共有多少种符合条件的摆法？

[LeetCode 51. N-Queens](https://leetcode.com/problems/n-queens/)

[LeetCode 52. N-Queens II](https://leetcode.com/problems/n-queens-ii/)

逐行遍历，对每一个位置检测同一列，同一对角线上之前是否有皇后出现。

求摆法：

```cpp
class Solution {
public:
    vector<vector<string>> solveNQueens(int n) {
        vector<vector<string>> res;
        vector<string> queen(n, string(n, '.'));
        solveNQueensCore(res, queen, 0, n);
        return res;
    }
    
private:
    void solveNQueensCore(vector<vector<string>>& res, vector<string>& queen, int row, int n){
        if(row == n){
            res.push_back(queen);
            return;
        }
        for(int col = 0; col < n; col++){
            if(isValid(queen, row, col, n)){
                queen[row][col] = 'Q';
                solveNQueensCore(res, queen, row + 1, n);
                queen[row][col] = '.';
            }
        }
    }
    
    bool isValid(vector<string>& queen, int row, int col, int n){
        //检查同一列是否放置了皇后
        for(int i = 0; i < row; i++)
            if(queen[i][col] == 'Q')
                return false;
        //检查左对角线是否放置了皇后
        for(int i = row - 1, j = col - 1; i >=0 && j >=0; i--, j--)
            if(queen[i][j] == 'Q')
                return false;
        //检查右对角线是否放置了皇后
        for(int i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++)
            if(queen[i][j] == 'Q')
                return false;
        return true;
    }
};
```

求摆放总数:

```cpp
class Solution {
public:
    int totalNQueens(int n) {
        vector<bool> cols(n, false);
        vector<bool> diags1(2 * n - 1, false);
        vector<bool> diags2(2 * n - 1, false);
        int count = 0; 
        dfs(0, count, n, cols, diags1, diags2);
        return count;
    }
    
private:
    void dfs(int row, int& count, int n, vector<bool>& cols, vector<bool>& diags1, vector<bool>& diags2){
        if(row == n){
            count++;
            return;
        }
        for(int col = 0; col < n; col++){
            if(cols[col] || diags1[row - col + n - 1] || diags2[row + col])
                continue;
            cols[col] = true, diags1[row - col + n - 1] = true, diags2[row + col] = true;
            dfs(row + 1, count, n, cols, diags1, diags2);
            cols[col] = false, diags1[row - col + n - 1] = false, diags2[row + col] = false;
        }
    }
};
```

## 39. 数组中出现次数超过一半的数字

> 数组中有一个数字出现的次数超过数组长度的一半，请找出这个数字。例如，输入一个长度为9的数组{1,2,3,2,2,2,5,4,2}。由于数字2在数组中出现了5次，超过数组长度的一半，因此输出2.

[LeetCode 169. Majority Element](https://leetcode.com/problems/majority-element/)

考虑在遍历数组的时候保存两个值：一个是数组中的一个数字，另一个是次数。当遍历到下一个数字的时候，如果下一个数字和之前保存的数字相同，则次数加1，如果不同，则次数减1。如果次数减为0，则保存下一个数字，并把次数设为1。由于要找的数字出现的次数比其他所有数字出现的次数之和还要多，则要找的数字肯定是最后一次把次数设为1时对应的数字。

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int res = nums[0];
        int count = 1;
        for(int i = 1; i < nums.size(); i++){
            if(count == 0){
                res = nums[i];
                count = 1;
            }
            else if(nums[i] == res)
                count ++;
            else
                count --;
        }
        return res;
    }
};
```

## 40. 最小的k个数

> 输入n个整数，找出其中最小的k个数。例如，输入4,5,1,6,2,7,3,8这8个数字，则最小的4个数字是1,2,3,4。

[LeetCode 215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)

[LeetCode 347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/)

**快速选择算法**

快速排序的 partition() 方法，会返回一个整数 j 使得 a[l..j-1] 小于等于 a[j]，且 a[j+1..h] 大于等于 a[j]，此时 a[j] 就是数组的第 j 大元素。可以利用这个特性找出数组的第 K 个元素，这种找第 K 个元素的算法称为快速选择算法。

* 复杂度：O(N) + O(1)
* 只有当允许修改数组元素时才可以使用

```cpp
class Solution {
public:
    vector<int> GetLeastNumbers_Solution(vector<int> input, int k) {
        vector<int> res;
        if(input.empty() || k <= 0 || k > input.size())
            return res;
        int left = 0, right = input.size() - 1;
        int index = 0;
        while(left <= right){
            index = partition(input, left, right);
            if(index == k - 1)
                break;
            else if(index < k - 1)
                left = index + 1;
            else
                right = index - 1;
        }
        res = vector<int>(input.begin(), input.begin() + k);
        return res;
    }
    
private:
    int partition(vector<int>& input, int left, int right){
        int pivot = input[left];
        int i = left;
        int j = left + 1;
        while(j <= right){
            if(input[j] <= pivot)
                swap(input[++i], input[j]);
            j++;
        }
        swap(input[i], input[left]);
        return i;
    }
};
```

**优先级队列维护最大堆**

* 复杂度：O(NlogK) + O(K)
* 特别适合处理海量数据

```cpp
class Solution {
public:
    vector<int> GetLeastNumbers_Solution(vector<int> input, int k) {
        vector<int> res;
        priority_queue<int> pq;
        if(input.empty() || k <= 0 || k > input.size())
            return res;
        for(auto a : input){
            pq.push(a);
            if(pq.size() > k)
                pq.pop();
        }
        while(!pq.empty()){
            res.push_back(pq.top());
            pq.pop();
        }
        return res;
    }
};
```

## 41. 数据流中的中位数

> 如何得到一个数据流中的中位数？如果从数据流中读出奇数个数值，那么中位数就是所有数值排序之后位于中间的数值。如果从数据流中读出偶数个数值，那么中位数就是所有数值排序之后中间两个数的平均值。

[LeetCode 295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/)

维护两个堆，最大堆left用来存储左半边的元素，最小堆right用来存放右半边的元素。为了保持平衡，left的元素个数最多比right的元素个数多一个。

* 首先将数字加到最大堆left中，因为接收到的数可能比最小堆right中的一些数据要大，所以必须调整最小堆，把最大堆中最大的数字拿出来插入最小堆。

* 经过上面的调整后，最小堆中的数据数目可能比最大堆中的要多，因此需要做一次修正，将最小堆中最小的数字拿出来插入到最大堆中。

```cpp
class MedianFinder {
public:
    /** initialize your data structure here. */
    MedianFinder() {
        
    }
    
    void addNum(int num) {
        left.push(num);
        right.push(left.top());
        left.pop();
        if(left.size() < right.size()){
            left.push(right.top());
            right.pop();
        }
    }
    
    double findMedian() {
        return left.size() > right.size() ? left.top() : (left.top() + right.top()) / 2.0;
    }
    
private:
    priority_queue<int> left;
    priority_queue<int, vector<int>, greater<int>> right;
};

/**
 * Your MedianFinder object will be instantiated and called as such:
 * MedianFinder obj = new MedianFinder();
 * obj.addNum(num);
 * double param_2 = obj.findMedian();
 */
```

## 42. 连续子数组的最大和

> 输入一个整型数组，数组里面有正数也有负数。数组中的一个或连续多个整数组成一个子数组。求所有子数组的和的最大值。要求时间复杂度为O(n)。

[LeetCode 53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)

动态规划，设dp[i]表示以第i个数字结尾的子数组的最大和。

$$
f(n) =
\begin{cases}
nums[i],  & \text{i = 0 或者 dp[i - 1] <= 0} \\
dp[i - 1] + nums[i], & \text{i > 0 并且 dp[i - 1] > 0}
\end{cases}
$$

```cpp
class Solution {
public:
    int maxSubArray(vector<int>& nums) {
        if(nums.empty())
            return 0;
        vector<int> dp(nums.size());
        dp[0] = nums[0];
        int maxSum = dp[0];
        for(int i = 1; i < nums.size(); i++){
            if(dp[i - 1] <= 0)
                dp[i] = nums[i];
            else
                dp[i] = dp[i - 1] + nums[i];
            maxSum = max(maxSum, dp[i]);
        }
        return maxSum;
    }
};
```

## 43. 1~n整数中1出现的次数

> 输入一个整数n，求1~n这n个整数的十进制表示中1出现的次数。例如，输入12,1到12这些整数中包含1的数字有1,10,11和12，1一共出现了5次。

[LeetCode 233. Number of Digit One](https://leetcode.com/problems/number-of-digit-one/)

设N = abcde ,其中abcde分别为十进制中各位上的数字。

如果要计算百位上1出现的次数，它要受到3方面的影响：百位上的数字，百位以下（低位）的数字，百位以上（高位）的数字。

* 如果百位上数字为0，百位上可能出现1的次数由更高位决定。比如：12013，则可以知道百位出现1的情况可能是：100 ~ 199，1100 ~ 1199,2100 ~ 2199，...，11100 ~ 11199，一共1200个。可以看出是由更高位数字（12）决定，并且等于更高位数字（12）乘以当前位数（100）。

* 如果百位上数字为1，百位上可能出现1的次数不仅受更高位影响还受低位影响。比如：12113，则可以知道百位受高位影响出现的情况是：100 ~ 199，1100 ~ 1199,2100 ~ 2199，....，11100 ~ 11199，一共1200个。和上面情况一样，并且等于更高位数字（12）乘以 当前位数（100）。但同时它还受低位影响，百位出现1的情况是：12100 ~ 12113,一共14个，等于低位数字（13）+1。

* 如果百位上数字大于1（2 ~ 9），则百位上出现1的情况仅由更高位决定，比如12213，则百位出现1的情况是：100 ~ 199,1100 ~ 1199，2100 ~ 2199，...，11100 ~ 11199,12100 ~ 12199,一共有1300个，并且等于更高位数字+1（12+1）乘以当前位数（100）。

```cpp
class Solution {
public:
    int countDigitOne(int n) {
        if(n <= 0)
            return 0;
        int  count = 0;
        long long factor = 1;
        while(n / factor){
            long long cur = (n / factor) % 10;
            long long high = (n / factor) / 10;
            long long low = n - (n / factor) * factor;
            if(cur == 0)
                count += high * factor;
            else if(cur == 1)
                count += high * factor + low + 1;
            else
                count += (high + 1) * factor;
            factor *= 10;
        }
        return count;
    }
};
```

精简写法：

三种情况可以统一为(n / factor + 8) / 10 * factor + (n / factor % 10 == 1) * (n % factor +1);

```cpp
class Solution {
public:
    int countDigitOne(int n) {
        if(n <= 0)
            return 0;
        int  count = 0;
        for(long long i = 1; i <= n; i *= 10){
            long long a = n / i, b = n % i;
            count += (a + 8) / 10 * i + (a % 10 == 1) * (b + 1);
        }
        return count;
    }
};
```

## 44. 数字序列中某一位的数字

> 数字以0123456789101112131415...的格式序列化到一个字符序列中。在这个序列中，第5位（从0开始计数）是5，第13位是1，第19位是4，等等。请写一个函数，求任意第n位对应的数字。

[LeetCode 400. Nth Digit](https://leetcode.com/problems/nth-digit/)

分成三步来求。第一步确定数字有几位数，第二步确定具体的数字，第三步确定数字的具体哪一位。

```cpp
class Solution {
public:
    int digitAtIndex(int n) {
        if(n < 0)
            return -1;
        //确定数字有几位数
        int digits = 1;
        while(true){
            long long total = totalNumberOfDigits(digits);
            if(n < total * digits)
                break;
            n -= total * digits;
            digits++;
        }
        //确定数字是哪个数
        int number = beginNumberOfDigits(digits) + n / digits;
        //找到具体的那一位
        int indexFromRight = digits - n % digits;
        for(int i = 1; i < indexFromRight; i++)
            number /= 10;
        return number % 10;
    }
    
private:
    //有digits位数的数字总数，1 -> 10(0 ~ 9)， 2 -> 90(10 ~ 99)， 3 -> 900(100 ~ 999) ...
    long long totalNumberOfDigits(int digits){
        if(digits == 1)
            return 10;
        return 9 * pow(10, digits - 1);
    }
    
    //digits位数的第一个数字
    int beginNumberOfDigits(int digits){
        if(digits == 1)
            return 0;
        return pow(10, digits - 1);
    }
};
```

## 45. 把数组排成最小的数

> 输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。例如，输入数组{3,32,321}，则打印出这3个数字能排成的最小数字321323。

[LeetCode 179. Largest Number](https://leetcode.com/problems/largest-number/)

可以看成是一个排序问题，在比较两个字符串 S1 和 S2 的大小时，应该比较的是 S1+S2 和 S2+S1 的大小，如果 S1+S2 < S2+S1，那么应该把 S1 排在前面，否则应该把 S2 排在前面。

```cpp
class Solution {
public:
    string PrintMinNumber(vector<int> numbers) {
        if(numbers.empty())
            return "";
        auto cmp = [](const int& a, const int& b){return to_string(a) + to_string(b) < to_string(b) + to_string(a);};
        sort(numbers.begin(), numbers.end(), cmp);
        string res = "";
        for(auto a : numbers)
            res += to_string(a);
        return res;
    }
};
```

## 46. 把数字翻译成字符串

> 给定一个数字，我们按照如下规则把它翻译为字符串：0翻译成"a"，1翻译成"b"，......11翻译成"1"，.....，25翻译成"z"。一个数字可能有多个翻译。例如，12258有5种不同的翻译，分别是"bccfi"，"bwfi"，"bczi"，"mcfi"和"mzi"。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。

[LeetCode 91. Decode Ways](https://leetcode.com/problems/decode-ways/)

定义dp[i]表示从第i位数字开始的不同翻译的数目，则有

$$
dp[i] = dp[i + 1] + g(i, i + 1) * dp[i + 2] 
$$

当第i位和第i + 1位两位数字拼接起来的数字在10~25的范围内时，函数g(i, i + 1)的值为1；否则为0。

```cpp
class Solution {
public:
    int getTranslationCount(string s) {
        if(s.empty())
            return 0;
        int n = s.size();
        vector<int> dp(n + 1);
        dp[n] = 1;
        dp[n - 1] = 1;
        for(int i = n - 2; i >= 0; i--){
            dp[i] = dp[i + 1];
            if(s[i] == '1' || (s[i] == '2' && s[i + 1] <= '5'))
                dp[i] += dp[i + 2];
        }
        return dp[0];
    }
};
```

## 47. 礼物的最大价值

> 在一个m * n的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格，直到到达棋盘的右下角。给定一个棋盘及其上面的礼物，请计算你最多能拿到多少价值的礼物?

[LeetCode 64. Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/)

定义dp[i][j]表示到达坐标为（i, j）的格子时能拿到的礼物总和的最大值。则有

$$
dp(i,j) = max(dp(i - 1, j), dp(i, j - 1)) + grid[i, j]
$$

```cpp
class Solution {
public:
    int getMaxValue(vector<vector<int>>& grid) {
        if(grid.empty() || grid[0].empty())
            return 0;
        int m = grid.size();
        int n = grid[0].size();
        vector<vector<int>> dp(m, vector<int>(n, 0));
        dp[0][0] = grid[0][0];
        for(int i = 1; i < n; i++)
            dp[0][i] = dp[0][i - 1] + grid[0][i];
        for(int i = 1; i < m ; i++)
            dp[i][0] = dp[i - 1][0] + grid[i][0];
        for(int i = 1; i < m; i++)
            for(int j = 1; j < n ; j++)
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1]) + grid[i][j];
        return dp[m - 1][n - 1];
    }
};
```

## 48. 最长不含重复字符的子字符串

> 请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。假设字符串中只包含'a' ~ 'z'的字符。例如，在字符串"arabcacfr"中，最长的不含重复字符的子字符串是"acfr"，长度为4。

[LeetCode 3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

维护一个<char, index>的 hashmap 来存储字符和对应的下标，采用滑动窗口双指针的思想，一个指针 start 用来维护最长不重复子串的左边界，另一个指针 i 用来遍历字符串及充当右边界，同时更新 s[i] 在 hashmap 中的值。如果 s[i] 已经在 hashmap 中，且s[i]的hash值大于等于start，即 s[i] 在之前维护的最长不重复子串内，则维护左指针到hash[s[i]] + 1 。

因为都是字符，所以可以考虑用数组来代替hashmap。

```cpp
class Solution {
public:
    int longestSubstringWithoutDuplication(string s) {
        if(s.empty())
            return 0;
        vector<int> hash(26, -1);
        int start = 0;
        int res = 0;
        for(int i = 0; i < s.size(); i++){
            if(hash[s[i] - 'a'] >= start)
                start = hash[s[i] - 'a'] + 1;
            hash[s[i] - 'a'] = i;
            res = max(res, i - start + 1);
        }
        return res;
    }
};
```

## 49. 丑数

> 我们把只包含因子2，3和5的数称作丑数。求按从小到大的顺序的第1500个丑数。例如，6，8都是丑数，但14不是，因为它包含因子7。习惯上我们把1当做第一个丑数。

[LeetCode 263. Ugly Number](https://leetcode.com/problems/ugly-number/)

[LeetCode 264. Ugly Number II](https://leetcode.com/problems/ugly-number-ii/)

[LeetCode 313. Super Ugly Number](https://leetcode.com/problems/super-ugly-number/)

由于丑数的因子也必定是丑数，它一定是某个丑数乘2、3、5得到的，因此我们可以采用动态规划的思想，利用前面已经得到的丑数序列来得到之后的丑数，而问题的关键在于如何确定状态转移方程。由于小的丑数乘5不一定比大的丑数乘2要小，我们没法直接使用目前最大的丑数来乘2、3、5顺序得到更大的丑数。不过，我们可以确定的是，小的丑陋数乘2，肯定小于大的丑陋数乘2。所以我们使用三个指针，分别记录乘2、3、5得出的目前最大丑陋数，而新的丑数就是这三个目前最大丑数中最小的那个，那么就需要更新被选中的丑数的指针，获得新的三个目前最大丑数，依次类推，从而得到最终结果。

```cpp
class Solution {
public:
    int nthUglyNumber(int n) {
        if(n < 7)
            return n;
        vector<int> res(n);
        res[0] = 1;
        int p2 = 0, p3 = 0, p5 = 0;
        for(int i = 1; i < n ; i++){
            res[i] = min(min(res[p2] * 2, res[p3] * 3), res[p5] * 5);
            if(res[i] == res[p2] * 2)
                ++p2;
            if(res[i] == res[p3] * 3)
                ++p3;
            if(res[i] == res[p5] * 5)
                ++p5;
        }
        return res[n - 1];
    }
};
```

## 50. 第一个只出现一次的字符

> 字符串中第一个只出现一次的字符。在字符串中找出第一个只出现一次的字符。如输入"abaccdeff"，则输出'b'。

[LeetCode 387. First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string/)

利用哈希表统计频度，两次遍历即可。

```cpp
class Solution {
public:
    int firstUniqChar(string s) {
        unordered_map<char, int> hash;
        for(auto c : s)
            hash[c]++;
        for(int i = 0; i < s.size(); i++)
            if(hash[s[i]] == 1)
                return i;
        return -1;
    }
};
```

> 字符流中第一个只出现一次的字符。请实现一个函数，用来找出字符流中第一个只出现一次的字符。例如，当从字符流中只读出前两个字符"go"时，第一个只出现一次的字符是'g'，当从该字符流中读出前6个字符"google"时，第一个只出现一次的字符是'l'。

利用哈希表来统计频度，用一个字符串来保存读出的字符。

```cpp
class Solution
{
public:
  //Insert one char from stringstream
    void Insert(char ch)
    {
        s += ch;
        hash[ch]++;
    }
  //return the first appearence once char in current stringstream
    char FirstAppearingOnce()
    {
        for(auto c : s)
            if(hash[c] == 1)
                return c;
        return '#';
    }

 private:
    unordered_map<char, int> hash;
    string s;
};
```

## 51. 数组中的逆序对

> 在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数。例如，在数组{7,5,6,4}中，一共存在5个逆序对，分别是(7,6)，(7,5)，(7,4)，(6,4)和(5,4)。

采用归并排序的思想，先把数组分隔成子数组，投统计出子数组内部的逆序对的数目，然后再统计出两个相邻子数组之间的逆序对的数目。在统计逆序对的过程中，还需要对逆序对进行排序。

```cpp
class Solution {
public:
    int InversePairs(vector<int> data) {
        if(data.empty())
            return 0;
        mergeSort(data, 0, data.size() - 1);
        return count % 1000000007;
    }
    
private:
    long long count = 0;
    
    void mergeSort(vector<int>& data, int left, int right){
        if(left >= right)
            return;
        int mid = left + (right - left) / 2;
        mergeSort(data, left, mid);
        mergeSort(data, mid + 1, right);
        merge(data, left, mid, right);
    }
    
    void merge(vector<int>& data, int left, int mid, int right){
        vector<int> tmp;
        int i = left, j = mid + 1;
        while(i <= mid && j <= right){
            if(data[i] <= data[j])
                tmp.push_back(data[i++]);
            else{
                tmp.push_back(data[j++]);
                count += mid - i + 1;
            }
        }
        while(i <= mid)
            tmp.push_back(data[i++]);
        while(j <= right)
            tmp.push_back(data[j++]);
        for(int k = 0; k < tmp.size(); k++)
            data[left + k] = tmp[k];
    }
    
};
```

## 52. 两个链表的第一个公共节点

> 输入两个链表，找出它们的第一个公共节点。

[LeetCode 160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/)

设 A 的长度为 a + c，B 的长度为 b + c，其中 c 为尾部公共部分长度，可知 a + c + b = b + c + a。

当访问链表 A 的指针访问到链表尾部时，令它从链表 B 的头部重新开始访问链表 B；同样地，当访问链表 B 的指针访问到链表尾部时，令它从链表 A 的头部重新开始访问链表 A。这样就能控制访问 A 和 B 两个链表的指针能同时访问到交点。

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        if(headA == nullptr || headB == nullptr)
            return nullptr;
        ListNode* p1 = headA;
        ListNode* p2 = headB;
        while(p1 != p2){
            p1 = (p1 == nullptr) ? headB : p1 -> next;
            p2 = (p2 == nullptr) ? headA : p2 -> next;
        }
        return p1;
    }
};
```

## 53. 在排序数组中查找数字

> 数字在排序数组中出现的次数。统计一个数字在排序数组中出现的次数。例如，输入排序数组{1,2,3,3,3,3,4,5}和数字3，由于3在这个数组中出现了4次，因此输出4。

[LeetCode 34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

参考lower_bound的二分查找写法，找到第一个大于等于k+1的数和第一个大于等于k的数，相减即可得到k出现的次数。

```cpp
class Solution {
public:
    int GetNumberOfK(vector<int> data ,int k) {
        if(data.empty())
            return 0;
        int start = binarySearch(data, k);
        if(start == data.size() || data[start] != k)
            return 0;
        return binarySearch(data, k + 1) - start;
    }
    
private:
    //[left...right)
    int binarySearch(vector<int>& data, int k){
        int left = 0, right = data.size();
        while(left < right){
            int mid = left + (right - left) / 2;
            if(data[mid] < k)
                left = mid + 1;
            else
                right = mid;
        }
        return left;
    }
};
```

> 0 ~ n - 1中缺失的数字。一个长度为n - 1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0 ~ n - 1内。在范围0 ~ n - 1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。

[LeetCode 268. Missing Number](https://leetcode.com/problems/missing-number/)

问题可以转换成在排序数组中找出第一个值和下标不相等的元素。利用二分查找，如果中间元素的值和下标相等，则下一轮只需要查找右半边。如果中间元素的值和下标不相等，并且它前面一个元素和它的下标相等，则中间这个数字即为所求。如果中间元素的值和下标不相等，并且它前面一个元素和它的下标不相等，则下一轮只需要查找左半边。

```cpp
class Solution {
public:
    int missingNumber(vector<int>& nums) {
		int left = 0, right = nums.size() - 1;
		while(left <= right){
			int mid = left + (right - left) / 2;
			if(mid == nums[mid])
                left = mid + 1;
            else if(mid == 0 || nums[mid - 1] == mid - 1)
                return mid;
            else
                right = mid - 1;
        }
        return left;
    }
};
```

> 数组中数值和下标相等的元素。假设一个单调递增的数组里的每个元素都是整数并且是唯一的。请编程实现一个函数，找出数组中任意一个数值等于其下标的元素。例如，在数组{-3,-1,1,3,5}中，数字3和它的下标相等。

二分查找即可。

```cpp
class Solution {
public:
    int getNumberSameAsIndex(vector<int>& nums) {
        int left = 0, right = nums.size() - 1;
        while(left <= right){
            int mid = left + (right - left) / 2;
            if(nums[mid] == mid)
                return mid;
            else if(nums[mid] > mid)
                right = mid - 1;
            else
                left = mid + 1;
        }
        return -1;
    }
};
```

## 54. 二叉搜索树的第K小节点

> 给定一棵二叉搜索树，请找出其中第k小的节点。

[LeetCode 230. Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/)

利用中序遍历，递归即可。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        int res = 0;
        dfs(root, k, res);
        return res;
    }

private:
    void dfs(TreeNode* root, int& k, int& res){
        if(root == nullptr)
            return;
        dfs(root -> left, k, res);
        --k;
        if(k == 0){
            res = root -> val;
            return;
        }
        dfs(root -> right, k, res);
    }
};
```

## 55. 二叉树的深度

> 二叉树到的深度。输入一棵二叉树的根结点，求该树的深度。从根节点到叶节点依次经过的节点形成树的一条路径，最长路径的长度为树的深度。

[LeetCode 104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/)

递归遍历即可。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    int maxDepth(TreeNode* root) {
        return root == nullptr ? 0 : 1 + max(maxDepth(root -> left), maxDepth(root -> right)); 
    }
};
```

> 平衡二叉树。输入一棵二叉树的根结点，判断该树是不是平衡二叉树。如果某二叉树中任意节点的左，右子树的深度相差不超过1，那么它就是一棵平衡二叉树。

[LeetCode 110. Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/)

从下往上遍历，如果子树是平衡二叉树，则返回子树的高度；如果发现子树不是平衡二叉树，则直接停止遍历，这样至多只对每个结点访问一次。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    bool isBalanced(TreeNode* root) {
        return height(root) != -1;
    }
    
private:
    int height(TreeNode* root){
        if(root == nullptr)
            return 0;
        int left = height(root -> left);
        if(left == -1)
            return -1;
        int right = height(root -> right);
        if(right == -1)
            return -1;
        if(abs(left - right) > 1)
            return -1;
        return 1 + max(left, right);
    }
    
};
```

## 56. 数组中数字出现的次数

> 数组中只出现一次的两个数字。一个整型数组里除两个数字之外，其他数字都出现了两次。请写程序找出这两个只出现一次的数字。要求时间复杂度为O(n)，空间复杂度为O(1)。

[LeetCode 136. Single Number](https://leetcode.com/problems/single-number/)

[LeetCode 260. Single Number III](https://leetcode.com/problems/single-number-iii/)

两个不相等的元素在位级表示上必定会有一位存在不同，将数组的所有元素异或得到的结果为不存在重复的两个元素异或的结果。

diff &= -diff 得到出 diff 最右侧不为 0 的位，也就是不存在重复的两个元素在位级表示上最右侧不同的那一位，利用这一位就可以将两个元素区分开来。

```cpp
class Solution {
public:
    vector<int> singleNumber(vector<int>& nums) {
        vector<int> res(2, 0);
        int diff = 0;
        for(auto num : nums)
            diff ^= num;
        diff &= -diff;
        for(auto num : nums){
            if((num & diff) == 0)
                res[0] ^= num;
            else
                res[1] ^= num;
        }
        return res;
    }
};
```

> 数组中唯一只出现一次的数字。在一个数组中除一个数字只出现一次之外，其他数字都出现了三次。请找出那个只出现一次的数字。

[LeetCode 137. Single Number II](https://leetcode.com/problems/single-number-ii/)

解法一：

如果一个数字出现三次，那么它的二进制表示的每一位也出现三次。把所有数字的二进制表示的每一位都加起来，如果某一位的和能被3整除，那么那个只出现一次的数字的二进制表示中对应的那一位是0，否则是1。

```cpp
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        int res = 0;
        for(int i = 0; i < 32; i++){
            int count = 0;
            for(auto a : nums)
                count += (a >> i) & 0x1;
            if(count % 3)
                res += (1 << i);
        }
        return res;
    }
};
```

解法二：

定义两个变量a和b，当遍历nums的时候，对于重复元素x，第一次碰到x的时候，我们会将x赋给a，第二次碰到后再赋给b，第三次碰到就全部消除。

```cpp
class Solution {
public:
    //  a b
    //0 0 0
    //1 x 0
    //2 0 x
    //3 0 0
    int singleNumber(vector<int>& nums) {
        int a = 0, b = 0;
        for(auto x : nums){
            a = (a ^ x) & ~b;
            b = (b ^ x) & ~a; 
        }
        return a;
    }
};
```

## 57. 和为s的数字

> 和为s的两个数字。输入一个递增排序的数组和一个数字s，在数组中查找两个数，使得它们的和正好是s。如果有多对数字的和等于s，则任意输出一对即可。

[LeetCode 167. Two Sum II - Input array is sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)

使用双指针，一个指针指向元素较小的值，一个指针指向元素较大的值。指向较小元素的指针从头向尾遍历，指向较大元素的指针从尾向头遍历。

* 如果两个指针指向元素的和 sum == target，那么得到要求的结果；
* 如果 sum > target，移动较大的元素，使 sum 变小一些；
* 如果 sum < target，移动较小的元素，使 sum 变大一些。

```cpp
class Solution {
public:
    vector<int> FindNumbersWithSum(vector<int> array,int sum) {
        vector<int> res;
        int left = 0, right = array.size() - 1;
        while(left < right){
            int cur = array[left] + array[right];
            if(cur == sum){
                res.push_back(array[left]);
                res.push_back(array[right]);
                break;
            }
            else if(cur < sum)
                left++;
            else
                right--;
        }
        return res;
    }
};
```

> 和为s的连续正数序列。输入一个正数s，打印出所有和为s的连续正数序列（至少含有两个数）。例如，输入15，由于1 + 2 + 3 + 4 + 5 = 4 + 5 + 6 = 7 + 8 = 15，所以打印出3个连续序列1 ~ 5，4 ~ 6和7 ~ 8。

[LeetCode 829. Consecutive Numbers Sum](https://leetcode.com/problems/consecutive-numbers-sum/)

利用滑动窗口，双指针求解。

```cpp
class Solution {
public:
    vector<vector<int> > FindContinuousSequence(int sum) {
        vector<vector<int>> res;
        int start = 1, end = 2;
        while(start < end){
            int total = (start + end) * (end - start + 1) / 2;
            if(total == sum){
                vector<int> tmp;
                for(int i = start; i <= end; i++)
                    tmp.push_back(i);
                res.push_back(tmp);
                start++;
            }
            else if(total < sum)
                end++;
            else
                start++;
        }
        return res;
    }
};
```

## 58. 翻转字符串

> 翻转单词顺序。输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。为简单起见，标点符号和普通字母一样处理。例如输入字符串"I am a student. "，则输出"student. a am I"。

[LeetCode 151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string/)

先旋转每个单词，再旋转整个字符串。

```cpp
class Solution {
public:
    string ReverseSentence(string str) {
        int start = 0, index = 0;
        while(index <= str.size()){
            if(index == str.size() || str[index] == ' '){
                reverse(str.begin() + start, str.begin() + index);
                start = index + 1;
            }
            index++;
        }
        reverse(str.begin(), str.end());
        return str;
    }
};
```

> 左旋转字符串。字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。请定义一个函数实现字符串左旋转操作的功能。比如输入字符串"abcdefg"和数字2，该函数将返回左旋转2位得到的结果"cdefgab"。

[LeetCode 189. Rotate Array](https://leetcode.com/problems/rotate-array/)

三次翻转。

```cpp
class Solution {
public:
    string LeftRotateString(string str, int n) {
        if(str.empty() || n <= 0)
            return str;
        int m = n % str.size();
        reverse(str.begin(), str.begin() + m);
        reverse(str.begin() + m, str.end());
        reverse(str.begin(), str.end());
        return str;
    }
};
```

## 59. 队列的最大值

> 滑动窗口的最大值。给定一个数组和滑动窗口的大小，请找出所有滑动窗口里的最大值。例如，如果输入数组{2, 3, 4, 2, 6, 2, 5, 1}及滑动窗口的大小3，那么一共存在6个滑动窗口，它们的最大值分别为{4, 4, 6, 6, 6, 5}。

[LeetCode 239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)

用一个双端队列，队首保存当前窗口的最大值的下标，当窗口滑动一次

1. 判断队首数字是否已经从窗口滑出，若滑出将队首丢掉
2. 新增加的值从队尾开始比较，把所有比他小的值丢掉，然后将该值的下标加入队尾

```cpp
class Solution {
public:
    vector<int> maxSlidingWindow(vector<int>& nums, int k) {
        vector<int> res;
        deque<int> q;
        for(int i = 0; i < nums.size(); i++){
            while(!q.empty() && i - q.front() >= k)
                q.pop_front();
            while(!q.empty() && nums[i] >= nums[q.back()])
                q.pop_back();
            q.push_back(i);
            if(i >= k - 1)
                res.push_back(nums[q.front()]);
        }
        return res;
    }
};
```

> 队列的最大值。请定义一个队列并实现函数max得到队列里的最大值，要求函数max，push_back和pop_front()的时间复杂度都是O(1)。

利用上题的解法实现带max函数的队列。

```cpp
template<typename T> class QueueWithMax
{
public:
	QueueWithMax() : currentIndex(0)
	{
	}

	void push_back(T number)
	{
		while (!maximums.empty() && number >= maximums.back().number)
			maximums.pop_back();

		InternalData internalData = { number, currentIndex };
		data.push_back(internalData);
		maximums.push_back(internalData);

		++currentIndex;
	}

	void pop_front()
	{
		if (maximums.empty())
			throw new exception("queue is empty");

		if (maximums.front().index == data.front().index)
			maximums.pop_front();

		data.pop_front();
	}

	T max() const
	{
		if (maximums.empty())
			throw new exception("queue is empty");

		return maximums.front().number;
	}

private:
	struct InternalData
	{
		T number;
		int index;
	};

	deque<InternalData> data;
	deque<InternalData> maximums;
	int currentIndex;
};
```

## 60. n个骰子的点数

> 把n个骰子扔在地上，所有骰子朝上一面的点数之和为s。输入n，打印出s的所有可能的值出现的概率。

考虑用动归，数组dp[i][j]表示用i个骰子扔出和为j的可能数，因为第i个骰子可能扔出1-6的点数，则dp[i][j]=dp[i-1][j-1]+dp[i-1][j-2]+dp[i-1][j-3]+dp[i-1][j-4]+dp[i-1][j-5]+dp[i-1][j-6]

```cpp
class Solution {
public:
	vector<double> numberOfDice(int n) {
		vector<double> res;
		vector<vector<int>> dp(n + 1, vector<int>(6 * n + 1, 0));
		for (int i = 1; i <= 6; i++)
			dp[1][i] = 1;
		for (int i = 2; i <= n; i++)
			for (int j = i; j <= 6 * n; j++)
				for (int k = 1; k <= 6 && k <= j; k++)
					dp[i][j] += dp[i - 1][j - k];
		for (int i = n; i <= 6 * n; i++)
			res.push_back(dp[n][i] / pow(6.0, n));
		return res;
	}
};
```

## 61. 扑克牌中的顺子

> 从扑克牌中随机抽5张牌，判断是不是一个顺子，即这5张牌是不是连续的。2～10为数字本身，A为1，J为11，Q为12，K为13，而大、小王可以看成任意数字。

首先把数组排序，然后统计数组中0的个数，最后统计排序之后的数组中相邻数字之间的空缺总数。如果空缺的总数小于或者等于0的个数，那么这个数组就是连续的。还需要注意一点：如果数组中的非0数字重复出现，则该数组不是连续的。

```cpp
class Solution {
public:
    bool IsContinuous( vector<int> numbers ) {
        if(numbers.size() < 5)
            return false;
        sort(numbers.begin(), numbers.end());
        int count = 0;
        for(int i = 0; i < numbers.size(); i++){
            if(numbers[i] == 0){
                count++;
                continue;
            }
            if(i + 1 < numbers.size() && numbers[i + 1] == numbers[i])
                return false;
            if(i + 1 < numbers.size())
                count -= numbers[i + 1] - numbers[i] - 1;
        }
        return count >=0 ? true : false;
    }
};
```

## 62. 圆圈中最后剩下的数字

> 0, 1, …, n-1这n个数字排成一个圆圈，从数字0开始每次从这个圆圈里删除第m个数字。求出这个圆圈里剩下的最后一个数字。

解法一：利用数学公式

首先定义最初的n个数字（0,1,…,n-1）中最后剩下的数字是关于n和m的方程为f(n,m)。

在这n个数字中，第一个被删除的数字是(m-1)%n，为简单起见记为k。那么删除k之后的剩下n-1个数字为0,1,…,k-1,k+1,…,n-1，并且下一个开始计数的数字是k+1。相当于在剩下的序列中，k+1排到最前面，从而形成序列k+1,…,n-1,0,…k-1。该序列最后剩下的数字也应该是关于n和m的函数。由于这个序列的规律和前面最初的序列不一样（最初的序列是从0开始的连续序列），因此该函数不同于前面函数，记为f’(n-1,m)。最初序列最后剩下的数字f(n,m)一定是剩下序列的最后剩下数字f’(n-1,m)，所以f(n,m)=f’(n-1,m)。

接下来我们把剩下的的这n-1个数字的序列k+1,…,n-1,0,…k-1作一个映射，映射的结果是形成一个从0到n-2的序列：
```
k+1    ->    0
k+2    ->    1
…
n-1    ->    n-k-2
0   ->    n-k-1
…
k-1   ->   n-2
```
把映射定义为p，则p(x)= (x-k-1)%n，即如果映射前的数字是x，则映射后的数字是(x-k-1)%n。对应的逆映射是p-1(x)=(x+k+1)%n。

由于映射之后的序列和最初的序列有同样的形式，都是从0开始的连续序列，因此仍然可以用函数f来表示，记为f(n-1,m)。根据我们的映射规则，映射之前的序列最后剩下的数字f’(n-1,m)= p-1 [f(n-1,m)]=[f(n-1,m)+k+1]%n。把k=(m-1)%n代入得到f(n,m)=f’(n-1,m)=[f(n-1,m)+m]%n。

经过上面复杂的分析，我们终于找到一个递归的公式。要得到n个数字的序列的最后剩下的数字，只需要得到n-1个数字的序列的最后剩下的数字，并可以依此类推。当n=1时，也就是序列中开始只有一个数字0，那么很显然最后剩下的数字就是0。因此有递推公式：

当n=1时，f(n, m) = 0 

当n>1时，f(n, m) = [f(n-1, m) +m] % n

```cpp
class Solution {
public:
    int LastRemaining_Solution(int n, int m)
    {
        if(n <= 0 || m <= 0)
            return -1;
        if(n == 1)
            return 0;
        int res = 0;
        for(int i = 2; i <= n; i++)
            res = (res + m) % i;
        return res;
    }
};
```

解法二：利用环形链表模拟

```cpp
class Solution {
public:
    int LastRemaining_Solution(int n, int m)
    {
        if(n <= 0 || m <= 0)
            return -1;
        list<int> nums;
        for(int i = 0; i < n; i++)
            nums.push_back(i);
        auto it = nums.begin();
        int k = m - 1;
        while(nums.size() > 1){
            while(k--){
                it++;
                if(it == nums.end())
                    it = nums.begin();
            }
            it = nums.erase(it);
            if(it == nums.end())
                it = nums.begin();
            k = m - 1;
        }
        return nums.front();
    }
};
```

## 63. 股票的最大利润

> 假设把某股票的价格按照时间先后顺序存储在数组中，请问买卖交易该股票可能获得的利润是多少？例如一只股票在某些时间节点的价格为{9, 11, 8, 5, 7, 12, 16, 14}。如果我们能在价格为5的时候买入并在价格为16时卖出，则能收获最大的利润11。

[LeetCode 121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

用buy记录当前最低的买入价格，用diff记录当前最大利润，遍历更新这两项。

```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if(prices.empty())
            return 0;
        int buy = prices[0], diff = 0;
        for(int i = 1; i < prices.size(); i++){
            buy = min(buy, prices[i]);
            diff = max(diff, prices[i] - buy);
        }
        return diff;
    }
};
```

## 64. 求1+2+...+n

> 求1+2+…+n，要求不能使用乘除法、for、while、if、else、switch、case等关键字及条件判断语句（A?B:C）。

最直接的想法就是用递归，sum(n) = n+sum(n-1)，但是要注意终止条件，由于求的是1+2+…+n的和，所以需要在n=0的时候跳出递归，但是题目要求不能使用if,while等分支判断，可以考虑利用&&短路运算来终止判断。

```cpp
class Solution {
public:
    int Sum_Solution(int n) {
        int sum = n;
        bool flag = sum && (sum += Sum_Solution(n - 1));
        return sum;
    }
};
```

## 65. 不用加减乘除做加法

> 写一个函数，求两个整数之和，要求在函数体内不得使用＋、－、×、÷ 四则运算符号。

1. 两个整数做异或^，得到各位相加不进位的运算结果；

2. 两个整数做与&，然后再左移一位，即得到进位的运算结果；

3. 将上面两个结果相加，即重复步骤1,2，直至进位的运算结果为0.

```cpp
class Solution {
public:
    int Add(int num1, int num2)
    {
        while(num2){
            int sum = num1 ^ num2;
            int carry = (num1 & num2) << 1;
            num1 = sum;
            num2 = carry;
        }
        return num1;
    }
};
```

## 66. 构建乘积数组

> 给定一个数组A[0, 1, …, n-1]，请构建一个数组B[0, 1, …, n-1]，其中B中的元素B[i] =A[0]×A[1]×… ×A[i-1]×A[i+1]×…×A[n-1]。不能使用除法。

[LeetCode 238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/)

利用两个变量left和right，

left[i] = A[0]* A[1] * … * A[i-1],  

right[i] = A[i+1] * A[i+2] * … * A[n-1]

B[i] = left[i] * right[i]

```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int>& nums) {
        vector<int> res(nums.size(), 1);
        int left = 1;
        for(int i = 0; i < nums.size(); i++){
            res[i] *= left;
            left *= nums[i];
        }
        int right = 1;
        for(int i = nums.size() - 1; i >= 0; i--){
            res[i] *= right;
            right *= nums[i];
        }
        return res;
    }
};
```

## 67. 把字符串转换为整数

> 请你写一个函数StrToInt，实现把字符串转换成整数这个功能。当然，不能使用atoi或者其他类似的库函数。

[LeetCode 8. String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi/)

考虑以下情况的处理：

1. 前导空格
2. '+'，'-'符号
3. 整数溢出
4. 无效字符

```cpp
class Solution {
public:
    int StrToInt(string str) {
        int i = 0, sign = 1, result = 0;
        if(str.empty())
            return 0;
        while(i < str.size() && str[i] == ' ')
            i++;
        if(i == str.size()) return 0;
        if(str[i] == '+' || str[i] == '-'){
            sign = str[i] == '+' ? 1 : -1;
            i++;
        }
        while(i < str.size()){
            if(str[i] < '0' || str[i] > '9')
                return 0;
            if(result > INT_MAX / 10 || (result == INT_MAX / 10 && str[i] - '0' > INT_MAX % 10))
                return sign == 1 ? INT_MAX : INT_MIN;
            result = result * 10 + (str[i] - '0');
            i++;
        }
        return sign * result;
    }
};
```

## 68. 树中两个节点的最低公共祖先

> 输入两个树结点，求它们的最低公共祖先。

[LeetCode 235. Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)

如果树为二叉查找树，则两个节点 p, q 的公共祖先 root 满足 root.val >= p.val && root.val <= q.val。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if(root == nullptr)
            return root;
        if(root -> val > p -> val && root -> val > q -> val)
            return lowestCommonAncestor(root -> left, p, q);
        if(root -> val < p -> val && root -> val < q -> val)
            return lowestCommonAncestor(root -> right, p, q);
        return root;
    }
};
```

[LeetCode 236. Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/)

如果树为普通的二叉树，在左右子树中查找是否存在 p 或者 q，如果 p 和 q 分别在两个子树中，那么就说明根节点就是最低公共祖先。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if(root == nullptr || root == p || root == q)
            return root;
        TreeNode* left = lowestCommonAncestor(root -> left, p, q);
        TreeNode* right = lowestCommonAncestor(root -> right, p, q);
        return left == nullptr ? right : right == nullptr ? left : root;
    }
};
```